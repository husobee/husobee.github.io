---
layout: post
title: "Reviewing Siesta"
date: 2015-11-10 08:00:00
categories: golang framework review siesta
---

I have blogged in the past about various web framework concepts in golang, and
have also named some names articulating what dissuades me from using said frameworks
but I have not yet reviewed an entire framework before.  In this blog I will 
attempt to explain my sentiment about [Vivid Cortex's Siesta][siesta] and explain
my perceived pain points.

## Context, oh my...

Firstly, there is nothing spectacularly new and exciting about this web framework
in my opinion, much like all new golang http web frameworks.  Moreover post go1.7
many of these frameworks will die off as that is the slated release of built in 
request net.Context.  This should get everyone extremely excited, as *finally* we 
will have a persistent cancel-able context built into the http.Request structure.  
Basically removing 70% of what the droves of http frameworks offer to the public, 
a scheme for context passing from handler to handler.

Getting back to Siesta, it is clear they are following the "make your own" context 
approach as opposed to using the vastly superior stacking net.Context which will
soon be in the standard library.  Furthermore in looking at their 
[context source code][threadsafe-context] clearly the built in context isn't safe 
for multiple goroutines, if you ever have the need to access the context variables
in parallel.

## Handlers

As I have griped about in [a previous blog entry][router-hate] siesta invented a
new function signature for their handlers, which requires an implementation of 
their context from above to be passed in.  The do however have a wrapping concept
though in practice if you are using this framework, likely you are using all the 
features including the context in your handlers, so it seems silly to me (maybe 
the middlewares only need the context??) to not just implement the siesta standard
handler func signature complete with the context. 

I just have to say again... this style of framework, where you need to alter the
function signature is merely a stop-gap until go1.7 comes out.  When we get 
net.Context living within the request, all of this BS goes away.  We can all go 
back to standard http.HandlerFunc signatures and all will be right in the world 
again.  It almost feels like this last generation of web frameworks will be extinct
in the very near future.

Another thing I have noticed, and maybe it is in progress, is that there is no 
built in concept of content negotiation.  I feel that in order to call yourself a
REST framework, as opposed to a Web Framework, you need to have that concept built
into the ServeHTTP.  If a REST client requests Accept: application/xml the REST 
framework needs to respond appropriately by performing the negotiation with the 
client.  

## Middleware Composition

As for the middleware composition, with siesta.Compose, I can't seem to figure out
how you are supposed to have wrapping middlewares.  For example, lets look at the
awesome negroni middleware handler definition: 

{% highlight go %}
func MyMiddleware(rw http.ResponseWriter, r *http.Request, next http.HandlerFunc) {
  // do some stuff before
  next(rw, r)
  // do some stuff after
}
{% endhighlight %}

As can be seen, we have the ability to fully wrap a handler, or a chain of handlers
with negroni, so you can perform some action, call the next handler in the chain, 
then do some more stuff after the chain completes.  I like how they implemented
the quit signaling in the middleware chaining in siesta, but I really do not like
how there is no way to clearly have precondition and postcondition operations
within middlewares if I wanted.

## Something Borrowed, Something Blue

Now to my favorite section of any web framework, the url routing.  I have to say, 
at least they picked a trie based router, by taking from HTTPRouter, and excellent
router.  That being said, they are running into [some more of my gripes][not-allow]
by having trees rooted in the request http method.  Try hitting a resource that 
exists under a different method.  I promise you will get a "Not Found" message.  
Then again the only golang router in existence that I know of that handles RFC 2616
correctly is [vestigo][vestigo].

Looking closer at this router bolt on, I have [discovered this][whaaa] which pains
me a bit to talk about.  So not only does the HTTPRouter code alloc key value pair
space on the heap for storing the URL parameters, but then siesta *also* allocs 
heap space to *also* store the parameters within the form data, like a Pat or 
vestigo does.

## Credit Where Due

I will mention that I very much like like the "flag"-esc helper functions to get
form and URL parameters based on type.  It reminds me of viper/flag a lot, and 
that is a very useful aspect of this web framework.


[siesta]: https://github.com/VividCortex/siesta
[threadsafe-context]: https://github.com/VividCortex/siesta/blob/b371862cfbc0199774d3a711e7e53da6bc8cdf2c/context.go#L27-L48
[handler-wrap]: https://github.com/VividCortex/siesta/blob/b371862cfbc0199774d3a711e7e53da6bc8cdf2c/handler.go#L20-L51
[not-allow]: https://husobee.github.io/golang/vestigo/405/2015/09/29/method-not-found.html
[vestigo]: https://github.com/husobee/vestigo
[whaaa]: https://github.com/VividCortex/siesta/blob/b371862cfbc0199774d3a711e7e53da6bc8cdf2c/service.go#L126-L128
[router-hate]: https://husobee.github.io/golang/url-router/2015/06/15/why-do-all-golang-url-routers-suck.html