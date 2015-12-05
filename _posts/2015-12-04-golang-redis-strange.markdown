---
layout: post
title: "golang redis strange"
date: 2015-12-04 08:00:00
categories: golang redis
---

## Testing With Load

Testing with behaviors is great, as I have mentioned [in prior posts][behavior],
and unit testing is great for making sure your code does what you think it does.
With those two forms of testing we cover all our bases for web api development
testing, right?

No.  Web api development must be tested at scale, under load, because under those
conditions strange issues start cropping up.  Where I work we are experimenting
with different load testing frameworks, and I believe we have settled on
[locust][locust] for our load testing needs.

Locust is neat, because with minimal python (example below stolen from 
[locust][locust]) you can have a fully functional load test on your application:

{% highlight python %}
from locust import HttpLocust, TaskSet, task

class WebsiteTasks(TaskSet):
    def on_start(self):
        self.client.post("/login", {
            "username": "test_user",
            "password": ""
        })
    
    @task
    def index(self):
        self.client.get("/")
        
    @task
    def about(self):
        self.client.get("/about/")

class WebsiteUser(HttpLocust):
    task_set = WebsiteTasks
    min_wait = 5000
    max_wait = 15000

{% endhighlight %}

Back to what I wanted to talk about in this post.  We were performing load tests
yesterday, when all of a sudden we saw a HUGE number of new connections being made
to our redis elasticache instance.  This was very strange as we were using a well
known redis library [redigo][redigo], moreover we were using the `redigo.RedisPool`
which to my knowledge was _supposed to_ perform connection pooling, thereby only
providing a certain number of connections to a redis server.

## Investigation

We were using [redigo][redigo] by consequence of using [negroni-sessions][sessions]
which uses [boj/redisstore][redisstore].  So right off the bat I can already tell
we are too many layers deep in this redis abstraction. Here is, at a high level,
the flow:

1. Request Comes In
2. Negroni ServeHTTP is called
3. Negroni Calls [negroni sessions][sessions]
4. Negroni Sessions provides a wrapper around [Gorilla Sessions][gsessions]
5. Gorilla Sessions uses a plugable Storage Backend
6. Gorilla Sessions calls [boj/redistore][redisstore]
7. Redistore wraps [garyburd/redigo][redigo] for persisting the session

As seen, this is a complicated flow for merely setting a cookie, and persisting
said cookie value in a Key/Value Store.  Anyways, back to the beginning, it 
appeared that there was no connection pooling to redis.  Here is a sample of how
we were initiating negroni-sessions:

{% highlight go %}
	store, _ := redistore.New(3, "tcp", "server:6379", "secretpass", nil)
	n.Use(sessions.Sessions("my_session", store))
{% endhighlight %}

Looking into [negroni-sessions][sessions-redistore] you can see that this 
basically creates a new boj/redistore.Redistore structure. Looking in
[boj/redistore][redistore-def] which contains a [garyburd/redis.Pool][redis-pool]
instance.

A few thoughts on everything so far:

* There is a lot of abstraction for the sake of abstraction in this flow
  * Abstraction is fine if reasonable, this is becoming unreasonable
  * [A little copying is better than a little dependency][proverbs]
* There are a lot of hidden assumptions that get lost along the way,
  * Looking at RedisPool in [redigo][redis-pool], Pool does a lot of stuff
    and has a lot of options, that are hidden in our app.
  * We don't have fine grained control over many options
  * [Make the zero value useful][proverbs]
* Interfaces describe behaviors, Structs describe state
  * redigo would be much more powerful (and mockable) if Pool was an Interface
    instead of a struct.

This is all great, but we are obviously creating a redis.Pool, and yet we are 
seeing just about one new connection for every api call we make to redis.  Looking closer 
at the [pooling implementation][pool-get-impl] specifically [this block][culprit]:

{% highlight go %}

	// Dial new connection if under limit.

	if p.MaxActive == 0 || p.active < p.MaxActive {
		dial := p.Dial
		p.active += 1
		p.mu.Unlock()
		c, err := dial()
		if err != nil {
			p.mu.Lock()
			p.release()
			p.mu.Unlock()
			c = nil
		}
		return c, err
	}

{% endhighlight %}

It is fairly obvious at this point, no where in my sessions journey was anyone
setting a realistic default value for Pool.MaxActive, and since it is an int in
go, we default to 0.  The above implementation for getting connections is saying
if there is no MaxActive, just go ahead and create a new connection if all other
idle connections are busy.... Since we are running a pair of webservers under 
load it was fairly easy to overrun our MaxIdle connections and require new 
connections to be setup, and reaped basically as we go.  I believe there is some
tuning we could do with MaxIdle to not have to reap basically every connection 
we make also.

This really flies in the face of Rob Pike's [go-proverbs][proverbs]
specifically "Making the zero value useful."  In my opinion standard operations 
of a connection "pool" should default to a sane MaxActive connection limit, as
opposed to defaulting to infinity.  This doesn't even resemble a pool under load
at this point because blocking connections are immediately reaped after the 
connection within the pool is "closed" releasing that connection back to the pool..

Moreover, in reviewing redigo, there is a lot of synchronization, with mutexes
that would be better suited using goroutines and channels. Specifically going 
against "Channels orchestrate; mutexes serialize." and "Don't communicate by sharing memory, share memory by communicating."


[proverbs]: http://go-proverbs.github.io/
[locust]: http://locust.io/
[behavior]: http://husobee.github.io/automate/behavior/testing/2015/06/13/automate-behaviors.html
[redigo]: https://github.com/garyburd/redigo
[redisstore]: https://github.com/boj/redistore
[sessions]: https://github.com/goincremental/negroni-sessions
[gsessions]: http://www.gorillatoolkit.org/pkg/sessions
[sessions-redistore]: https://github.com/GoIncremental/negroni-sessions/blob/8843c2f94d7853cf0b4c8799e23f9bca0bbb5690/redisstore/main.go
[redistore-def]: https://github.com/boj/redistore/blob/9c0e6bab4dd444285424f189b8fb0cb03f653242/redistore.go#L85-L93
[redis-pool]: https://github.com/garyburd/redigo/blob/3d0709611e0e29c05a4e925e8447b065abb4eef6/redis/pool.go#L43-L134
[pool-get-impl]: https://github.com/garyburd/redigo/blob/3d0709611e0e29c05a4e925e8447b065abb4eef6/redis/pool.go#L195-L273
[culprit]: https://github.com/garyburd/redigo/blob/3d0709611e0e29c05a4e925e8447b065abb4eef6/redis/pool.go#L250
