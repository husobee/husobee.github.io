---
layout: post
title: "Method Not Found"
date: 2015-09-29 20:00:00
categories: golang vestigo 405
---

## Method Not Found... grrr.

I wanted to rant just a little bit about how [few of the URL routers][url-routers] I have used in golang 
(except of course [Vestigo][vestigo-main] :) ) handle HTTP Method Not Allowed correctly. In 
a vast majority of the URL routers I have seen (gorilla mux/pat, echo, etc) the most common 
way of route matching is by glomming the method into a "matcher" and handling it the exact same 
way a route not found is handled.  Returning a 404 or Resource Not Found, is NOT the same as 
a 405, or Method Not Allowed.  Lets take an example:

{% highlight go %}
    router.Get("/welcome", GetWelcomeHandler)
    router.Post("/welcome", PostWelcomeHandler)
{% endhighlight %}

Here we are specifying one resource with two methods, GET and POST, that are served by different
handlers, GetWelcomeHandler, and PostWelcomeHandler respectively.  What is the proper behavior when 
a user-agent makes the following request to the server:

{% highlight text %}
    PUT /welcome HTTP/1.1
{% endhighlight %}

Most URL routers fail miserably with this type of request setup as shown above.  I have personally witnessed
Gorilla Mux/Pat, and others respond with the following:

{% highlight text %}
    HTTP/1.1 404 Not Found
{% endhighlight %}

## What is the problem

I think the problem stems from the complacency of "an error is an error, what is the big deal?" Well, this 
particular error is trying to communicate something to the caller.  The caller is now under the impression 
that this _resource_ doesn't exist, which isn't true.

A resource is defined by the URL Path.  In days of yore, a resource was a physical file, hosted by a web
server.  This is still true today, but when dealing with api development we sometimes loose the context. Clearly,
in our example *"/welcome"* does exist as a resource, that resource being the output of a function.

# What is the solution

Even though it is hard, we need to give proper error messages back to callers.  In our example above, we should be
giving the caller back this as a response:

{% highlight text %}
    HTTP/1.1 405 Method Not Allowed
    Allow: GET, POST
{% endhighlight %}

Clearly, the response indicates firstly nothing about the resource being Not Found which is not true.  Furthermore 
it is clear to the caller what methods *ARE* allowed for this resource.  This is of huge significance in working toward
the self describing API concept.  Not only that, but it tells the developer of a client integration that the method
they are using is not allowed, as opposed to the resource not being found, and wasting cycles on rabbit holes.

In conclusion, come on people.  We need to respond appropriately to failure situations.  [Vestigo][vestigo-main] takes
this into account, and will, behind the scenes, take care of people with regards to correctly responding to requests for 
resources using particular methods.  Give it a try!


[url-routers]: http://husobee.github.io/golang/url-router/2015/06/15/why-do-all-golang-url-routers-suck.html
[vestigo-main]: https://github.com/husobee/vestigo
