---
layout: post
title: "TCP Strangeness with Elasticache"
date: 2015-11-02 20:00:00
categories: tcp keepalive timeout
---

# Connections Strangely Closing

A few days ago my team at work were running some automated tests against a staging
environment, and we had the strangest thing happen.  A single request to a service
we control gave back a 504 status code after 60 seconds, and then worked right away
on the next try.

## High level architecture

We trying to follow a [microservice oriented approach][microservice] to our web 
services. (Trying..) There are several single purpose web services that do 
particular tasks and all of these services work together to create our whole service
which is excellent delegation of responsibilities, and will allow us to scale.

The service in question we were seeing one 504 response from sits behind an Load
Balancer.  Below is a diagram:

{% highlight text %}
 ------------                                    ----        --------------
| User Agent | ---> Request To Invite User ---> | LB |  --> | Service Node |
 ------------   <---  504 Response     <---      ----        -------------- 

{% endhighlight %}

As shown, this is the information we had to start with.  Well obviously, the 504 
status tells us that there was a Gateway Timeout, most likely at the Load Balancer.

Logging into the service node, and seeing that other traffic was able to hit the 
service as well as the endpoint that originally wasnt responding properly caused
us all to scratch our heads for a minute.

We then proceeded to look in the application logs for the request in question, and 
found that the web api application logged that ONE and only ONE api call to said
endpoint took 15 minutes and 31 seconds to complete...

Well, obviously this was the request in question.  The request took over 60 seconds
to write back to the Load Balancer, therefore causing the Load Balancer to 
communicate back to the User Agent the request timed out.  Looking in the code, it
was evident that this service made use of another web service.

The service in question provides our system stored single use unique identifiers
which are very short lived.  These identifiers are persisted in [Redis][redis] which
is hosted in [Amazon Elasticache][elasticache].

Our diagram now looks like this:

{% highlight text %}
 ------------                    ----       --------          ------          -----
| User Agent | ---> Request---> | LB | --> | Invite | -New-> | UUID | -Set-> |Redis|
 ------------  <---Response<---  ----  <--  --------  <-UUID- ------          -----
{% endhighlight %}

After looking in this unique identifiers service, it was found from the logs that 
the request to make a new UUID from our invitation service took the full 15 minutes
and 31 seconds to complete.  This focused the blame squarely at the SET operation in
Redis.  But why couldn't we immediately replicate it?  Why wasn't it acting this 
way all the time?

I immediately started a packet capture with tcpdump on another development 
environment that is setup exactly as the other environment was, using elasticache 
and amazingly, the first time I tried the same operation I was able to replicate 
the behavior exactly. While watching redis port 6379 communications I saw a pretty
clear tcp psh followed by a bunch of acks being sent in sequence using a backoff 
trying to re-establish communications with the redis server over the tcp socket.  
This was interesting because the retry duration of 15 minutes lined up perfectly
with linux tcp_retries_2.  It was obviously trying to retransmit over a dead tcp
socket.

So, Elasticache is a little magic, in that I don't believe Amazon will give you
access to the host, and or the network topology.  This was a bummer.  I did do an
INFO command on the redis instance and saw that timeout was set to 0.

According to the documentation if timeout is set to 0, redis is to NEVER close a tcp
connection with a client.  I also saw tcp-keepalive set to 0.  Redis documentation 
says if tcp-keepalive is set to 0 redis is not to send any nil acks to clients to 
maintain a socket.

With these settings in place, I believe what is happening is a long polling,
non-utilized (staging doesn't get much traffic) connection has such a long period
of no communications between the client and server that the magic networking/proxy 
that Amazon is doing looses sight of that tcp connection, and severs the connection,
as the proxies/firewalls/network equipment only have a certain amount of connection 
tracking history.  Since the network equipment are the ones in the middle cutting 
the connections, both the client and server still think the tcp socket is alive 
and well, and then timeout when trying to communicate to each other over said 
socket.

[This description is helpful][keepalive] in explaining what is going on here.  If
you use tcp-keepalive on a particular number of seconds in redis, redis will perform 
a keepalive to talk to the clients to make sure all clients are alive and well, thus
by a side-effect also informing various network gear inbetween that this connection
is still alive and shouldn't be evicted from the connection table in the network gear.

Another way of handling this error is by setting Redis' timeout to a number of seconds, 
which will close connections after a certain amount of idle connection time.  This method
seems like it would have more overhead when reconnection happens for idled clients, but
it will not cause the very strange issues we originally saw.


[microservice]: http://martinfowler.com/articles/microservices.html
[redis]: http://redis.io
[elasticache]: http://docs.aws.amazon.com/AmazonElastiCache/latest/UserGuide/WhatIs.html
[keepalive]: http://www.tldp.org/HOWTO/TCP-Keepalive-HOWTO/usingkeepalive.html