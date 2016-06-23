---
layout: post
title: "http implementation fasthttp in golang"
date: 2016-06-23 08:00:00
categories: golang fasthttp
---

In some early morning github trending browsing I stumbled across a web framework
called [Iris][iris], which looks to be really fast, faster in fact that the stdlib,
which seemed strange to me.  After some browsing of the source code I noticed
that Iris is actually using the [fasthttp][fasthttp] library instead of the 
[net/http][nethttp] standard library for serving http.

It is always a bold claim to say you have created a package that is better, faster
and stronger that the implementation of in the standard library.  I would know
from [vestigo][vestigo] url routing.

## Looking into fasthttp

After reviewing how handlers are served from fasthttp, I completely have fallen
for this library.  The creator has done it right.  I have mentioned to colleagues
how I felt the implementation of the stdlib for net/http has been pretty much 
completely due to how they run handlers.  Looking at the
[stdlib implementation][stdlib-imp] below it is pretty clear that this implementation
uses a separate goroutine PER http request.

{% highlight go %}
func (srv *Server) Serve(l net.Listener) error {
	defer l.Close()
	if fn := testHookServerServe; fn != nil {
		fn(srv, l)
	}
	var tempDelay time.Duration // how long to sleep on accept failure
	if err := srv.setupHTTP2(); err != nil {
		return err
	}
	for {
		rw, e := l.Accept()
		if e != nil {
			if ne, ok := e.(net.Error); ok && ne.Temporary() {
				if tempDelay == 0 {
					tempDelay = 5 * time.Millisecond
				} else {
					tempDelay *= 2
				}
				if max := 1 * time.Second; tempDelay > max {
					tempDelay = max
				}
				srv.logf("http: Accept error: %v; retrying in %v", e, tempDelay)
				time.Sleep(tempDelay)
				continue
			}
			return e
		}
		tempDelay = 0
		c := srv.newConn(rw)
		c.setState(c.rwc, StateNew) // before Serve can return
		go c.serve()
	}
}
{% endhighlight %}

Conversely in [fasthttp's implementation][workerpool-imp] there is a workerpool
model as the implementation, seen below:

{% highlight go %}
func (s *Server) Serve(ln net.Listener) error {
	var lastOverflowErrorTime time.Time
	var lastPerIPErrorTime time.Time
	var c net.Conn
	var err error

	maxWorkersCount := s.getConcurrency()
	s.concurrencyCh = make(chan struct{}, maxWorkersCount)
	wp := &workerPool{
		WorkerFunc:      s.serveConn,
		MaxWorkersCount: maxWorkersCount,
		LogAllErrors:    s.LogAllErrors,
		Logger:          s.logger(),
	}
	wp.Start()

	for {
		if c, err = acceptConn(s, ln, &lastPerIPErrorTime); err != nil {
			wp.Stop()
			if err == io.EOF {
				return nil
			}
			return err
		}
		if !wp.Serve(c) {
			s.writeFastError(c, StatusServiceUnavailable,
				"The connection cannot be served because Server.Concurrency limit exceeded")
			c.Close()
			if time.Since(lastOverflowErrorTime) > time.Minute {
				s.logger().Printf("The incoming connection cannot be served, because %d concurrent connections are served. "+
					"Try increasing Server.Concurrency", maxWorkersCount)
				lastOverflowErrorTime = time.Now()
			}

			// The current server reached concurrency limit,
			// so give other concurrently running servers a chance
			// accepting incoming connections on the same address.
			//
			// There is a hope other servers didn't reach their
			// concurrency limits yet :)
			time.Sleep(100 * time.Millisecond)
		}
		c = nil
	}
}
{% endhighlight %}

## So what?

Well, this is a much better implementation for several reasons:

1. The worker pool model is a zero allocation model, as the workers are already
  initialized and are ready to serve, whereas in the stdlib implementation the
  `go c.serve()` has to allocate memory for the goroutine.
2. The worker pool model is easier to tune, as you can increase/decrease the buffer
  size of the number of work units you are able to accept, versus the fire and
  and forget model in the stdlib
3. The worker pool model allows for handlers to be more connected with the server
  through channel communications, if the server needs to shutdown for example, it
  would be able to more easily communicate with the workers than in the stdlib
  implementation
4. The handler function definition signature is better, as it takes in only a 
  context which includes both the request and writer needed by the handler. this
  is HUGELY better than the standard library, as all you get from the stdlib is 
  a request and response writer... The work in go1.7 to include context within 
  the request is pretty much a hack to give people what they really want (context)
  without breaking anyone.

Overall it is just better to write a server with a worker pool model for serving
requests, as opposed to just spawning a "thread" per request, with no way of 
throttling out of the box.

I really hope that the golang net/http maintainers keep this model in mind for
golang version 2. 

[fasthttp]: https://github.com/valyala/fasthttp
[nethttp]: https://golang.org/pkg/net/http
[workerpool-imp]: https://github.com/valyala/fasthttp/blob/master/server.go#L1214-L1258
[stdlib-imp]: https://golang.org/src/net/http/server.go?s=63385:63431#L2097
[iris]: https://github.com/kataras/iris
[vestigo]: https://github.com/husobee/vestigo
