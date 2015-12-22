---
layout: post
title: "Golang http middleware implementation in 7 LOC"
date: 2015-12-22 08:00:00
categories: golang http middleware
---

I was recently asked, how I would implement a http middleware scheme in golang.
As is probably fairly evident by my online writing, I am a person of simplicity
and always opt for the simple solution, with the least magic.  Below is my 
middleware solution:

{% highlight go %}
// middleware - a type that describes a middleware, at the core of this 
// implementation a middleware is merely a function that takes a handler
// function, and returns a handler function.
type middleware func(http.HandlerFunc) http.HandlerFunc

// buildChain - a function that takes a handler function, a list of middlewares
// and creates a new application stack as a single http handler
func buildChain(f http.HandlerFunc, m ...middleware) http.HandlerFunc {
    // if there are no more middlewares, we just return the 
    // handlerfunc, as we are done recursing.
	if len(m) == 0 {
		return f
	}
    // otherwise pop the middleware from the list,
    // and call build chain recursively as it's parameter
	return m[0](buildChain(f, m[1:cap(m)]...))
}

// ... later in the code when we are setting up the router

	// set up awesome router ;)
	router := vestigo.NewRouter()
	router.Get("/v:version/private", buildChain(f, 
        RecoveryMiddleware,
        LoggingMiddleware,
        AuthenticationMiddleware,
        //...
    ))

// ... later in code when defining a middleware

// AuthMiddleware - takes in a http.HandlerFunc, and returns a http.HandlerFunc
var AuthMiddleware = func(f http.HandlerFunc) http.HandlerFunc {
	// one time scope setup area for middleware
	return func(w http.ResponseWriter, r *http.Request) {
		// ... pre handler functionality
		fmt.Println("start auth")
		f(w, r)
		fmt.Println("end auth")
		// ... post handler functionality
	}
}


{% endhighlight %}

The above 7 line middleware implementation is very flexible and when you think 
of a middleware as functions that call other functions you can do some pretty 
interesting things.  This is all possible due to the fact that functions are 
first class citizens in golang.  Here is a [sample implementation.][gist]


[gist]: https://gist.github.com/husobee/fd23681261a39699ee37
