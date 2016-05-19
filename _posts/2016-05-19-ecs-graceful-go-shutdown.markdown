---
layout: post
title: "Graceful Shutdown of Go App in AWS ECS"
date: 2016-05-19 08:00:00
categories: golang ecs
---

At work we came across a problem set where we needed to stop one of our golang application gracefully
in Amazon ECS.  This application Shepard's many transactions that are in flight, and it is not acceptable
for us to merely stop the container without at least changing state of those transactions, if not actually
completing the transactions.

After some research into how docker stops containers it seems that the following mapping is how docker tells
containers to stop processing:

* `docker stop` => `kill -SIGTERM` then after a time period `kill -SIGKILL`
* `docker rm -f` => `kill -SIGKILL`

Basically if you do a docker stop to a container PID 1 in that container will be handed a SIGTERM signal,
and told to terminate.  This is great, because we can write our code to handle that signal appropriately
and stop accepting new transactions, in order to clean up the old transactions before shutdown.

As seen in this [excellent article][century-docker] `docker stop` takes a timeout parameter, which should default
to 30 seconds.  After that 30 second timer ends, docker will send a `SIGKILL` to the container process 
and all hope will be lost, as there is no way to intercept a SIGKILL.

What is really enlightening about that article is the fact that if you do not construct your dockerfile right
using the `exec` CMD syntax, your application will start from a parent `/bin/sh` which _will not_ broadcast
the signals received to the child process, which is the thing we want to signal.  By making our Dockerfile
look like the below we will be running `/app` as PID 1 inside the container:

{% highlight text %}
FROM debian:jessie
ADD app /app
CMD ["/app"] # this is the exec syntax as opposed to `CMD /app`
{% endhighlight %}

Below you can see, I made [the following gist][gist] as a proof of concept golang service
that will watch for the appropriate signal from docker:

{% highlight go %}
func main() {
	// create a "returnCode" channel which will be the return code of the application
	var returnCode = make(chan int)

	// finishUP channel signals the application to finish up
	var finishUP = make(chan struct{})

	// done channel signals the signal handler that the application has completed
	var done = make(chan struct{})
	
	// gracefulStop is a channel of os.Signals that we will watch for -SIGTERM
	var gracefulStop = make(chan os.Signal)
	
	// watch for SIGTERM and SIGINT from the operating system, and notify the app on
	// the gracefulStop channel
	signal.Notify(gracefulStop, syscall.SIGTERM)
	signal.Notify(gracefulStop, syscall.SIGINT)
	
	// launch a worker whose job it is to always watch for gracefulStop signals
	go func() {
		// wait for our os signal to stop the app
		// on the graceful stop channel
		// this goroutine will block until we get an OS signal
		sig := <-gracefulStop
		fmt.Printf("caught sig: %+v", sig)
		
		// send message on "finish up" channel to tell the app to
		// gracefully shutdown
		finishUP<-struct{}{}
		
		// wait for word back if we finished or not
		select {
		case <-time.After(30*time.Second):
			// timeout after 30 seconds waiting for app to finish,
			// our application should Exit(1)
			returnCode<-1
		case <-done:
			// if we got a message on done, we finished, so end app
			// our application should Exit(0)
			returnCode<-0
		}
	}()
	

	// ... Do business Logic in goroutines

	fmt.Println("waiting for finish")
	// wait for finishUP channel write to close the app down
	<-finishUP
	fmt.Println("stopping things, might take 2 seconds")

	// ... Do business Logic for shutdown simulated by Sleep 2 seconds
	time.Sleep(2*time.Second)

	// write to the done channel to signal we are done.
	done <-struct{}{}
	os.Exit(<-returnCode)
}
{% endhighlight %}

When the above running application receives a kill -SIGTERM or kill -SIGINT it will be caught by
our signal watcher worker anonymous function, which in turn will signal the main app to signal
all of it's worker goroutines to shutdown.  After business logic for shutdown occurs, we signal
back to the signal watcher worker we have completed on the done channel, which will then write 
an appropriate code to the returnCode.

Since we use ECS in Amazon for our container deployment, and de-deployment, we needed to make 
sure that ECS was using the same docker stop mechanism to make sure this will work, [and behold][ecs-stop-container]
it does use the docker client StopContainer function to stop containers.

I guess the point is, there are times when your app needs to be responsible on shutdown,
and you should make every effort to clean up after yourself, even if you are being signaled to quit.

[ecs-stop-container]: https://github.com/aws/amazon-ecs-agent/blob/0931217222db278f78be3d246076261a5e702720/agent/engine/docker_container_engine.go#L448
[gist]: https://gist.github.com/husobee/531062f48f7f35688e7e4c63139a9616
[century-docker]: https://www.ctl.io/developers/blog/post/gracefully-stopping-docker-containers/
