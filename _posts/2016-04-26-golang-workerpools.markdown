---
layout: post
title: "Worker Pool in Golang"
date: 2016-04-26 08:00:00
categories: golang workerpool
---

It is pretty often that I come across reasons to require a worker pool 
within my code.  It seems easy for me to come up with finite tasks that 
could be easily performed concurrently, and a work queue to feed those
task runners.

A worker pool in my opinion is a grouping of workers (threads, processes, 
coroutines), that are running the same general task definition, or code reading
from a queue of work tasks.

What I like to think of is a fast food restaurant when thinking about worker 
pools.  Imagine your code is a fast food restaurant.  You have a customer walk 
up to the counter and order a hamburger.  In a super simplified procedure 
without concurrency in place, there is only one worker in the restaurant.

This worker takes the order, runs back to the kitchen, makes the burger, brings
the burger back to the customer, and repeats for the next customer.  This is
such a terrible restaurant that you leave while waiting for hours in line to 
get your order in, and then you blog about it and the restaurant goes out of 
business.  Failure.

Well, you might be thinking, hey, lets just add more workers! Brilliant!  Take 
two:  We will have each customer serviced by their own worker on demand.

"n" workers take "n" customer orders all at once, so each customer order is
handled as they come in, and everyone is happy.  Well, in theory, this is
perfect, then you realize that your restaurant only has the ability to make 15 
burgers on the grill at a time, each burger taking 5 minutes to grill.

What happens in this situation is everyone is immediately served and then there 
is a huge race to the kitchen to take up resources.  Resources are suddenly 
completely consumed, and more importantly, the back of the restaurant each "worker"
who is doing all the tasks together start bumping into each other to the point
there is no way that finished hamburgers can even make it back to the counter, as
the crowd is so thick in the kitchen. Nice-ness breaks, and no work can get done.
The customers then leave, write blogs about how their order took a long time
from placed order to food delivered, and the restaurant collapses.  Failure.

Now, those two options sucked, what if we instead give people particular tasks
to work in, and feed those people with orders/completed orders?  Take three: 
there are "a" cooks, "b" order takers, and "c" order runners that run orders
back and forth from the counter to the kitchen.

What happens here is we can scale "b" order takers independently of the "a" cooks,
and "c" order runners.  We get a tour bus of burger eaters coming to our restaurant, 
we can then scale our order takers to reasonable accommodate.  Order takers hand 
the finished order request to the "c" order runners.  The "c" order runners then
take the order to the "a" cooks who are cooking 15 burgers at a time, paralleling 
blocking task (5 minutes) for each burger.

This situation has the best ability to scale, as resources are maximized, the throughput 
of burgers is maximized, and the amount of people running to and from the kitchen 
is minimized, allowing for consistent throughput.


## ok, explain in golang

### Try One:

{% highlight go %}

func runRestraunt() {
    order := takeOrder()
    completedOrder := makeOrder(order)
    deliverOrder()
}

func main(){
    for {
        runRestraunt()
    }
}

{% endhighlight %}

### Try Two:

{% highlight go %}

func runRestraunt() {
    order := takeOrder()
    completedOrder := makeOrder(order)
    deliverOrder()
}

func main(){
    for {
        // spawn each worker so they don't have to wait
        // to take orders
        go runRestraunt()
    }
}

{% endhighlight %}

### Try Three:

{% highlight go %}

func orderTakerWorker(numOrderTakers int) {
    // for the number of ordertakers needed, 
    for i:=0; i<numOrderTakers; i++ {
        // spawn a goroutine to handle taking orders, and 
        go func() {
            for {
                // take the order and put the order on the 
                // cook's in queue
                ordersIn <- takeOrder()

            }
        }()
    }
}

func cookWorker(numCooks int) {
    for i:=0; i<numCooks; i++ {
        go func() {
            for {
                // cook the order, and when done, put on the out channel
                ordersOut <- makeOrder(<-ordersIn)
            }
        }()
    }
}

func main(){
    // make our order runners!
    ordersIn := make(chan order)
    ordersOut := make(chan completedOrder)

    // spawn our workers!
    spawnOrderTakerWorkers(runtime.NumCPU*2)
    spawnCookWorkers(runtime.NumCPU*2)

    for {
        // deliver finished orders
        deliverOrders(<-ordersOut)
    }
}

{% endhighlight %}


Obviously, the above are naive implementations, but you get the point.  You never want
too many cooks in the kitchen!

