---
layout: post
title: "Your Git Signing Key Shouldn't Live on Your Laptop"
date: 2026-06-12 19:00:00
categories: security enclaves git
description: "Part 1 of 2: I moved a git commit signing key into an AWS Nitro Enclave. This half is about the enclave itself — what it is, how you get TLS into a box with no network, and why a remote client can trust a self-signed certificate when it's backed by attestation."
image: /img/nitro-signer.png
---

<p align="center">
  <img src="/img/nitro-signer.png" alt="Nitro-Enclave-enabled git commit signer" width="320" />
</p>

Eleven years ago I [told everyone to use GPG][gpg] to sign things. I still think
signing is good. What I've soured on is *where the key lives*. Your commit
signing key is sitting in `~/.ssh` or a GPG keyring on the same laptop that runs
`npm install` against the entire internet. The signature proves the commit came
from your *machine*. It does not prove it came from *you*, and it definitely
doesn't prove the key hasn't been slurped up by something that got a toehold on
that machine.

So I built a [small thing][repo] to ask a more interesting question: what if the
signing key were generated somewhere you couldn't read it — not your filesystem,
not your shell, not even the host operating system — and you could still
`git commit -S` like nothing changed?

That somewhere is an [AWS Nitro Enclave][nitro]. This is a two-part tour. **This
post** is about the enclave itself: what it is, how TLS gets *into* one, and how
a remote client decides to trust it. [**Part 2**][part2] is the fun trick that
makes `git` cooperate without a single patch.

## The shape of it

The private key is an ed25519 keypair generated *inside* the enclave on boot. It
is never serialized, never logged, never sent anywhere. Your laptop talks to the
enclave over TLS, asks it to sign some bytes, and gets back a signature. `git` is
pointed at a tiny shim that stands in for `ssh-keygen` — but that's next post's
problem. Here's the whole board:

<div class="mermaid">
flowchart LR
  subgraph laptop["Your laptop"]
    git["git commit -S"]
    shim["git-enclave-signer<br/>(the shim)"]
  end
  subgraph host["EC2 Nitro host (parent)"]
    runner["run-enclave.sh<br/>socat bridge + gvproxy"]
    subgraph enc["Nitro Enclave"]
      nit["nitriding<br/>TLS termination"]
      app["signer app<br/>ed25519 key + NSM"]
    end
  end
  git -->|gpg.ssh.program| shim
  shim -->|"HTTPS :443"| runner
  runner -->|vsock| nit
  nit -->|"http 127.0.0.1:8081"| app
  app -.->|"attestation + signatures"| shim
</div>

The thing to internalize about Nitro Enclaves: an enclave is a stripped-down
virtual machine carved off from a parent EC2 instance, with **no persistent
storage, no interactive access, and no network device**. The only way in or out
is a `vsock` — a socket to the parent. That's it. Even the operator who owns the
AWS account can't `ssh` into it or read its memory. Which is wonderful for
holding a key, and mildly infuriating the first time you try to get a TLS
connection to one.

## Getting TLS into a box with no network

There's no NIC in there, so there's nothing to bind `:443` to in the normal
sense. The pattern that's emerged in this space is [nitriding][nitriding], a
daemon that runs *inside* the enclave, terminates TLS there, and reverse-proxies
the decrypted request to your app on `127.0.0.1`. Traffic reaches it because the
parent runs a dumb `socat` bridge: host `TCP:443` ⇄ `vsock` ⇄ nitriding.

The key decision is the certificate. You can have nitriding do the full ACME
dance and get a real Let's Encrypt cert — but that needs a public DNS name, a
raw-TCP load balancer on 443, and outbound internet from the enclave. For an
example I wanted to *run*, I let nitriding present a **self-signed** cert
instead. Which sounds like the part where I tell you to click through a browser
warning. It isn't — and that's the whole point of the next section.

## Trust comes from attestation, not the certificate

A self-signed cert is untrusted by the web PKI, by design. We don't get our
trust from the PKI. We get it from an **attestation document**.

The enclave has access to one magic device, the Nitro Security Module (NSM).
Ask it for an attestation and it hands back a [COSE][cose]-signed document
containing the enclave's measurements — most importantly **PCR0**, a hash of the
exact enclave image that's running — plus two fields you get to fill in: a
`nonce` and some `user_data`. That document chains, cryptographically, all the
way up to an [AWS-published root certificate][aws-root].

The signer app puts `sha256(public_key)` into `user_data`. So the document
doesn't just say "*some* enclave is running"; it says "the enclave running image
`PCR0=…` is holding the key whose hash is `…`." A client verifies the chain,
checks the nonce it just generated (so the document can't be a replay), and
checks that `user_data` matches the public key it was handed. Only then does it
trust that key.

<div class="mermaid">
sequenceDiagram
  actor Dev as You
  participant Shim as git-enclave-signer
  participant App as Enclave app
  participant NSM as Nitro NSM
  participant Root as AWS Nitro root
  Dev->>Shim: attest https://enclave-host
  Shim->>Shim: generate random nonce
  Shim->>App: GET /attestation?nonce=…
  App->>NSM: attest(user_data = sha256(pubkey), nonce)
  NSM-->>App: COSE doc {PCR0, user_data, nonce}
  App-->>Shim: attestation doc + pubkey
  Shim->>Root: verify certificate chain
  Shim->>Shim: nonce matches? user_data == sha256(pubkey)?
  Shim-->>Dev: ✓ trusted key, PCR0 = 1f2e…
</div>

So when I run `git-enclave-signer attest https://my-host`, what I get back is a
key I can trust *because I verified which code is holding it*, not because some
CA vouched for a hostname. The self-signed cert is just an encrypted pipe; the
attestation is the trust.

> If the enclave is faked — say someone runs the app as a plain process with no
> NSM — there's no valid COSE document and no chain to the AWS root. The client
> refuses it. There's a deliberately obvious dev-mode stub for running on a
> laptop, and the verifier rejects it on sight.

That's the hard part, and it's done. We have an encrypted channel to a box we
can prove is running known code, holding a key nobody can read. What's left is
almost cheeky by comparison: convincing `git` to use it without modifying `git`
at all. That's [Part 2][part2].

The whole thing — enclave app, the git shim, the EIF build, and a one-host CDK
stack to deploy it — is at [github.com/husobee/secure-git-signer][repo].

[repo]: https://github.com/husobee/secure-git-signer
[part2]: /security/enclaves/git/2026/06/13/git-signing-nitro-enclave-part-2.html
[gpg]: /security/gpg/2015/06/18/shhh-use-pgp.html
[nitro]: https://docs.aws.amazon.com/enclaves/latest/user/nitro-enclave.html
[nitriding]: https://github.com/brave/nitriding-daemon
[cose]: https://www.rfc-editor.org/rfc/rfc8152
[aws-root]: https://docs.aws.amazon.com/enclaves/latest/user/verify-root.html

<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true, theme: 'neutral' });
</script>
