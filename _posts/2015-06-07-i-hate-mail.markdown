---
layout: post
title: "I Hate Email"
date: 2015-06-07 15:30:00
categories: email mail-server rant dmarc dkim spf
---
So, I realized I hate email.

Then again, I really _hate the idea of my personal email communications being used to "figure me out."_  
I had this wild hair over the past two months to 
start hosting my own email again, after my email maintenance skill fatigued over
the past 8 years of using a very popular email service.

From what I remembered, I did everything a good mail server should do... Here
was my checklist:

1. Setup MX record pointed at my mail-server
2. Setup my mail-server host's reverse IP to the A record forward for the host
3. Be free of companies having, and harvesting my email ;)

Back In The Day this was all that I remember needed.  Great.  Send some emails 
to inform everyone that I was free from the evil big email companies.

WELL, a few days later, (with no responses from ANYONE) I had this feeling that 
something was wrong...

ALL MY MAIL WAS GOING TO PEOPLE'S SPAM FOLDERS!  GAAAAHH!!!

Researching Online I found that I really shouldn't be hosting my own mail, as it
isn't the same friendly kind internet it used to be when I hosted my own mail...
Apparently the following items should have been on my checklist:

1. Setup MX record pointed at my mail-server -- CHECK!
2. Setup reverse dns lookup to my forward -- CHECK!
3. Setup an SPF TXT dns record -- WTF?
4. Setup a DKIM TXT dns record -- HUH??
5. Setup a DMARC TXT dns record -- Is this a joke??

The Internets told me that a SPF (Sender Policy Framework) entry should look as 
follows in DNS based on [rfc 4408][rfc4408]:

{% highlight text %}
example.com. IN TXT "v=spf1 +mx a:colo.example.com/28 -all" 
{% endhighlight %}


From What I can gather, this looks like a good idea as to allow for programmatic
email to be allowable, and not marked as spam coming from something other than 
the MX record of the domain... but I am still not understanding why reverse DNS
wouldn't provide exactly the same protection as an SPF record for a single self
hosted mail-server...

Great, I am now SPF-ed, should be good to go?  YAY, check it out!  Gmail accepts
my SPF record!

{% highlight text %}
Received-SPF: pass (google.com: best guess record for domain of
        hxxxxxx@xxxxx designates xx.xx.xx.xx as permitted sender)
        client-ip=xx.xx.xx.xx;
Authentication-Results: mx.google.com; spf=pass (google.com: best guess record
        for domain of hxxxxxxx@xxxxx designates xx.xx.xx.xx as permitted sender)
        smtp.mail=hxxxxxx@xxxxx
{% endhighlight %}

Time to email everyone!  Oh, crap, they are still getting my emails in their
spam folder... Well, shoot, what else do I need to do?  I decided to look at 
the headers of an email from one big email company to another to see what I am 
missing:

{% highlight text %}
Received-SPF: pass (google.com: domain of fxxxx@yahoo.com designates
        98.139.213.55 as permitted sender) client-ip=98.139.213.55;
Authentication-Results: mx.google.com; spf=pass (google.com: domain of
        fxxxxx@yahoo.com designates 98.139.213.55 as permitted sender)
        smtp.mail=fxxxxx@yahoo.com; dkim=pass header.i=@yahoo.com; dmarc=pass
        (p=REJECT dis=NONE) header.from=yahoo.com
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/relaxed; d=yahoo.com; s=s2048;
        t=1433352789; bh=vaJZcMSXdi5HK/I/aymlJ/FMhQYF6QqYQog3rO9zTak=;
        h=Subject:References:From:In-Reply-To:Date:To:From:Subject;
        b=(cut for brevity)
{% endhighlight %}

Hmm... Well, I see the SPF pass, just like mine so that is good.  But WTF is 
dkim=pass... and WTF is dmarc=pass (p=REJECT dis=NONE)... And what is a 
DKIM-Signature?!?!  Why is email so foreign to me all of a sudden... it used to 
be so clear...

Digging into DKIM on the Internets, I see DKIM (DomainKeys Identified Mail) is a
cryptographic signing scheme made for mail-servers to sign emails going to remote
mail servers.  This actually makes a ton of sense to me.  I personally like 
signing my emails with GPG, and totally understand this check.  Learning about
DKIM on the internet, basically you create an RSA keypair, and publish the 
public key as a DNS TXT entry on your DNS server!  Wow, that kinda makes sense. 
This way a remote mail-server can do a single DNS lookup and verify that the 
email coming in is really and truly from the owner of the domain.  Moreover in
conjunction with the SPF record, one can tell if that mail-server is an authority
to send emails for that domain based on [dkim specifications][dkim]

A few package installs and configurations, and I am now able to pass emails from
my mail-server through a DKIM milter, and amazingly I get this header now when I
send emails to gmail:

{% highlight text %}
Received-SPF: pass (google.com: best guess record for domain of
        hxxxxxx@xxxxxx designates xx.xx.xx.xx as permitted sender)
        client-ip=xx.xx.xx.xx;
Authentication-Results: mx.google.com; spf=pass (google.com: best guess record
        for domain of hxxxxxx@xxxxxx designates xx.xx.xx.xx as permitted sender)
        smtp.mail=hxxxxxx@xxxxxx; dkim=pass (test mode) header.i=@xxxxxx
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/relaxed; d=xxxxxx; s=mail;
        t=1433702390; bh=mAxubT+K5IgwV+hEgjKzw0Zdjb22UYjgGIV1GU6Sk/4=;
        h=Date:From:To:Subject:From;
        b=(cut for brevity)
{% endhighlight %}

Whoa, I am starting to look like a real email service... Nice!  I still notice
one more "pass" in the previous email, DMARC.

DMARC (Domain-Based Message Authentication, Reporting and Conformance) seems to 
me to be a mechanism where email services communicated with (real or spammed 
messages sent) can send authentication failed reports.  It seems to give the 
email Host the ability to specify how spam reports are sent back to the domain
owner.  Seems like I need to make a mailbox to handle all of the conformance 
reporting, as well as add a DNS entry that looks kind-of like this to have a 
valid dmarc:

{% highlight text %}
example.com IN TXT "v=DMARC1;p=none;pct=100;rua=mailto:emailaddress@example.com;ruf=mailto:emailaddress@example.com;"
{% endhighlight %}

Now, lets take a look at the headers from gmail!!! So Exciting!

{% highlight text %}
Authentication-Results: mx.google.com; spf=pass (google.com: best guess record
        for domain of hxxxxxx@xxxxxx designates xx.xx.xx.xx as permitted sender)
        smtp.mail=hxxxxxx@xxxxxx; dkim=pass (test mode) header.i=@xxxxxx;
        dmarc=pass (p=NONE dis=NONE) header.from=xxxxxx

{% endhighlight %}

YAY, all the checks I know about are being done successfully!  *I wonder if people will get my email now*  ;)

Hope this was helpful to anyone.

[rfc4408]: http://www.ietf.org/rfc/rfc4408.txt
[dkim]: http://dkim.org/ietf-dkim.htm#published
