---
layout: post
title: "Getting Code Coverage from External Testing"
date: 2015-11-17 08:00:00
categories: golang test coverage
---

## External Testing

In past posts [I have talked a bit about behavior testing][behaviors].  I honestly feel
like this is the best way to clearly articulate across functional teams, namely
product, quality assurance and development, the requirements and expected behaviors
a software system should obey.  Up until last week I had always maintained that
unit tests are there for coverage, and behavior tests are there to make the 
company happy we are performing as expected with our software systems.  I mean
it can't possibly be practical to get real server code coverage from a consumer,
or external testing solution, right...

## Golang Coverage

Ever wonder how `go test -cover` gives you your code's coverage?  Well, under 
the hood, `go test -cover` actually compiles your executable, but before it 
compiles the code, the cover tool actually re-writes your code with various
telemetry needed.  An excellent primer can be [found here][cover-story].

Armed with this story, I set out on some experimentation to see if it was possible
to actually see a coverage report from the copious volumes of behavior testing
the company has been amassing.  My first inkling is to see if there was a way to
generate a static executable with all of the coverage magic built into it. Turns
out that is really easy, as seen below([and on this gist][covergist]):

{% highlight bash %}
# first to see if we can make an executable
# the -c flag tells the test tool to compile by name
# and the -o flag tells the test tool to just output
# the executable created from the go tests, instead of running
go test -c main.go main_test.go -o test-able-exe
# armed with this, lets actually build in the coverage magic
go test -coverprofile=coverage.out -c main.go main_test.go -o test-able-exe
{% endhighlight %}

This is great.  Now lets figure out how we can wrap our `main()` function in a
test so we can make a running http server that will collect this code coverage
telemetry.  Below is the main.go I have created to accomplish this, with comments
inline explaining what is going on.

{% highlight go %}
package main

import (
        "fmt"
        "net/http"
        "time"

        "github.com/codegangsta/negroni"
        "github.com/husobee/vestigo"
        "github.com/tylerb/graceful"
)

var ( 
		// srv is tylerb's graceful server, which allows us to turn the 
		// server off at will within the code.  This is super handy for
		// us because we want to be able to end our test, in order for
		// go's testing framework to report the coverage (no good if
		// the service is interrupted or canceled or terms)
        srv = &graceful.Server{
                Timeout: 5 * time.Second,
        } 
		// stop is a channel that tells the service to stop.  As seen
		// later we will make a highly destructive deathblow endpoint
		// so that in test we can conclude the test and turn the service off.
        stop     chan bool
		// testMode is a bool that allows for deathpunch endpoint to exist or 
		// not exist... we don't want that running in production ;)
        testMode bool = false
)

// runMain - entry-point to perform external testing of service, this is 
// where go test will enter main.  we have to setup test mode in here, as 
// well as the stop channel so we can stop the service
func runMain() {
        // start the stop channel
        stop = make(chan bool)
        // put the service in "testMode"
        testMode = true
        // run the main entry point
        go main()
        // watch for the stop channel
        <-stop
        // stop the graceful server
        srv.Stop(5 * time.Second)
}
 
// main - main entry point
func main() {
        // setup middlware stack
        n := negroni.Classic()

        // setup routes
        router := vestigo.NewRouter()

        // endpoints
        router.Post("/test", func(w http.ResponseWriter, r *http.Request) {
                if false {
                        // we should see this endpoint not covered
                        // if we hit the /test endpoint externally
                        fmt.Println("totally never getting here")
                }
                w.WriteHeader(200)
                w.Write([]byte("done"))
        })
		// all of the above is basic boring service setup stuff.

        // only if we are in testMode should we attempt to add the death blow
        if testMode {
                // death blow endpoint - endpoint that will stop the service if stop is
                // a live channel (only live if started from RunMain)
                router.Post("/deathblow", func(w http.ResponseWriter, r *http.Request) {
                        // end the graceful server if being run from RunMain()
                        stop <- true
                })
        }

        // add our router to negroni
        n.UseHandler(router)

        // graceful start/stop server
        srv.Server = &http.Server{Addr: ":1234", Handler: n}
        // serve http
        srv.ListenAndServe()
}
{% endhighlight %}

## The Test!

Armed with the above design, you can already see what the test will look like,
basically a "TestMain" function that kicks off `runMain()`.  The normal 
`go build` command will never run `runMain()` and thereby not setup the 
deathblow endpoint that kills the service.  Below is our super easy test:

{% highlight go %}
package main

import (
        "testing"
)

// TestMain - test to drive external testing coverage
func TestMain(t *testing.T) {
        runMain()
}

{% endhighlight %}*

Pretty simple!  Our new commands to build a coverage enabled executable are as follows:

{% highlight bash %}
go test -coverprofile=coverage.out -c main.go main_test.go -o test-able-exe
./test-able-exe -test.coverprofile=coverage.out -test.v -test.run=TestMain
# or all condensed using go test as follows:
go test -coverprofile=coverage.out -run=TestMain
# in another terminal run these two curl statements:
curl -XPOST http://127.0.0.1:1234/test
curl -XPOST http://127.0.0.1:1234/deathblow
# we can then feel the coverage into the go cover tool to get various outputs:
go tool cover -func=coverage.out 
codecovergist/main.go:22:       runMain         100.0%
codecovergist/main.go:35:       main            92.3%
total:                          (statements)    94.4%
{% endhighlight %}

Hopefully you can impress your local QA peeps by giving them code coverage for
all of your external testing!  Have fun!

[cover-story]: http://blog.golang.org/cover
[behaviors]: http://husobee.github.io/automate/behavior/testing/2015/06/13/automate-behaviors.html
[covergist]: https://gist.github.com/husobee/92f742c851f1083fd3cb
