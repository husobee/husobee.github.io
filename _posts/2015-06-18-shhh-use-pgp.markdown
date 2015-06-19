---
layout: post
title: "It's a Secret... Use GPG."
date: 2015-06-18 20:00:00
categories: security gpg
---

GNU Privacy Guard is an [RFC-4480][rfc4880] compliant OpenPGP implementation.  PGP was created 
in 1992 out of spite from what I gather by Phil Zimmerman.  In 1991 Senate Bill 266 required manufacturers 
of secure communications equipment to insert special "trap doors" in their products, so that the government
could read any encryped communications... (Sounds fairly similar to [current events today][stupid]).

What came from Phil Zimmerman's head has changed history.  PGP gave the world the access to an easy mechanism to 
keep messages confidental as well as authenticatible.  Using asymetric-key cryptography as well as symetric-key, PGP
can be used to encrypt and digitally sign data. 

# Keys Everywhere

As mentioned above, PGP uses asymetric-key cryptography.  The specification mentions several different types of key
algorithms you can choose from, including RSA, DSA, etc.  What all these have in common is they all consist of two
distinct keys that are linked together with our good friend math.  Below is a very simplified example of asymetric-key
cryptography.

{% highlight text %}

Given 
    R = U * V
We can calculate phi(R) as shown below:
    phi(R) = (U - 1) * (V - 1)
We want R to be hard to factor, so:
    P * Q = 1 (mod phi(R))

P = Public Key
Q = Private Key

To encrypt a message all we do is:
    (Plaintext) ^ P mod(R)

To decrypt a message all we do is:
    (Ciphertext) ^ Q mod(R)

{% endhighlight %}

If you are following the above, you will notice that this is some pretty simple math.  To encrypt we raise the plaintext
version of what we want to encrypt with one key, and we raise the ciphertext to the other key to decrypt.  This is very simple
to perform, but EXTREMELY hard to figure out without the large primes P and Q because it is hard to find the common divisors.

This form of cryptography allows us to widely publish our public key, so that anyone can send us encrypted messages, and if we 
keep our private key completely secret, you will be the only one who can read the message.  PGP uses asymetric-key encryption, but 
not for encrypting the message itself.  PGP uses symetric key encryption for actually encrypting the message you are trying to hide,
and is capable of using many different mechanisms to accomplish this, including blowfish, twofish, aes, etc.  Symetric key encryption is 
where you share a key, such as a passphrase, and you encrypt and decrypt with the same key.

PGP is clever in that the algorithm first creates a really random session "password" which is used to encrypt the payload of the 
message.  Then uses the recipient's public key to asymetrically encrypt said random session password.  The actual message encryption 
is very much stronger than the encryption of the random session password, which is a one time use password.

To start using GPG to encrypt and sign your documents install gpg.  Then generate a keypair as shown below:

{% highlight sh %}

[husobee@localhost ~]$ gpg --gen-key
gpg (GnuPG) 2.1.3; Copyright (C) 2015 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Note: Use "gpg2 --full-gen-key" for a full featured key generation dialog.

GnuPG needs to construct a user ID to identify your key.

Real name: somone real
Email address: something@e-ie.io
You selected this USER-ID:
    "somone real <something@e-ie.io>"

    Change (N)ame, (E)mail, or (O)kay/(Q)uit? o
    We need to generate a lot of random bytes. It is a good idea to perform
    some other action (type on the keyboard, move the mouse, utilize the
    disks) during the prime generation; this gives the random number
    generator a better chance to gain enough entropy.

{% endhighlight %}

Then sit and wait, for a while.  In order to generate such a huge random prime, your computer has to create a ton of entropy.  When this is done, 
you should have a keypair, and be ready to start encrypting messages or documents.  Below is an example of encrypting a file with gpg.

{% highlight sh %}
[husobee@buzzard ~]$ gpg -a -e 2015-06-18-shhh-use-pgp.markdown 
You did not specify a user ID. (you may use "-r")

Current recipients:

Enter the user ID.  End with an empty line: husobee@e-ie.io
gpg: 53A7719F: There is no assurance this key belongs to the named user
pub  rsa2048/53A7719F 2015-06-03 <husobee@e-ie.io>
 Primary key fingerprint: 1FB4 128F 270E B14D 7556  4081 63E0 6DBC 53A7 719F

Enter the user ID.  End with an empty line: 
 [husobee@buzzard ~]$ ls \*.asc
2015-06-18-shhh-use-pgp.markdown.asc

{% endhighlight %}

Looking at that file that was created:

{% highlight text %}
-----BEGIN PGP MESSAGE-----
Version: GnuPG v2

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
=A7vC
-----END PGP MESSAGE-----

{% endhighlight %}

After you are done encrypting files for yourself, you will likely want to start sharing with friends.  PGP is based on a 
web of trust.  So you trust someone, you sign their public key, and they become more trustworthy.  You can also get keys from a
keyserver, and import individual's public keys as seen below:

{% highlight sh %}
[husobee@localhost ~]$ gpg --search-keys husobee@e-ie.io
gpg: data source: http://107.130.139.161:11371
(1)     (husobee) <husobee@e-ie.io>
          4096 bit RSA key DBB16047, created: 2015-06-05, expires: 2016-06-04
          Keys 1-1 of 1 for "husobee@e-ie.io".  Enter number(s), N)ext, or Q)uit > q
          gpg: error searching keyserver: Operation cancelled
          gpg: keyserver search failed: Operation cancelled

[husobee@localhost ~]$ gpg --recv-keys DBB16047
gpg: key DBB16047: public key "(husobee) <husobee@e-ie.io>" imported
gpg: Total number processed: 1
gpg:               imported: 1

{% endhighlight %}

Always make sure to verify the fingerprint of keys from people you know.  The fingerprint is a SHA hash of the public
key and is used to make sure you have their proper key.

[rfc4880]: https://tools.ietf.org/html/rfc4880
[stupid]: https://www.techdirt.com/articles/20150502/14414630864/encryption-what-fbi-wants-it-can-only-have-destroying-computing-censoring-internet.shtml
[gpg]: https://gnupg.org/
