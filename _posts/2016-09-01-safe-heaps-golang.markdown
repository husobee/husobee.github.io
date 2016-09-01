---
layout: post
title: "Safe Heaps in Golang"
date: 2016-09-01 08:00:00
categories: heaps golang safe
---


## Heaps and Heaps of fun

I had the need for a priority queue today in golang so I implemented 
`container/heap.Interface` which has the following methods:

{% highlight go %}
type Interface interface {
	sort.Interface
	Push(x interface{}) // add x as element Len()
	Pop() interface{}   // remove and return element Len() - 1.
}
{% endhighlight %}

Pretty simple interface, Push, Pop and a sorting interface, which contains Len,
Swap, and Less. Which ended up looking a lot like the priority queue example in 
the godocs:

{% highlight go %}

// Item - the item that holds the structure needed for the queue
type Item struct {
	value interface{}
	priority int
	index int
}

type PriorityQueue []*Item

// Len - get the length of the heap
func (pq PriorityQueue) Len() int { return len(pq) }

// Less - determine which is more priority than another
func (pq PriorityQueue) Less(i, j int) bool {
	return pq[i].priority < pq[j].priority
}

// Swap - implementation of swap for the heap interface
func (pq PriorityQueue) Swap(i, j int) {
	pq[i], pq[j] = pq[j], pq[i]
	pq[i].index = i
	pq[j].index = j
}

// Push - implemenetation of push for the heap interface
func (pq *PriorityQueue) Push(x interface{}) {
	n := len(*pq)
	item := x.(*Item)
	item.index = n
	*pq = append(*pq, item)
}

// Pop - implementation of pop for heap interface
func (pq *PriorityQueue) Pop() interface{} {
	old := *pq
	n := len(old)
	item := old[n-1]
	item.index = -1 // for safety
	*pq = old[0 : n-1]
	return item
}

{% endhighlight %}

At this point we have a working priority queue. We use `heap.Push` and `heap.Pop`
to interface with the heap, and things are lovely. Until you try to use this in 
a web application, that potentially has many goroutines trying to push and pop 
at the same time.

Much like Golang's map implementation the `container.heap` implementation is not
threadsafe, and is not threadsafe for a reason, the application knows better if 
this implemenatation needs to be safe or not.  

With this in mind, we need to protect this datastructure.

## The Wrong Way

The wrong way to try to protect this datastructure is to try to protect it within the 
implementation of the `heap.Interface` interface.  You might be tempted to use 
channels for serialization of writes to the underlying datastructure.  What
could go wrong, you can protect the physical underlying datastructure by having
a single goroutine doing all of the writes to the datastructure, and being messaged from
many goroutines.

{% highlight go %}

// Push - implemenetation of push for the heap interface
func (pq *PriorityQueue) Push(x interface{}) {
	pq.PushChannel <- x
}

func (pq \*PriorityQueue) doWrites() chan bool{
	var quit = make(chan bool)
	go func () {
		for {
			select {
			case quit:
				return
			case x <- pq.PushChannel:
				n := len(*pq)
				item := x.(*Item)
				item.index = n - 1
				*pq = append(*pq, item)
			// ...
			}
		}
	}
	return quit
}

{% endhighlight %}

Well, not really.  You can not protect it in the implementation of the interface
mainly because the heap sorting is performed in the `container.heap` package, and
you can not be sure that each operation happens in the right order due to the
channel selection randomness.


## The Right Way?

In the end I have decided to serialize the access to the heap.Push and heap.Pop
function calls in my application by doing what can be seen below:

{% highlight go %}

// heapPopChanMsg - the message structure for a pop chan
type heapPopChanMsg struct {
	h      heap.Interface
	result chan interface{}
}

// heapPushChanMsg - the message structure for a push chan
type heapPushChanMsg struct {
	h heap.Interface
	x interface{}
}

var (
	quitChan chan bool
	// heapPushChan - push channel for pushing to a heap
	heapPushChan = make(chan heapPushChanMsg)
	// heapPopChan - pop channel for popping from a heap
	heapPopChan = make(chan heapPopChanMsg)
)

// HeapPush - safely push item to a heap interface
func HeapPush(h heap.Interface, x interface{}) {
	heapPushChan <- heapPushChanMsg{
		h: h,
		x: x,
	}
}

// HeapPop - safely pop item from a heap interface
func HeapPop(h heap.Interface) interface{} {
	var result = make(chan interface{})
	heapPopChan <- heapPopChanMsg{
		h:      h,
		result: result,
	}
	return <-result
}

//stopWatchHeapOps - stop watching for heap operations
func stopWatchHeapOps() {
	quitChan <- true
}

// watchHeapOps - watch for push/pops to our heap, and serializing the operations
// with channels
func watchHeapOps() chan bool {
	var quit = make(chan bool)
	go func() {
		for {
			select {
			case <-quit:
				// TODO: update to quit gracefully
				// TODO: maybe need to dump state somewhere?
				return
			case popMsg := <-heapPopChan:
				popMsg.result <- heap.Pop(popMsg.h)
			case pushMsg := <-heapPushChan:
				heap.Push(pushMsg.h, pushMsg.x)
			}
		}
	}()
	return quit
}

{% endhighlight %}

This code now serializes the Push/Pops to the heap and should be threadsafe.

