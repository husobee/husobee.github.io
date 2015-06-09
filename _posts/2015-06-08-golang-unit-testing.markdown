---
layout: post
title: "Go Mock Yourself"
date: 2015-06-08 20:00:00
categories: golang testing unit-test
---

Unit testing is important.  That being said, there are a few things about 
golang that make testing difficult.  I am going to try to explain a few 
mechanisms I use to deal with common unit testing problems in golang:

1.) Monkey Patching Functions For Unit Testing
2.) Mocking Struct Methods via Interfacing

*tl;dr: [unit testing in golang][unit-test-gist]*

My coworker [agtorre][agtorre-blog] and I implement our mocking with these mechanisms for our unit testing.

## Monkey Patching in Golang

So, from what I understand, monkey patching is the art of modifying a module,
or package at runtime to alter the course of execution.  To those ends, golang 
fits this model pretty well, as functions are first class constructs.  When we 
want to make a third party package function "mocked" we can merely do the following:

{% highlight go %}
package main

import "database/sql"

var sqlOpen = sql.Open

func main(){
    db, err := sql.Open("sqlite3", "~/my.db")
    if err != nil {
        panic()
    }
    db.close()
}
{% endhighlight %}

Since package functions are assignable to variables, we can mock like this:

{% highlight go %}
package main
import (
    "errors"
    "testing"
)

func TestMain(t *testing.T) {
    // set oldSqlOpen to old sqlOpen 
    oldSqlOpen := sqlOpen
    // as we are exiting, revert sqlOpen back to oldSqlOpen at end of function
    defer func () { sqlOpen = oldSqlOpen }()
    sqlOpen = func (driver, conn string) (*sql.DB, error) {
        return nil, errors.New("failed to connect to db")
    }
    main()
    // assertion for main panicing
}
{% endhighlight %}

The following things are happening here:

* When this test is run, we are storing the old location of the mocked function.  
* Then we are detering the reset of the old location of the original function back.  
* This defer will happen after the function finishes, before the stack is popped. 
* Then we are overwriting our sqlOpen package function with a new definition that will throw an error
* Then we run main()

At this point main is run with our fake sqlOpen definition and we should see a panic.  This 
is an ugly, but functional way of testing different potential outcomes of package functions.

## Mocking Struct Methods via Interfacing

Go allows for interfaces, which is excellent for making things like other things.  We are going to 
use interface implementations of third party structs to mock out a virtual struct that will allow us to 
perform mocking of a struct's methods:

{% highlight go %}
package main

type Something struct {
    counter int
}

func (s *Something) Increment() error {
    counter++
    return nil
}
// Something Interface is something for something to implement
type SomethingInterface interface {
    Increment() error
}

func actionOnSomething(s SomethingInterface) {
    if s.Increment(); err != nil {
        panic()
    }
}
{% endhighlight %}

We are tasked with testing the edge case of when Increment returns an error, and we need to test if a panic 
occurs.  To do this we need to mock out Something.Increment and return a non-nil error result.  Here is how 
I usually go about doing this:

{% highlight go %}
package main

import (
    "errors"
    "testing"
)
// we are making a mock Something struct with an attribute of mockIncrement which
// has the same function signature as real Something.Increment
// Something Interface is implemented just like the real something
type MockSomething struct {
    mockIncrement func() error
}

//Increment will check to see if we have a defined mockIncrement,
// otherwise return success happy path
func (s *MockSomething) Increment() error {
    if s.mockIncrement != nil {
        return s.mockIncrement()
    }
    return nil
}


func TestActionOnSomething (t *testing.T) {
    // create a MockSomething, we will use as a drop in replacement
    // of Something in our function call we are trying to test
    ms := &MockSomething{
        mockIncrement: func() error {
            // this time we want to mock our a failure scenario
            return errors.New("failed to increment")
        },
    }
    // perform the function call we want to test
    actionOnSomething(ms)
    // ... check for panic
}
{% endhighlight %}

Hope this was helpful to anyone.

[unit-test-gist]: https://gist.github.com/husobee/9ff87a6f27e9abb4a3bc
[agtorre-blog]: http://www.aaron-torres.com
