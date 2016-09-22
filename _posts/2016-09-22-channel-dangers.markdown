---
layout: post
title: "Golang Channel Dangers"
date: 2016-09-22 05:00:00
categories: channel golang danger
---

### Background

I had a problem with a service I was working on today.  The problem was very 
confusing.  Symptoms shown were very strange.  I had implemented a very simple
[worker pool][workerpool] as I have on many projects similar to this:

{% highlight go %}

// ErrTimeout - timeout error
var ErrTimeout = errors.New("timeout")

// workersDone - package signal that workers are done
var workersDone = make(chan bool)

// WorkRequest - inputs for an operation
type WorkRequest struct {
	Ret       chan Result
	Input     []byte
}

// Result - result struct for work results
type Result struct {
	Value []byte
	Err   error
}

// WaitForResult - this is a wrapper function to serve as a helper to listen on the
// result channel of the work item passed in, will timeout in 2 seconds
func (w *WorkRequest) WaitForResult() (Result, error) {
	select {
	case <-time.After(2 * time.Second):
		return Result{}, ErrTimeout
	case result := <-w.Ret:
		return result, nil
	}
}

// worker - worker that is in the pool
func worker(input chan WorkRequest) chan bool {
	quit := make(chan bool)
	go func(input chan WorkRequest, quit chan bool) {
		for {
			select {
			case <-quit:
				workersDone<-true
				return
			case v := <-input:
				// do lots of work!!
				result := &Result{} // this result is the result of all the work..
				v.Ret <- result
			}
		}
	}(input, quit)
	return quit
}

{% endhighlight %}

Pretty simple.  I had this setup to run with a number of workers in a worker
pool.  I put in a timeout in the event a worker takes a long time I would be
able to respond back to the caller in a reasonable amount of time (2 seconds).

Then madness happened.  Due to the way we are doing work, it is very CPU intensive
and when the system gets loaded down the work queue starts backing up.  This makes 
the WorkRequest lifetime extend past it's 2 second timeout window.  The second I
start seeing failures due to timeouts, the system "just locks up" and the service
ends up hanging.

This is really strange.  I started by looking at the work the worker was performing
trying to figure out why it would just stop taking requests... I started looking at 
the possibility of me signaling the workers in the pool to quit...

It turns out the real problem was in how the communication was happening, or not happening
as it turns out, in the `WaitForResult` method call.

Zooming in on that particular code:

{% highlight go %}

// WaitForResult - this is a wrapper function to serve as a helper to listen on the
// result channel of the work item passed in, will timeout in 2 seconds
func (w *WorkRequest) WaitForResult() (Result, error) {
	select {
	case <-time.After(2 * time.Second):
		return Result{}, ErrTimeout
	case result := <-w.Ret:
		return result, nil
	}
}

{% endhighlight %}

It shows pretty clearly here that after 2 seconds, I am returning an empty result,
and an error.  And therein lies the problem.  Since we are returning before we get
any messages on w.Ret, there is no active reader reading w.Ret.  This means that
when the worker tries to write to v.Ret, since no go-routines are listening for
that channel, the worker becomes blocked.

A much better solution is to bring the timeout check closer to the work in which
we are testing the timeout on, and then sending the error on the channel if the timeout expires:

{% highlight go %}

// WaitForResult - this is a wrapper function to serve as a helper to listen on the
// result channel of the work item passed in, will timeout in 2 seconds
func (w *WorkRequest) WaitForResult() (Result, error) {
	result := <-w.Ret
	return result, result.Err
}

// worker - worker that is in the pool
func worker(input chan WorkRequest) chan bool {
	quit := make(chan bool)
	go func(input chan WorkRequest, quit chan bool) {
		for {
			select {
			case <-quit:
				workersDone<-true
				return
			case v := <-input:
				done := make(chan bool)
				var result Result
				go func(){
					// do lots of work!!
					result := &Result{} // this result is the result of all the work..
					done<-true // this result is the result of all the work
				}()
				select {
				case result := <-done:
					v.Ret <-result
				case <-time.After(2*time.Second):
					v.Ret <-&Result{Err:ErrTimeout}
				}
			}
		}
	}(input, quit)
	return quit
}

{% endhighlight %}

[workerpool]: http://husobee.github.io/golang/workerpool/2016/04/26/golang-workerpools.html
