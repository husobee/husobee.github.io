---
layout: post
title: "Why do I not like any Golang URL Routers?"
date: 2015-06-15 20:00:00
categories: golang url-router
---

For those who don't know, URL routing is the practice of taking a URL partial, and mapping
it to some form of request handler.  This enables one to create a structured URL partial path
and "route" that traffic to a function that takes in a request and a response stream.  There 
are a few options out there for Gophers to use that are not strictly tied to a particular web
framework that I have looked at and have opinions on.

# net/http.ServeMux [docs][net-http-servemux]

Lets start with the basics!  Built into the standard library is a basic implementation of a URL
router.  To the Code! [net/http.ServeMux][net-http-servemux-code] (parts chopped to save space)

{% highlight go %}
type ServeMux struct {
    mu    sync.RWMutex
    m     map[string]muxEntry
    hosts bool // whether any patterns contain hostnames
}

type muxEntry struct {
    explicit bool
    h        Handler
    pattern  string
}

// Does path match pattern?
func pathMatch(pattern, path string) bool {
    if len(pattern) == 0 {
        // should not happen
        return false
    }
    n := len(pattern)
    if pattern[n-1] != '/' {
        return pattern == path
    }
    return len(path) >= n && path[0:n] == pattern
}

// Find a handler on a handler map given a path string
// Most-specific (longest) pattern wins
func (mux *ServeMux) match(path string) (h Handler, pattern string) {
    var n = 0
    for k, v := range mux.m {
        if !pathMatch(k, path) {
            continue
        }
        if h == nil || len(k) > n {
            n = len(k)
            h = v.h
            pattern = v.pattern
        }
    }
    return
}
{% endhighlight %}

