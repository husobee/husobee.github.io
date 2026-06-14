---
layout: post
title: "Teaching git to Sign Inside an Enclave"
date: 2026-06-13 19:00:00
categories: security enclaves git
description: "Part 2 of 2: with a trustworthy enclave holding the key, the rest is convincing git to use it — no patches required. The SSHSIG seam, the byte format that has to be exactly right, the one test that proves it, and how to deploy the whole thing on a single EC2 host."
image: /img/nitro-signer.png
---

<p align="center">
  <img src="/img/nitro-signer.png" alt="Nitro-Enclave-enabled git commit signer" width="320" />
</p>

[Last post][part1] I built up an AWS Nitro Enclave that generates an ed25519
signing key it won't let anyone read, serves it over TLS, and proves — by
attestation — exactly which code is holding that key. We ended with an encrypted
channel to a box we can trust, and a public key we've verified.

Now the cheeky part: getting `git` to sign with that key, without patching `git`,
without a custom credential helper, without anyone learning a new command.

## Making git cooperate, with zero patches

`git` has supported SSH signing for a while now (`gpg.format = ssh`). When it
signs, it shells out to a program — `ssh-keygen` by default — like this:

{% highlight text %}
ssh-keygen -Y sign -n git -f <key> <bufferfile>
{% endhighlight %}

…and reads the signature back from `<bufferfile>.sig`. That's a *seam*. Nothing
says the program on the other end has to be `ssh-keygen`. So I wrote a shim that
git calls instead. When git asks it to sign, it forwards the bytes to the
enclave and writes back the answer. When git asks it to do anything else
(verifying, finding principals), it just `exec`s the real `ssh-keygen`, which
verifies the enclave's signatures perfectly well — because they're ordinary
[SSHSIG][sshsig] signatures.

<div class="mermaid">
sequenceDiagram
  participant Git as git commit -S
  participant Shim as git-enclave-signer
  participant Nit as nitriding (TLS)
  participant App as Enclave app
  Git->>Shim: -Y sign -n git -f key buffer
  Shim->>Nit: POST /sign {namespace, data}
  Nit->>App: forward over localhost
  App->>App: SSHSIG over sha512(data), ed25519
  App-->>Nit: armored "BEGIN SSH SIGNATURE"
  Nit-->>Shim: signature
  Shim->>Git: write buffer.sig
  Git->>Git: embed signature in the commit
</div>

The enclave produces the *entire* armored SSHSIG — magic preamble, the public
key, the namespace, the signature blob — so the shim stays dumb. It reads the
buffer git handed it, POSTs it, and drops the result at `<file>.sig`. That's the
whole shim.

## The byte format that has to be exactly right

The reason this works at all is that I'm not inventing a signature format — I'm
reproducing OpenSSH's, to the byte. SSHSIG doesn't sign your message directly;
it signs a `sha512` of it, wrapped in a small framed blob with a magic preamble
and a namespace:

{% highlight go %}
// per OpenSSH's PROTOCOL.sshsig — sign H(message), not the message
h := sha512.Sum512(message)

var signed bytes.Buffer
signed.WriteString("SSHSIG")
writeSSHString(&signed, []byte(namespace))
writeSSHString(&signed, nil) // reserved
writeSSHString(&signed, []byte("sha512"))
writeSSHString(&signed, h[:])

sig, err := signer.Sign(rand.Reader, signed.Bytes())
{% endhighlight %}

Get a length prefix wrong and nothing complains loudly — you just produce a blob
that silently fails to verify later. So the most important test in the whole
project isn't a mock or a unit assertion. It signs something inside the enclave,
then shells out to the **real** `ssh-keygen -Y verify` and asserts it's happy:

{% highlight text %}
--- PASS: TestSSHSignatureVerifiesWithSSHKeygen
{% endhighlight %}

If stock OpenSSH accepts the signature, so will `git`. That one test is the
contract.

## Wiring it up

Point git at the shim once and you never think about it again:

{% highlight bash %}
export ENCLAVE_URL=https://my-enclave-host
git config gpg.format ssh
git config gpg.ssh.program "$(pwd)/git-enclave-signer"
git config user.signingkey ~/.config/git/enclave-signer.pub
git config commit.gpgsign true
{% endhighlight %}

The `enclave-signer.pub` there is the key you got back from the `attest` step in
[Part 1][part1] — the one you *verified*. From here, `git commit` is signed by a
key that has never touched your disk:

{% highlight text %}
$ git commit -m "signed in an enclave"
$ git log --show-signature -1
Good "git" signature with ED25519 key SHA256:…
{% endhighlight %}

## Deploying the whole thing

The repo ships a single-file [CDK][cdk] stack that stands up one Nitro-enabled
EC2 host, configures the enclave allocator, and runs a tiny systemd unit that
keeps the enclave alive. The loop end to end:

{% highlight bash %}
make eif                 # build the enclave image, capture its PCR0
cd iac && npx cdk deploy # one Nitro host + an S3 bucket for the image
aws s3 cp build/app.eif s3://<bucket>/app.eif   # hand it the image
git-enclave-signer attest https://<host>        # verify + grab the key
{% endhighlight %}

Notably, there's no key material to provision and no secret to inject. The
enclave makes its own key on boot. The host never sees it. You verify it from
your laptop and you're done.

## What I left out on purpose

This is an *example*, and examples earn their keep by being small. Two honest
simplifications:

- **The key is ephemeral.** It's regenerated on every enclave boot, so a restart
  is a new identity. For a key you actually want to keep, you'd wrap it with KMS
  and gate the decrypt on PCR0 — so the key only comes back to life inside an
  enclave running *this exact image*, and nowhere else. That's the genuinely
  powerful pattern, and it deserves its own post.
- **The cert is self-signed.** Covered in [Part 1][part1] — trust is from
  attestation. Bolt on ACME when you want the green padlock too.

What I like about the result is how little ceremony it takes to get a real
security property: a signing key with a hardware-rooted, *remotely verifiable*
boundary, behind a `git commit -S` that looks exactly like it always did. A
decade ago I was [poking at TLS handshakes][golang-tls] to understand what trust
even means on the wire. This is the same itch, scratched one layer deeper — the
trust isn't in a name on a certificate, it's in a measurement of the code.

The whole thing is at [github.com/husobee/secure-git-signer][repo]. Clone it,
read it, break it.

Hope this was helpful to someone.

[repo]: https://github.com/husobee/secure-git-signer
[part1]: /security/enclaves/git/2026/06/12/git-signing-nitro-enclave-part-1.html
[golang-tls]: /golang/tls/2016/01/27/golang-tls.html
[sshsig]: https://github.com/openssh/openssh-portable/blob/master/PROTOCOL.sshsig
[cdk]: https://docs.aws.amazon.com/cdk/v2/guide/home.html

<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@11/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ startOnLoad: true, theme: 'neutral' });
</script>
