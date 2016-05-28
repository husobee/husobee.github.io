---
layout: post
title: "REST v. gRPC"
date: 2016-05-28 08:00:00
categories: golang rest grpc
---

As some might realize [I have been getting into gRPC][grpc-blog] as of late for
internal API development at the company I work at.  In the blog aforementioned
I talked about how easy it was to get started with that in golang.

Yesterday, we are concluding a big project where we need to migrate much data at
work, and we decided to try gRPC on it because we had this feeling that it was 
going to be faster than REST calls to our existing API.  When we retrospected 
about the project yesterday it was mentioned that we could have finished the
project faster if maybe we did the project in a technology that more people on
the team knew, basically if we didn't do gRPC more people could swarm on that project.

Here I am going to do some benchmarks against a classic REST API and the exact
same endpoint in gRPC so we can see the performance differences.

# Results for the impatient

I [wrote some code][grpcvrest] to show the amazing difference.  Here are the results:

{% highlight text %}
BenchmarkGRPCSetInfo-8      5000            224633 ns/op
BenchmarkRESTSetInfo-8       200           5748596 ns/op
{% endhighlight %}

Wow *25 Times* performance gain!!

# Setup for the interested

In order to do an apples to apples comparison, I created a REST HTTPS endpoint that 
took in json, performed a validation check on the input, and then responded.  
Along the same lines, I also created a ProtoBuffer service description, converted
that to golang code and copy and pasted the same validation mechanism.

Here is the REST http.HandlerFunc:

{% highlight go %}

// SetInfo - Rest HTTP Handler
func SetInfo(w http.ResponseWriter, r *http.Request) {
	var (
		input    apiInput
		response apiResponse
	)

	// decode input
	decoder := json.NewDecoder(r.Body)
	decoder.Decode(&input)
	r.Body.Close()

	// validate input
	if err := validate(input); err != nil {
		response.Success = false
		response.Reason = err.Error()
		respBytes, _ := json.Marshal(response)

		w.WriteHeader(400)
		w.Write(respBytes)
		return
	}
	response.Success = true
	respBytes, _ := json.Marshal(response)

	w.WriteHeader(200)
	w.Write(respBytes)
}

{% endhighlight %}

Pretty reasonable handler.  Here is the competing GRPC handler:

{% highlight go %}

// SetInfo - implements our InfoServer
func (s *server) SetInfo(ctx context.Context, in *InfoRequest) (*InfoReply, error) {
	if err := validate(in); err != nil {
		return &InfoReply{
			Success: false,
			Reason:  err.Error(),
		}, err
	}
	return &InfoReply{
		Success: true,
	}, nil
}

{% endhighlight %}

Wow, that GRPC endpoint is so much cleaner in my opinion.  There is no messy
input/response serialization baloney as grpc deals with the serialization.  Moreover,
there is actually a service description that generates all the InfoRequest/InfoReply
structures for me.  It is pretty clear that a scheme change would cause many many
touch points in the REST version of the code, whereas there are basically no refactoring
touch points other than the service definition file. 

It is fairly clear to me based on the above that for internal only APIs I much prefer
to use gRPC over REST.

[grpc-blog]: http://husobee.github.io/golang/grpc/2016/03/22/grpc-exploration.html
[grpcvrest]: http://github.com/husobee/grpc_v_rest
