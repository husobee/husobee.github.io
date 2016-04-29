---
layout: post
title: "More Email Fun"
date: 2016-04-29 08:00:00
categories: postfix tls
---


As [mentioned before in another posting][email-hate] i really dislike email.  I 
feel like the whole system is broken, and yet we have all bought into it.  
Allowing service providers to "read" all your messages in order to "figure you
out" or profile you based on your written word or even images that sit in your 
inbox for years makes me upset.  

That said, in order to be a lead actor in the world's digital security theater, 
google recently started penalizing independent mail providers who do not 
completely buy into using TLS to secure email communications from their mail
servers to google's mail servers.  Below is an image showing pretty clearly there
is something "wrong" with this email:

{% include image.html url="/img/badgmail.png" description="Oh No!! RED!!!" %}

You can see here there is a little pad lock that is red.  In the world of UI, 
red is danger.  WATCH OUT GMAIL USER!  It basically screams at the user saying
oh, dude, this message is not good!

Well great, now I need to explain to all my gmail users I email on occasion why
they are getting messages that are "flagged" as a red padlock.  I explain to them
that gmail is putting this flag on the message because my mail server is not using
TLS encryption to their mail server.  They then proceed to say, why are you not
encrypting messages, I thought you were a security minded person.  Then I have a
rage induced aneurysm.

## Ranting aside...

Here is how you can "fix" emails going to gmail users to not have the red padlock
if you are running postfix.  In your /etc/postfix/main.cf config file add the 
following lines:

{% highlight text %}
# for logging tls
smtpd_tls_loglevel = 1
# don't bother with stupid broken protocols
smtp_tls_protocols = !SSLv2, !SSLv3
# use the best ciphers you can
smtp_tls_ciphers = high
# this is your cert file for your mail server, you can get one from letsencrypt...
smtp_tls_cert_file=/etc/ssl/certs/cow_e-ie_io.crt
# this is the private key file for the cert
smtp_tls_key_file=/etc/ssl/private/cow.e-ie.io.key
# why yes, we want to use tls
smtp_use_tls=yes
{% endhighlight %}

Now when we email a gmail user we get something that looks like this:

{% include image.html url="/img/goodgmail.png" description="Yay, back to normal!" %}

Solved. Great.

## Or is it?

No, it isn't.  This is all just security theater in my opinion.  Your messages 
still live on gmail servers in plain text form (or at least they can decrypt your
messages if they are encrypting them) and gmail can do what they want with your
data.  They can feed it into various algorithms that tell you what you should
buy, or they can hand the messages over to the local authorities on request,
currently without any form of warrant.

I claim security theater simply because there should ALWAYS be a red padlock on
any message that isn't end-to-end encrypted.  I am probably one out of 10,000 people
who [takes any effort][use-gpg] to secure emails. I use GPG for several of my 
contacts, and have my fingerprint on my personal calling cards when I meet people.

I accept that email is broken, but I do everything I can to have secrecy when I 
can end-to-end, not just on a transport level.  Moreover, the entire certificate
trust system is easily broken in my opinion.  Have you looked through all the 
trusted certificates signing authorities on your distribution?  Do YOU trust them?

Emails are like postcards.  What this red padlock is saying is, basically do you
have a lock on your mailbox so someone off the street cant look inside the mailbox
and see what your postcard says.  Gmail, the postman can still read the postcard
and do what they will with said postcard, not limited to handing the postcard 
to someone you don't it handed to.  Honestly, I would rather have the email 
wrapped in an envelope so even the postman can't read it.

I feel like gmail is just playing a publicity stunt to try to safe face for 
exploiting their users for years.

[email-hate]: http://husobee.github.io/email/mail-server/rant/dmarc/dkim/spf/2015/06/07/i-hate-mail.html
[use-gpg]: http://husobee.github.io/security/gpg/2015/06/18/shhh-use-pgp.html
