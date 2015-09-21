---
layout: post
title: "Exploring Echo.. echo... echo..."
date: 2015-09-20 20:00:00
categories: golang webframework echo
---

As some recall, I have already expressed my [url routers][views of certain golang URL routers] before, and
I have been interested in a new one lately, the router that is built into the [master_echo][Echo] web framework.

## Why Echo?

Echo is very interesting to me, as, well honestly, it has one of the fastest Web Router implementation out there, which
is awesome, and backed up by [go-perf-test][this fork] of [go-perf-test-orig][julienschmidt's] go http router benchmark suite.

Here is a shortened performance report of the top url routers using this performance test agains't the huge github api:

{% highlight text %}

BenchmarkAce_GithubAll                     10000            101551 ns/op           13792 B/op        167 allocs/op
BenchmarkBear_GithubAll                    10000            322018 ns/op           84560 B/op        943 allocs/op
BenchmarkBeego_GithubAll                    3000            659363 ns/op          146272 B/op       2092 allocs/op
BenchmarkBone_GithubAll                      500           2857703 ns/op          648016 B/op       8119 allocs/op
BenchmarkDenco_GithubAll                   20000            101808 ns/op           20224 B/op        167 allocs/op
*BenchmarkEcho_GithubAll                    30000             40348 ns/op               0 B/op          0 allocs/op*
BenchmarkGin_GithubAll                     50000             39749 ns/op               0 B/op          0 allocs/op
BenchmarkGocraftWeb_GithubAll               5000            489575 ns/op          131656 B/op       1686 allocs/op
BenchmarkGoji_GithubAll                     3000            602825 ns/op           56112 B/op        334 allocs/op
BenchmarkGoJsonRest_GithubAll               3000            619631 ns/op          137416 B/op       2737 allocs/op
BenchmarkGoRestful_GithubAll                 100          16725309 ns/op          837832 B/op       6913 allocs/op
BenchmarkGorillaMux_GithubAll                200           6997254 ns/op          167232 B/op       2469 allocs/op
BenchmarkHttpRouter_GithubAll              20000             67171 ns/op           13792 B/op        167 allocs/op
BenchmarkHttpTreeMux_GithubAll              5000            221756 ns/op           65856 B/op        671 allocs/op
BenchmarkKocha_GithubAll                   10000            179933 ns/op           26016 B/op        843 allocs/op
BenchmarkMacaron_GithubAll                  2000            750287 ns/op          201138 B/op       1803 allocs/op
BenchmarkMartini_GithubAll                   200           6372486 ns/op          228214 B/op       2483 allocs/op
BenchmarkPat_GithubAll                       300           5161129 ns/op         1546352 B/op      27435 allocs/op
BenchmarkPossum_GithubAll                  10000            290861 ns/op           97440 B/op        812 allocs/op
BenchmarkR2router_GithubAll                10000            257402 ns/op           77328 B/op        979 allocs/op
BenchmarkRevel_GithubAll                    1000           1511440 ns/op          343936 B/op       5512 allocs/op
BenchmarkRivet_GithubAll                    5000            256087 ns/op           84272 B/op       1079 allocs/op
BenchmarkTango_GithubAll                    5000            462216 ns/op           90323 B/op       2267 allocs/op
BenchmarkTigerTonic_GithubAll               2000           1168838 ns/op          244640 B/op       5035 allocs/op
BenchmarkTraffic_GithubAll                   200           8561435 ns/op         2666096 B/op      21848 allocs/op
BenchmarkVulcan_GithubAll                   5000            300177 ns/op           22736 B/op        609 allocs/op


{% endhighlight %}

As you can see [master_echo][Echo] is a top contender for complex api routing, moreover it is much faster than the standard
library implementation, and has a smaller memory footprint _(did you see the 0B alloc/op??)_ which makes it an awesome choice for a 
small fast url router implementation.

## Sounds Great, What is the problem?

Well, like anything else, you can't have great performance without robbing from somewhere else.  In [master_echo][Echo's] case
we have the following issues from what I can see:

1. No Regular Expression Matching on URL parameters
2. Required to use the Echo Context Scheme to get url parameters
3. Reports back 404 when 405 is more correct
4. No capability for Requesting of the Web Service what it is capable of [options][using OPTIONS]

Basically, a lot of the issues I have laid out in my [url_routers][prior blog on url routers].  But, I decided to see if I couldn't help
correct some of these issues, as I really like the potential within this implementation.

