---
layout: post
title: "golang memory leak"
date: 2016-02-11 08:00:00
categories: golang memory leak
---

## background

Yesterday was an exciting day.  We had just launched a golang service that I had
helped develop, and were excited to see it being used.  And then it happened...

*CRISIS* - the built in metrics collection started complaining about the service
using thousands and thousands of goroutines, when in all reality the service was
not being used very much (initial migration of only a few customers).  

## beginnings of an interesting day

When the reports started coming in about the metrics setting off alarms, with an
insane number of ever increasing goroutines, as well as a constant heap size 
growth we all started working very hard to reproduce the problem in our sandboxes.

Following the logic of the web handler that was continuously being, we noticed that
it was making a web service call using http.Client to another service.  We had a
hunch that was where the problem was.

## reproducing

For some reason the goroutines were not being garbage collected by golang.  The 
only thing we could think of was maybe there is something happening inside the 
goroutine that is allocating memory that isn't being marked.

Below is some sample code I made which illustrates what was happening:

{% highlight go %}
package main

import (
	"fmt"
	"io/ioutil"
	"log"
	"net/http"
	"runtime"
)

// signal when goroutine finishes
var done chan bool

func main() {
	// make a new channel
	done = make(chan bool)
	// print number of initial goroutines
	fmt.Printf("goroutines start: %d\n", runtime.NumGoroutine())

	// doSomething 10 times concurrently... think web handler
	// style where each requests kicks off another goroutine
	for i := 0; i < 10; i++ {
		go doSomething()
	}
	// wait for all the goroutines to finish, and return
	for i := 0; i < 10; i++ {
		<-done
	}

	fmt.Printf("goroutines end: %d\n", runtime.NumGoroutine())
}

func doSomething() {
	// signal we are done doing something
	defer func() { done <- true }()
	// perform a web request
	resp, err := http.Get("https://husobee.github.io/")
	if err != nil {
		log.Fatal(err)
	}
	// check the status code of the response
	// if it returns ok, in our example,
	// we dont care about the body, but if
	// not okay, then we need to read the body
	if resp.StatusCode != http.StatusOK {
		_, err = ioutil.ReadAll(resp.Body)
		if err != nil {
			log.Fatal(err)
		}
		defer resp.Body.Close()
	}
}

{% endhighlight %}

As you can tell, our doSomething is a surrogate web handler that calls another
web service, in this case my blog URL.  We perform a get, then check if the status
code is not OK, then we read the response body and defer the closing of that response body.

Pretty standard fare. Well, when you run this, you will notice that the number of goroutines
printed at the end of the application will be in the high 20s.  This means that
GC is not removing those goroutines.

## solution

The problem was not immediately obvious, but if you use the response struct inside
of the goroutine, i.e. check the status code on the response, this will denote
to the garbage collector that you intend on using the resp structure.  When this
happens the standard http.Client initiates a tcp connection and starts transferring
the response into an `io.ReadCloser` which is tracked by the default http.Client,
which will not allow the garbage collection to consume the response OR the goroutine
itself from memory.

Basically, all this said, if we read the documentation:

    // Body represents the response body.
    //
    // The http Client and Transport guarantee that Body is always
    // non-nil, even on responses without a body or responses with
    // a zero-length body. It is the caller's responsibility to
    // close Body. The default HTTP client's Transport does not
    // attempt to reuse HTTP/1.0 or HTTP/1.1 TCP connections
    // ("keep-alive") unless the Body is read to completion and is
    // closed.
    //

They are warning us to close the body of the response, even if it isn't used, or
read in.  Below is the one line change that corrects this problem:

{% highlight go %}
func doSomething() {
	// signal we are done doing something
	defer func() { done <- true }()
	// perform a web request
	resp, err := http.Get("https://husobee.github.io/")
	if err != nil {
		log.Fatal(err)
	}
	defer resp.Body.Close() // close it on defer
	// check the status code of the response
	// if it returns ok, in our example,
	// we dont care about the body, but if
	// not okay, then we need to read the body
	if resp.StatusCode != http.StatusOK {
		_, err = ioutil.ReadAll(resp.Body)
		if err != nil {
			log.Fatal(err)
		}
	}
}

{% endhighlight %}