This is a fairly understandable url router, in that the type contains a route map of strings to muxEntry,
which contains a handler and a url pattern.  This allows us to create a pattern for a url, such as
*/amazing/path/*.  Sounds great, out of the box and we have a mechanism to route urls to handlers.

This router implementation works, but it works for very narrowly focused applications with fairly static routes.
If you want, say, URL parameters in your url such as */amazing/<customer_id>/path/* you are not going to 
have very much success with this URL router.

Reviewing the matching logic you can see that ServeMux.match takes a path, and then iterates over the entire
range of routes to muxEntries map on every request and then performs a check to see if the pattern exactly matches
any routes.  This is *very strict*, no wildcards, no url parameters, and not a very efficient datastructure choice for
applications with lots of url routes.

# Gorilla Mux [docs][gorilla-mux]

Gorilla Mux is another router implementation that takes care of the need for dynamic url routes pretty well. 
With a line like the one listed below, we can also specify which http methods are allowed on what url routes!  This is 
pretty sweet, because we can really lock down our API and make sure we know what we are supporting method wise:

{% highlight go %}
r := mux.NewRouter()
r.Methods("GET", "POST").HandleFunc("/products/{key}", ProductHandler)
{% endhighlight %}

We can pass in a standard net/http Handler to this router, and amazingly we can get the variables within "{}" within the 
handler function by performing a call to *mux.Vars()*.  This is a pretty sweet router.  Lets take a look at how this works:

{% highlight go %}

type Router struct {
    // Configurable Handler to be used when no route matches.
    NotFoundHandler http.Handler
    // Parent route, if this is a subrouter.
    parent parentRoute
    // Routes to be matched, in order.
    routes []*Route
    // Routes by name for URL building.
    namedRoutes map[string]*Route
    // See Router.StrictSlash(). This defines the flag for new routes.
    strictSlash bool
    // If true, do not clear the request context after handling the request
    KeepContext bool
}

// Match matches registered routes against the request.
func (r *Router) Match(req *http.Request, match *RouteMatch) bool {
    for _, route := range r.routes {
        if route.Match(req, match) {
            return true
        }
    }
    return false
}

{% endhighlight %}

The router looks like a slice of Route Pointers.  Also it appears that the route matching is performed by iterating 
over that slice, and calling "Match" on each route that is registered with the router.  Below is an excerpt of how Matching
is done in the Route structure:

{% highlight go %}
// Match matches the route against the request.
func (r *Route) Match(req *http.Request, match *RouteMatch) bool {
    if r.buildOnly || r.err != nil {
        return false
    }
    // Match everything.
    for _, m := range r.matchers {
        if matched := m.Match(req, match); !matched {
            return false
        }
    }
    // Yay, we have a match. Let's collect some info about it.
    if match.Route == nil {
        match.Route = r
    }
    if match.Handler == nil {
        match.Handler = r.handler
    }
    if match.Vars == nil {
        match.Vars = make(map[string]string)
    }
    // Set variables.
    if r.regexp != nil {
        r.regexp.setMatch(req, match, r)
    }
    return true
}
{% endhighlight %}

Looking at the Route.Match method we can see that a route is using a list of "matchers" to validate that the url route is 
matched to the http.Request that is incomming.  You can see in here that on a match a match.Vars map is created to hold the 
matched variables from the url for our dynamic routes. This seems like an okay-ish approach to me, but I am wondering where these 
matchers come from, and how they are made.  Specifically how do the method matches work?

{% highlight go %}
// Methods --------------------------------------------------------------------

// methodMatcher matches the request against HTTP methods.
type methodMatcher []string

func (m methodMatcher) Match(r *http.Request, match *RouteMatch) bool {
    return matchInArray(m, r.Method)
}

// Methods adds a matcher for HTTP methods.
// It accepts a sequence of one or more methods to be matched, e.g.:
// "GET", "POST", "PUT".
func (r *Route) Methods(methods ...string) *Route {
    for k, v := range methods {
        methods[k] = strings.ToUpper(v)
    }
    return r.addMatcher(methodMatcher(methods))
}

{% endhighlight %}

Ruh, Roh... Mux is using the method as a matcher... How I read this is, since a matcher is an implementation of an interface with a boolean 
response only, they can't possibly pass any more information back when the matcher fails.  Furthermore since they are re-using this matching
concept, we are going to end up getting 404's as responses when we don't match an exact HTTP method.. This sucks, as it is completely in violation
of the [RFC-2616][rfc2616] if you specify the methods in your matching.

I really appreciate the use of interfaces, and implementation of those interfaces with the various matchers, but sometimes the matchers shouldn't 
treated the same way, and I really have a feeling that method matching was a bolt on addition to this url router.

# Gorilla Pat[docs][gorilla-pat]

Maybe Gorilla's "other" url router will be better?  Lets take a look at the source code real quick to see what we are getting into:

{% highlight go %}

// Router is a request router that implements a pat-like API.
//
// pat docs: http://godoc.org/github.com/bmizerany/pat
type Router struct {
    mux.Router
}

// Add registers a pattern with a handler for the given request method.
func (r *Router) Add(meth, pat string, h http.Handler) *mux.Route {
    return r.NewRoute().PathPrefix(pat).Handler(h).Methods(meth)
}

// Delete registers a pattern with a handler for DELETE requests.
func (r *Router) Delete(pat string, h http.HandlerFunc) *mux.Route {
    return r.Add("DELETE", pat, h)
}

// Get registers a pattern with a handler for GET requests.
func (r *Router) Get(pat string, h http.HandlerFunc) *mux.Route {
    return r.Add("GET", pat, h)
}

{% endhighlight %}

Nice, we can use this syntax: *r.Get("/path/{var}/", handler)*, but, whaaaa? wait a second... Are you KIDDING ME?

Well this router is just a wrapper of Gorilla Mux with some nicities.  Unfortunately, it will suffer from the exact same
problems I had with Mux.  I do however like how in the below they are re-writing the URL variables inside of the url query
string!  This means that the handler will have access to the parameters the same way it accesses GET and POST data!

{% highlight go %}
// ServeHTTP dispatches the handler registered in the matched route.
func (r *Router) ServeHTTP(w http.ResponseWriter, req *http.Request) {
    // Clean path to canonical form and redirect.
    if p := cleanPath(req.URL.Path); p != req.URL.Path {
        w.Header().Set("Location", p)
        w.WriteHeader(http.StatusMovedPermanently)
        return
    }
    var match mux.RouteMatch
    var handler http.Handler
    if matched := r.Match(req, &match); matched {
        handler = match.Handler
        registerVars(req, match.Vars)
    }
    if handler == nil {
        if r.NotFoundHandler == nil {
            r.NotFoundHandler = http.NotFoundHandler()
        }
        handler = r.NotFoundHandler
    }
    defer context.Clear(req)
    handler.ServeHTTP(w, req)
}

// registerVars adds the matched route variables to the URL query.
func registerVars(r *http.Request, vars map[string]string) {
    parts, i := make([]string, len(vars)), 0
    for key, value := range vars {
        parts[i] = url.QueryEscape(":"+key) + "=" + url.QueryEscape(value)
        i++
    }
    q := strings.Join(parts, "&")
    if r.URL.RawQuery == "" {
        r.URL.RawQuery = q
    } else {
        r.URL.RawQuery += "&" + q
    }
}


{% endhighlight %}

Neat Idea! But still have issues with this package due to the underlying package!


# HTTPRouter [docs][httprouter]

oooooooohhh!  The benchmarks of this implementation are *very* impressive.  And the high level overview of how the route matching works seems very 
nice, using Radix Tree data structure for building url paths.  Makes sense, since in the end we are string matching routes!  It does allow for "named parameters"
which is usually a must have for api's these days.  I did notice something a little annoying about how this package deals with named parameters though on the 
github page:

{% highlight text %}
Note: Since this router has only explicit matches, you can not register static routes and parameters for the same path segment. For example you can not register the patterns /user/new and /user/:user for the same request method at the same time. The routing of different request methods is independent from each other.
{% endhighlight %}

Well shoot, throw a url scheme like this out the window: */path/:customer_id/name* and */path/:customer_id/email*

Also, the real elephant in the room is how these named parameters are passed into our handlers; your handler will need to accept a third Parameter for the named parameters
just for this url router.  This is really painful because it almost feels like you are locked into this url package for the long run.  Why on earth didn't this package
follow a more "conventional" model of using a per request context, or url rewrites to accomplish this?

In conclusion, I would like to say that everyone's design ideas are different about URL routing.  They all bring benefit to the ecosystem, but what I really want
is something that will use Pat's URL Query String rewriting, and HTTPRouter's Radix Tree matching.  Maybe that will be something I can make ;)


*UPDATE* - I have followed through on the above conclusion and created  [Vestigo][vestigo]!

[net-http-servemux]: http://golang.org/pkg/net/http/#ServeMux
[net-http-servemux-code]: http://golang.org/src/net/http/server.go?s=41918:42043#L1411
[gorilla-mux]: http://www.gorillatoolkit.org/pkg/mux
[gorilla-pat]: http://www.gorillatoolkit.org/pkg/pat
[httprouter]: https://github.com/julienschmidt/httprouter
[rfc2616]: http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html
[vestigo]: https://husobee.github.io/golang/urlrouter/vestigo/2015/09/22/vesigo.html
