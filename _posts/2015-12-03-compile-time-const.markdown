---
layout: post
title: "golang compile time variables"
date: 2015-12-03 08:00:00
categories: golang compile time variables
---

## Compile Time Variables

A very handy build flag allows you to set symbol values from the command line 
go build tool.  Check out the example below:

{% highlight go %}
package main

import "fmt"

var (
    version string
)

func main() {
    fmt.Println("version: ", version)
}

{% endhighlight %}

This example has an empty string called version.  when we compile and run this 
program we get the following output: 

{% highlight bash %}

go build && ./main
version:  


go tool nm main | grep vers
  5942d0 D main.version

{% endhighlight %}

Great, we were able to print an empty version variable! The tool `nm` shows you
the symbol table of your executable.  That is pretty neat, we can see our 
`main.version` variable defined.  

## What if

What if we could make our CI build system embed the version and other meta data 
about the build of this executable into the application, so the application is 
aware of the version??  

With the below command, we are able to populate that variable statically inside 
the executable at compile time:

{% highlight bash %}
[husobee@localhost ugh]$ go build -ldflags "-X main.version=1.0.1" main.go
[husobee@localhost ugh]$ ./main
version:  1.0.1
{% endhighlight %}

Go's Linker allows you to set string symbols with the -X flag, which is exposed 
through go build's -ldflags.  

## In real life

In real life at my company we use this technique to embed commit hash, semver 
versions and other meta data into our executables at build time.  This has been
handy as we have a route we use in our web application we can GET from that will
display this type of meta data information.