## How does Echo's URL Router Work?

Echo has followed a good design on how to handle routing from a high level.  The router is created as a radix tree datastructure allowing
for efficient string searching capabilities.  This implementation currently contains a radix tree per HTTP method as seen [multiple-trees][here].
When a route is searched on, the first thing that is done in Router.Find, is isolating the correct method radix tree to use:

{% highlight go %}
func (r *Router) findTree(method string) (n *node) {
    switch method[0] {
    case 'G': // GET
        m := uint32(method[2])<<8 | uint32(method[1])<<16 | uint32(method[0])<<24
        if m == 0x47455400 {
            n = r.getTree
        }
    case 'P': // POST, PUT or PATCH
        switch method[1] {
        case 'O': // POST
            m := uint32(method[3]) | uint32(method[2])<<8 | uint32(method[1])<<16 |
                uint32(method[0])<<24
            if m == 0x504f5354 {
                n = r.postTree
            }
        case 'U': // PUT
            m := uint32(method[2])<<8 | uint32(method[1])<<16 | uint32(method[0])<<24
            if m == 0x50555400 {
                n = r.putTree
            }
// continued...

{% endhighlight %}

As you can see in the above snip-it, We are finding the tree to use based on method.  This immediately causes item #3 and #4 to become extremely hard to do, as
within the POST tree, we have no insight as to what is available in the GET tree.  Imagine we have one endpoints shown below:

{% highlight go %}
    e := echo.New()
    e.Get("/hi")
{% endhighlight %}

What happens when we do the following:

{% highlight text %}
    curl -XPOST http://localhost/hi
{% endhighlight %}

We end up getting a HTTP Status Code of 404, or "Resource Not Found" as a response.  This isn't really true, as the resource */hi* exists, we just aren't allowed to use
that method....  We should be getting back a 405, or "Method Not Allowed" response back.  Moreover, to point #4 above, we also get a 404 if we use an OPTIONS HTTP method
as the OPTIONS tree isn't populated with this route. To these ends I have [405_pull][submitted a PR] which is still being fleshed out to improve the speed implications.

## What does this PR do?

[405_pull][This PR] basically eliminates the per method tree structure in favor of a full tree combined for all routes, which contains all methods applicable for each leaf node route
that is specified.  Below is a snip-it of the visualization of the Github API loaded into this new single tree. You can pretty clearly see the available methods on particular routes.

{% highlight text %}
└── /, 0xc820423920: type=0, parent=0x0, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> }
    ├── a, 0xc8201b5a40: type=0, parent=0xc820423920, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> }
    │   ├── uthorizations, 0xc820423c20: type=0, parent=0xc820423920, handler=&{<nil> <nil> 0x48c0d0 <nil> <nil> <nil> 0x48c0d0 <nil> <nil> GET, POST}
    │   │   └── /, 0xc820423a40: type=0, parent=0xc820423920, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> GET}
    │   │       └── :, 0xc820423b00: type=1, parent=0xc820423a40, handler=&{<nil> 0x48c0d0 0x48c0d0 <nil> <nil> <nil> <nil> <nil> <nil> GET, DELETE}
    │   └── pplications/, 0xc820423d40: type=0, parent=0xc820423920, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> GET}
    │       └── :, 0xc820423e00: type=1, parent=0xc820423d40, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> GET}
    │           └── /tokens, 0xc820423ec0: type=0, parent=0xc820423e00, handler=&{<nil> 0x48c0d0 <nil> <nil> <nil> <nil> <nil> <nil> <nil> DELETE}
    │               └── /, 0xc8201b4f60: type=0, parent=0xc820423ec0, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> GET}
    │                   └── :, 0xc820423f80: type=1, parent=0xc820423ec0, handler=&{<nil> 0x48c0d0 0x48c0d0 <nil> <nil> <nil> <nil> <nil> <nil> GET, DELETE}
    ├── e, 0xc8201b2480: type=0, parent=0xc820423920, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> }
    │   ├── vents, 0xc8204fe060: type=0, parent=0xc8201b2480, handler=&{<nil> <nil> 0x48c0d0 <nil> <nil> <nil> <nil> <nil> <nil> GET}
    │   └── mojis, 0xc8204fe180: type=0, parent=0xc8201b2480, handler=&{<nil> <nil> 0x48c0d0 <nil> <nil> <nil> <nil> <nil> <nil> GET}
    ├── r, 0xc8201b26c0: type=0, parent=0xc820423920, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> }
    │   ├── epos, 0xc8204fe900: type=0, parent=0xc8201b26c0, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> }
    │   │   ├── /, 0xc82029c600: type=0, parent=0xc8204fe900, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> GET}
    │   │   │   └── :, 0xc8201b2a80: type=1, parent=0xc8201b26c0, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> GET}
    │   │   │       └── /, 0xc8201b2ea0: type=0, parent=0xc8201b2a80, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> GET}
    │   │   │           └── :, 0xc8201b3260: type=1, parent=0xc8201b2ea0, handler=&{<nil> 0x48c0d0 0x48c0d0 <nil> <nil> <nil> <nil> <nil> <nil> GET, GET, DELETE}
    │   │   │               └── /, 0xc8201b35c0: type=0, parent=0xc8201b3260, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> }
    │   │   │                   ├── events, 0xc82015ec60: type=0, parent=0xc8201b35c0, handler=&{<nil> <nil> 0x48c0d0 <nil> <nil> <nil> <nil> <nil> <nil> GET}
    │   │   │                   ├── notifications, 0xc82015ed80: type=0, parent=0xc8201b35c0, handler=&{<nil> <nil> 0x48c0d0 <nil> <nil> <nil> <nil> 0x48c0d0 <nil> GET, PUT}
    │   │   │                   ├── s, 0xc82015f080: type=0, parent=0xc8201b35c0, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> }
    │   │   │                   │   ├── ta, 0xc82015f6e0: type=0, parent=0xc82015f080, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> }
    │   │   │                   │   │   ├── rgazers, 0xc8201828a0: type=0, parent=0xc82015f6e0, handler=&{<nil> <nil> 0x48c0d0 <nil> <nil> <nil> <nil> <nil> <nil> GET}
    │   │   │                   │   │   └── t, 0xc8201829c0: type=0, parent=0xc82015f6e0, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> }
    │   │   │                   │   │       ├── s/, 0xc8201830e0: type=0, parent=0xc8201829c0, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> }
    │   │   │                   │   │       │   ├── co, 0xc820182d20: type=0, parent=0xc8201829c0, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> }
    │   │   │                   │   │       │   │   ├── ntributors, 0xc820182a80: type=0, parent=0xc8201829c0, handler=&{<nil> <nil> 0x48c0d0 <nil> <nil> <nil> <nil> <nil> <nil> GET}
    │   │   │                   │   │       │   │   ├── mmit_activity, 0xc820182ba0: type=0, parent=0xc8201829c0, handler=&{<nil> <nil> 0x48c0d0 <nil> <nil> <nil> <nil> <nil> <nil> GET}
    │   │   │                   │   │       │   │   └── de_frequency, 0xc820182c60: type=0, parent=0xc8201829c0, handler=&{<nil> <nil> 0x48c0d0 <nil> <nil> <nil> <nil> <nil> <nil> GET}
    │   │   │                   │   │       │   └── p, 0xc820182e40: type=0, parent=0xc8201829c0, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> }
    │   │   │                   │   │       │       ├── articipation, 0xc820182f00: type=0, parent=0xc820182e40, handler=&{<nil> <nil> 0x48c0d0 <nil> <nil> <nil> <nil> <nil> <nil> GET}
    │   │   │                   │   │       │       └── unch_card, 0xc820183020: type=0, parent=0xc820182e40, handler=&{<nil> <nil> 0x48c0d0 <nil> <nil> <nil> <nil> <nil> <nil> GET}
    │   │   │                   │   │       └── uses/, 0xc820183200: type=0, parent=0xc8201829c0, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> GET}
    │   │   │                   │   │           └── :, 0xc8201832c0: type=1, parent=0xc820183200, handler=&{<nil> <nil> 0x48c0d0 <nil> <nil> <nil> 0x48c0d0 <nil> <nil> GET, POST}
    │   │   │                   │   └── ubscri, 0xc82015f800: type=0, parent=0xc82015f080, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> }
    │   │   │                   │       ├── bers, 0xc82015fc80: type=0, parent=0xc82015f800, handler=&{<nil> <nil> 0x48c0d0 <nil> <nil> <nil> <nil> <nil> <nil> GET}
    │   │   │                   │       └── ption, 0xc82015fda0: type=0, parent=0xc82015f800, handler=&{<nil> 0x48c0d0 0x48c0d0 <nil> <nil> <nil> <nil> 0x48c0d0 <nil> GET, PUT, DELETE}
    │   │   │                   ├── git/, 0xc82015d980: type=0, parent=0xc8201b35c0, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> }
    │   │   │                   │   ├── blobs, 0xc8202781e0: type=0, parent=0xc82015d980, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> 0x48c0d0 <nil> <nil> POST}
    │   │   │                   │   │   └── /, 0xc8202780c0: type=0, parent=0xc82015d980, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> GET}
    │   │   │                   │   │       └── :, 0xc820278000: type=1, parent=0xc82015d980, handler=&{<nil> <nil> 0x48c0d0 <nil> <nil> <nil> <nil> <nil> <nil> GET}
    │   │   │                   │   ├── commits, 0xc820278300: type=0, parent=0xc82015d980, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> 0x48c0d0 <nil> <nil> POST}
    │   │   │                   │   │   └── /, 0xc820278480: type=0, parent=0xc820278300, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> GET}
    │   │   │                   │   │       └── :, 0xc8202783c0: type=1, parent=0xc820278300, handler=&{<nil> <nil> 0x48c0d0 <nil> <nil> <nil> <nil> <nil> <nil> GET}
    │   │   │                   │   ├── refs, 0xc8202785a0: type=0, parent=0xc82015d980, handler=&{<nil> <nil> 0x48c0d0 <nil> <nil> <nil> 0x48c0d0 <nil> <nil> GET, POST}
    │   │   │                   │   └── t, 0xc820278660: type=0, parent=0xc82015d980, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> }
    │   │   │                   │       ├── ags, 0xc820278900: type=0, parent=0xc820278660, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> 0x48c0d0 <nil> <nil> POST}
    │   │   │                   │       │   └── /, 0xc8202787e0: type=0, parent=0xc820278660, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> GET}
    │   │   │                   │       │       └── :, 0xc820278720: type=1, parent=0xc820278660, handler=&{<nil> <nil> 0x48c0d0 <nil> <nil> <nil> <nil> <nil> <nil> GET}
    │   │   │                   │       └── rees, 0xc820278a20: type=0, parent=0xc820278660, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> 0x48c0d0 <nil> <nil> POST}
    │   │   │                   │           └── /, 0xc820278ba0: type=0, parent=0xc820278a20, handler=&{<nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> <nil> GET}
    │   │   │                   │               └── :, 0xc820278ae0: type=1, parent=0xc820278a20, handler=&{<nil> <nil> 0x48c0d0 <nil> <nil> <nil> <nil> <nil> <nil> GET}
{% endhighlight %}

