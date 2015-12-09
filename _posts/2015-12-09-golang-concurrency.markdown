---
layout: post
title: "golang concurrency empirical evidence"
date: 2015-12-09 08:00:00
categories: golang concurrency
---

## More Problems with Load

As I have mentioned in my [prior posting][redis-load], we are performing load testing at work
specifically on our authentication system.  While looking at the graphs we 
noticed that a huge amount of time was being spent performing bcrypt password
hashing operations. (TODO: I will have to do a posting on how we implemented telemetry
in our applications)

It almost seemed from the data like bcrypt was causing the login portion of the 
authentication service to start eating itself under load, to the point where the
number of requests and spinning up of goroutines had blown right through the 
point of diminishing returns.  The entire application started slowing down to 
the point where responses were taking several seconds to complete! Yikes.

It would seem that the golang concurrency was too much for itself!

To be fair, it is more likely that our naive hash checking could have been more
optimized, as it turns out, authentication systems do a lot of password hash
checking. 

To mimic this scenario I created [this gist][homework] to mimic what we were
seeing and start to get numbers so we can turn the knobs.

I started with a very simple benchmark tests to get a feel for what the 
performance was and could be generally:

{% highlight go %}

func BenchmarkSerial(b *testing.B) {
	hash, _ := bcrypt.GenerateFromPassword([]byte("super Secret!!!"), 10)

	for i := 0; i < b.N; i++ {
		bcrypt.CompareHashAndPassword(hash, []byte("some string"))
	}
}

func BenchmarkAsManyGoroutinesPossible(b *testing.B) {
	var done = make(chan bool)
	hash, _ := bcrypt.GenerateFromPassword([]byte("super Secret!!!"), 10)

	for i := 0; i < b.N; i++ {
		go func() {
			bcrypt.CompareHashAndPassword(hash, []byte("some string"))
			done <- true
		}()
	}
	for i := 0; i < b.N; i++ {
		<-done
	}
}

{% endhighlight %}

As seen above I performing benchmarks on a serial implementation base case, and 
then benchmarking using as many goroutines as I can.  Below are the results of
those two runs:

{% highlight bash %}
BenchmarkSerial-2                             10         145922320 ns/op
BenchmarkAsManyGoroutinesPossible-2           20          78825761 ns/op
{% endhighlight %}

So far so good, my machine is a Core Two Duo ( don't make fun ).  We got what we
expected, double the performance of doing it in a serial fashion.

I then started making a sample application as, possibly the runs in the golang
benchmarking were too short and not making the test realistic.  Plus there were
not many knobs to turn.

Below is the sample application to test:

{% go highlight %}
package main

import (
	"flag"
	"fmt"
	"time"

	"golang.org/x/crypto/bcrypt"
)

var (
	concurrency int
	bufsize     int
	workerpool  int
	mechanism   string
)

func init() {
	// parse the command line flags
	flag.IntVar(&concurrency, "c", 15, "concurrency to run bcrypts")
	flag.StringVar(&mechanism, "m", "max_out", "mechanism")
	flag.IntVar(&bufsize, "b", 15, "buffer size")
	flag.IntVar(&workerpool, "w", 1, "worker pool size")
	flag.Parse()
}

func main() {
	// create a bcrypted password
	passwords := make(chan string, bufsize)
	hash, _ := bcrypt.GenerateFromPassword([]byte("super Secret!!!"), 10)
	total := make(chan int)

	count := 0
	mainStart := time.Now()
	go func() {
		for {
			count += <-total
		}
	}()

	if mechanism != "max_out" {
		for i := 0; i < workerpool; i++ {
			go func() {
				for p := range passwords {
					bcrypt.CompareHashAndPassword(hash, []byte(p))
					total <- 1
				}
			}()
		}
	}

	// hashcompare a bunch of becrypted passwords
	for i := 0; i < concurrency; i++ {
		go func() {
			// create anonymous goroutines up to number of concurrency
			for {
				// forever, make this goroutine just do
				if mechanism == "max_out" {
					bcrypt.CompareHashAndPassword(hash, []byte("some string"))
					total <- 1
				} else {
					passwords <- "some string"
				}
			}
		}()
	}

	time.Sleep(5 * time.Second)
	fmt.Printf("number of compares: %d\n", count)
	fmt.Printf("time taken: %v\n", time.Now().Sub(mainStart))
}

{% endhighlight %}

Now this gives is more options; I can set the concurrency level, I can set 
if I want pure max_out goroutines or a buffered channel that goes to a fan-in
worker pool, and I can set the size of the worker pool as well as the size
of the fan-in buffer, all in 71 LOC.

Here we go, lets do some test runs:

{% highlight bash %}
[husobee@localhost test-bcrypt]$ ./test-bcrypt -c 30 -m max_out # 30 goroutines crunching bcrypts
number of compares: 61
time taken: 5.320700462s
[husobee@localhost test-bcrypt]$ ./test-bcrypt -c 30 -m buffered -b 1 # 30 goroutines faning in to one worker on a buffer of 1
number of compares: 37
time taken: 5.000279935s
[husobee@localhost test-bcrypt]$ ./test-bcrypt -c 30 -m buffered -b 2 # 30 goroutines faning in to one worker on a buffer of 2
number of compares: 37
time taken: 5.000272393s
[husobee@localhost test-bcrypt]$ ./test-bcrypt -c 30 -m buffered -b 10 # 30 goroutines faning in to one worker on a buffer of 10
number of compares: 37
time taken: 5.000235865s

{% endhighlight %}

It is fairly obvious with one worker we get less work done than goroutines that
could go on either of my 2 cpus.  Lets up the workers next.

{% highlight bash %}
[husobee@localhost test-bcrypt]$ ./test-bcrypt -c 30 -m buffered -b 10 -w 2
number of compares: 69
time taken: 5.009044877s
[husobee@localhost test-bcrypt]$ ./test-bcrypt -c 30 -m buffered -b 10 -w 3
number of compares: 70
time taken: 5.017078581s
[husobee@localhost test-bcrypt]$ ./test-bcrypt -c 30 -m buffered -b 10 -w 4
number of compares: 69
time taken: 5.031552141s
{% endhighlight %}

Above you can see the fan in with a number of workers in the pool, and not too 
surprising we start loosing performance as we go above the number of cpus 
available.  Okay, this is great, lets see what happens when we crank up
concurrency on these two mechanisms (max_out goroutines versuses fan in worker
pool):

{% highlight bash %}
[husobee@localhost test-bcrypt]$ ./test-bcrypt -c 90 -m buffered -b 10 -w 3
number of compares: 69
time taken: 5.034115806s
[husobee@localhost test-bcrypt]$ ./test-bcrypt -c 90
number of compares: 27
time taken: 5.691676655s
{% endhighlight %}

Wow.  With just running bcrypt in 90 concurrent goroutines we quickly swamp
our machine and half the number of results we were getting with a concurrency
of 30.

In the mean time, our fan in buffered channel approach is able to sustain it's near
max of 69 bcrypts in 5 seconds with the exact same concurrency, and not slow
down it's processing.

I guess this behavior makes sense, as if you keep the cpu's worrying about one 
bcrypt operation at a time, and not causing them to flip their stacks all the
time you can get better results.

[homework]: https://gist.github.com/husobee/e7613552a3c22fdb680b
[redis-load]: http://husobee.github.io/golang/redis/2015/12/04/golang-redis-strange.html
