---
layout: post
title: "API Input Validation in Golang"
date: 2016-01-08 08:00:00
categories: golang validation
---

## Problem Definition

It is important to always validate inputs.  If you don't validate inputs you 
are leaving yourself open to future headaches, annoyances and bugs.

Input validation to me should be done as generically as possible and yet every
single input type should be flexible enough to validate independently.  


## Solution
I have been using this mechanism for input validation for a while, and it works very
well for me:

{% highlight go %}

// InputValidation - an interface for all "input submission" structs used for
// deserialization.  We pass in the request so that we can potentially get the
// context by the request from our context manager (future net.http will include
// a context in the request)
type InputValidation interface {
	Validate(r *http.Request) error
}

var (
	// ErrInvalidUUID - error when we have a UUID validation issue
	ErrInvalidUUID = errors.New("invalid uuid")
	// ErrInvalidName - error when we have an invalid name
	ErrInvalidName = errors.New("invalid name")
)

// Thing - is our implementation of InputValidation and the structure we will
// deserialize into
type Thing struct {
	ID       string `json:"id"`
	Name     string `json:"name"`
	Category string `json:"category"`
}

// Validate - implementation of the InputValidation interface
func (t Thing) Validate(r *http.Request) error {
	// validate the ID is a uuid
	if !govalidator.IsUUID(t.ID) {
		return ErrInvalidUUID
	}
	// validate the name is not empty or missing
	if govalidator.IsNull(t.Name) {
		return ErrInvalidName
	}
	return nil
}

{% endhighlight %}

As you can see from the above, we have one small, tight interface `InputValidation`
which contains a single `Validate` method.  This is very powerful, because we only
need to implement validation in one place, and yet we can also have the flexibility
of keeping the validation logic right next to the the definition of the structure 
it is validating.

In our example above, `Thing.Validate` validates that the ID field is a valid UUID
and also validates that the Name field is not an empty or missing attribute.

What this looks like in practice is as follows:

{% highlight go %}

package main

import (
	"encoding/json"
	"errors"
	"net/http"

	"github.com/asaskevich/govalidator"
)

// main - a simple main entry point
func main() {
	srv := &http.Server{
		Addr:    ":1234",
		Handler: f,
	}
	srv.ListenAndServe()
}

// f - our only application handler to demonstrate validation
var f http.HandlerFunc = func(w http.ResponseWriter, r *http.Request) {
	// we have an input structure, of type Thing
	var thing *Thing
	// we want to decode and validate Thing from request body
	err := decodeAndValidate(r, thing)
	// there was an error with Thing
	if err != nil {
		// send a bad request back to the caller
		w.WriteHeader(http.StatusBadRequest)
		w.Write([]byte("bad request"))
		return
	}
	// it was decoded, and validated properly, success
	w.WriteHeader(http.StatusOK)
	w.Write([]byte("ok"))
}

// decodeAndValidate - entrypoint for deserialization and validation
// of the submission input
func decodeAndValidate(r *http.Request, v InputValidation) error {
	// json decode the payload - obviously this could be abstracted
	// to handle many content types
	if err := json.NewDecoder(r.Body).Decode(v); err != nil {
		return err
	}
	defer r.Body.Close()
	// peform validation on the InputValidation implementation
	return v.Validate(r)
}
{% endhighlight %}


Inside our handler we instantiate the type of Thing we want to decode from the 
request body and validate.  We pass the request and the Thing to `decodeAndValidate`
where we perform a json decode.  We then take the resultant struct and call it's
own validate method, returning any validation errors.

## Why is this cool?

I really like this mechanism because it is trivial to mock for unit tests
as well as a super clean and sexy solution to the problem.  Not to mention
it also makes sense that the logic for validation should be a function pointer 
off the structure itself.

Another reason why this is cool is that you will not have to write extra amounts
of copious boiler plate to take advantage of this solution.  You will already be
making your struct you will be deserializing into, and already be writing validation
logic.

This solution also scales for reuse if you need to reuse these input structures 
in other places.  Not to mention you have a consistent means for deserialization
and validation of these structures.  [Here][gist] is a link to the full gist.


[gist]: https://gist.github.com/husobee/8c5284786a741e2a1909