The result of combining all the method trees into a single tree wasn't inconsequential as can be seen below (about 8.5K ns slower), but then again i am working on ways to speed up the implementation, by
possibly bringing back the per method tree in conjunction with a full tree.  When the router is instantiated, it will populate the full tree with all of the routes and related methods into one
tree, then create per method trees, and when a leaf node is created, perform a find on the full tree for that url path and grab all of the applicable method handlers.  This will give 
the benefit of the per method tree speed, with the ability to stay within the HTTP specification, and hopefully allow for self documenting apis.

{% highlight text %}
BenchmarkEcho_GithubAll                    30000             48911 ns/op               0 B/op          0 allocs/op
{% endhighlight %}

## Next Steps

I believe I am going to take a look at what it would take to put the URL variables _(/hi/*:name*/*:action*)_ into the HTTP request itself, in order to not have the router tied specifically
to the context concept.  I feel like this will be an easy task, and allow for the router itself to become portable, without having to bend your handlers to include this particular context implementation.


[url_routers]: http://husobee.github.io/golang/url-router/2015/06/15/why-do-all-golang-url-routers-suck.html
[master_echo]: https://github.com/labstack/echo/
[405_pull]: https://github.com/labstack/echo/pull/205
[go-perf-test-orig]: https://github.com/julienschmidt/go-http-routing-benchmark
[go-perf-test]: https://github.com/vishr/go-http-routing-benchmark
[options]: http://zacstewart.com/2012/04/14/http-options-method.html
[multiple-trees]: https://github.com/labstack/echo/blob/49440e762b68617d93d533b06846c83450cdd299/router.go#L7-L15
