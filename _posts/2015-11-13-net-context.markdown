---
layout: post
title: "net.Context in Golang"
date: 2015-11-13 15:30:00
categories: golang net context
---

## We need some Context

Within web application development a nagging issue always crops up, how do I get 
data from one function to another in a pipeline?  For an example, I have a simple 
flow which consists of a chain of middlewares, one of which adds some trivial 
information, say a unique identifier for each request, which we need to use in 
the next function in our pipeline.  How can we concisely get that information passed
in our processing pipeline?  

Some solutions involve a global hashmap of request pointer to ```interface{}``` but 
this is a frustrating central clamping point.  The amount of time waiting for a lock
to write to, and read from these types of solutions are very unappealing.  

Other solutions involve creating a custom handler function, or middleware type 
wherein you can pass in a context created per request, where you can stow this 
enriched information from middleware to middleware to handler in a pipeline.  This 
can be a fine solution, except for the wildly different solutions everyone invents
for their context structures or interfaces.

## Enter net.Context

Luckily, Google came to the rescue, and provided a very interesting solution for 
dealing with the context problem, explained in depth [here][context-pattern]. 
According to the blog entry, Google requires every function signature to contain a 
net.Context parameter as the first argument for every function on the call path 
between incoming and outgoing requests.  This allows every request from creation
to response to have context as to what this request is for, and extra collected meta
data for every step of the pipeline.

In short, you can think of net.Context as a stackable context.  Take the following:

{% highlight golang %}
	// we are "stacking" our new context with value below on top 
	// of the "background" context which is just an empty context
	baseCTX := context.WithValue(context.Background(), "requestID", 12345)
	
	// we are now "stacking" a cancel-able context on top of our 
	// previous value context, net.Context allows us to be able 
	// to cancel a context by running the cancel function when we 
	// want to cancel this context.  The context interface defines 
	// a Done() <-chan struct{} struct method for us to check to see 
	// if a context has been timed-out or canceled, as well as 
	// an Err() error method to tell use why.  VERY HANDY!
	cancelCTX, cancel := context.WithCancel(baseCTX, "requestID", 12345)

	// we are now "stacking" another valued context on top of our cancel-able
	// context.
	lastCTX := context.WithValue(cancelCTX, "shortLived", true)

	// now comes the magic: calling the cancel function will traverse down the child
	// context hierarchy and iteratively cancel all of the child contexts.  This 
	// is pretty great because you can have a clean organization of where to prune
	// the context
	cancel()

{% endhighlight %}

## Coming Soon!

I am also very excited to hear that this net.Context interface will be coming soon
to the http.Request structure.  Hindsight is 20x20 of course, but this will solve 
so many implementation issues that are stop gapped by current web framework 
implementations.  Basically it will take out all need for using a framework at all
for passing context.

I believe personally that the days of context laden web frameworks is nearing the 
end.  I am also very glad that the golang community has shown the core contributors 
that there is a need for a clean context implementation attached to the http.Request
structure.

I can very clearly see piecemeal framework groupings being a thing, where people 
can pick the best router, and marry that url router with the best middleware 
implementation, and all the while not going through the effort of doing web api
development in a particular boxed framework.

[context-pattern]: http://blog.golang.org/context