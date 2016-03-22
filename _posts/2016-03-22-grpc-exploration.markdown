---
layout: post
title: "gRPC exploration in golang"
date: 2016-03-22 08:00:00
categories: golang gRPC
---

## background

As a web api developer, when I hear "RPC" my first inclination is to run and
hide behind my good friends REST and HTTP.  But recently I came across a task
where I couldn't do that.  We all like to stay in our comfort zone for work 
tasks and tend not to stray too far from what we understand, and is easy for
us.  

Well I found a project recently at work where "normal" web development wouldn't
work.  Transferring tens of millions of customer accounts from our legacy
data storage to our new micro-service architecture written in golang.  Just some
napkin math showed that "just writing an API" wouldn't cut it.

## introducing protobuf

Doing some thought experimentation with a peer and manager it came pretty clear
that we needed a better serialization mechanism than JSON, or XML.  Then 
[Protocol Buffers][protobuf] came up as an option.

Protocol Buffers, as mentioned on the site, is a language-neutral, 
platform-neutral extensible mechanism for serializing structured data.  The
Protocol Buffers project is a transpiler of sorts.  It has a structured language
which then compiles the transport serialization code into your chosen language
to be included in your project.  For example you can see below where I am making
a simple RPC definition with a simple input and output message:

{% highlight text %}
    syntax = "proto3";
    package migrate;

    service Migration {
        rpc PutAccount (AccountRequest) returns (AccountReply) {}
    }

    message AccountRequest {
        string request_id = 1;
        string account_id = 2;
        //...
    }
    message AccountResponse {
        string request_id = 1;
        bool success = 2;
        //...
    }
{% endhighlight %}

Simply put, the above `Migration` service has a `PutAccount` RPC method which 
takes in an `AccountRequest` and outputs an `AccountReply`.  How easy was that?

After downloading the version 3 proto buffers compiler you can run:

    protoc --proto_path=./ --go_out=./ ./foo.proto

Where foo.proto is the proto buffer definition from above.  You will then find a
file called foo.pb.go which is the "compiled" proto buffer go code.  Pretty
cool that you can make a service definition in a meta language and have it
fully implemented.

## introducing gRPC


gRPC is google's HTTP2 wrapped RPC protocol.  RPC is Remote Procedure Call.  What
this allows is for a client running on a computer to access a remote computer, 
via computer network, and call a "function" on that remote computer as if the
function was local to the client.  You can already see the benefits I would imagine
for creating a migration client/server application using RPC.

gRPC looked interesting as a wrapper for protocol buffers as we would get a lot
of things for free.  These things include:

* Ability to reuse tcp connections
* ability to have authentication per transport and per call
* ability to use web technologies
* ability to take advantage of http2 in golang

Below You can see an image, taken from the [grpc.io][grpc] site which explains 
what gRPC conceptually looks like.  It mainly consists of a server and clients
which call functions with parameters and recieve responses.

{% include image.html url="http://www.grpc.io/img/grpc_concept_diagram_00.png" description="gRPC concept diagram, from grpc.io" %}

gRPC has many benefits over straight RPC (which is also easy with protocol
buffers in golang) in that you inherit nice things like TLS for free.  You 
also inherit the ability to use client certificates for transport authentication.
Moreover if you are really interested in authentication and authorization you
can even use oauth2.0 to auth your RPC calls on a per call level.

Below is a quick example of our service defined above in a gRPC server:

{% highlight go %}
package main

// imports and stuff ...

// this is assuming that your foo.pb.go is in the main package

type accountMigrationServer struct {}

// PutAccount - this is our "implemention" of our RPC, which takes a context as
// well as our AccountRequest as input and returns our AccountReply as output
func (s *accountMigrationServer) PutAccount(ctx context.Context, in *AccountRequest) (*AccountReply, error) {
    // do stuff
    return &AccountReply{
        RequestId: in.RequestId,
    }
}

func main() {
    lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
    if err != nil {
                log.Fatalf("failed to listen: %v", err)
    }
    // create our grpc server below
    grpcServer := grpc.NewServer()
    // register our Migration Service
    pb.RegisterMigrationServer(grpcServer, &accountMigrationServer{})
    grpcServer.Serve(lis)
}
{% endhighlight %}

## conclusion

This seems like the way to go for migration type tasks, and I am excited to see 
how it goes.  It turns out it is pretty nice going outside your comfort zone on
occasion.

[grpc]: http://www.grpc.io/
[protobuf]: https://developers.google.com/protocol-buffers/

