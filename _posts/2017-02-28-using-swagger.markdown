---
layout: post
title: "Using Swagger to Develop APIs"
date: 2017-02-28 08:00:00
categories: api rest swagger
---

Recently, I started taking a real look into using [swagger][swagger] to document
REST APIs.  I feel like REST APIs can be very hard to convey from a producer to 
a consumer, and there is often a bit of confusion as to how the response will be
returned, or what is required in a request to an endpoint.  This article 
attempts to explain how swagger works from a schema standpoint, what one can do
with swagger to improve a REST API.

## What is Swagger

Swagger is an Open API description standard, which allows for clear
communication about what is needed to interact with an API.  The [swagger 
specification][swaggerspec] allows for one to completely describe a REST API
in a way that others can consume in an automated fashion, and which will allow
for ultimately code generation of an API client.

Below is a contrived example of an API described with swagger:

{% highlight yaml %}
swagger: '2.0'
info:
  title: Music Store API
  description: >
    This is a Music Store API which is made up just to
    explain a little bit about how swagger works, and
    what you are able to do with it.
  version: 0.0.1

basePath: /v1
schemes:
  - https
consumes:
  - application/json
produces:
  - application/json

paths:
  /purchase:
    post:
      parameters:
        -
          name: purchaseOrder
          in: body
          required: true
          schema:
            $ref: "#/definitions/PurchaseOrderRequest"
      responses:
        200:
          description: >
            Successful Purchase Response, this is the OK
            response
          schema:
            $ref: "#/definitions/PurchaseSuccessResponse"

definitions:
  PurchaseSuccessResponse:
    type: object
    required:
      - success
      - message
    properties:
      success:
        type: boolean
      message:
        type: string
      confirmation:
        type: string
        format: uuid
      sent_to:
        type: string
        format: email
      cost:
        type: integer
        format: int64
        minimum: 0

  PurchaseOrderRequest:
    type: object
    required:
      - sku
      - payment_id
    properties:
      playlist:
        type: string
        enum:
          - "workout"
          - "home"
          - "car"
      sku:
        type: string
        pattern: /\d{4}-\d{10}/
        minLength: 15
        maxLength: 15
      payment_id:
        type: string
        format: uuid

{% endhighlight %}

What is all of this doing?  Well, it is describing what this one endpoint API is
requiring, and committing to.  We can see this API has a base path of "/v1" so
all routes enumerated will begin with "/v1".  This api clearly also produces and
consumes json, and is only accessible via https.

Then we get to the paths.  This API has one path /v1/purchase which is an HTTP
POST operation, which takes in an object referenced by PurchaseOrderRequest.  This
API has one possible response, a 200 HTTP status code response, which returns an
object referenced by PurchaseSuccessResponse.

Getting into the object structures, in the definitions section, you can see the 
PurchaseOrderRequest object is an object with two required fields, sku and payment_id.
You can also see it has another field which is a string called playlist which can only
be of value "workout" "home" or "car".

Swagger is very flexible in how you describe an API.


## Great, now what?

Well, swagger is more than just a documentation tool.  There is a lot of tooling
around the OpenAPI format which allows you to do some neat things.

[Swagger Editor][swaggereditor] is a tool which will help you visualize an API
which has been developed in swagger.  This tool will also allow you to create
api requests to an actual instance if you were so inclined to see what the API
will do based on certain inputs.

[Swagger Codegen][swagger-codegen] is a tool which will actually dump out working
code in a number of different languages which are able to integrate with the API
you generated the code from.  This is extremely handy if you want to offer your
customers an SDK of your APIs, or if you want to perform some automated testing
of your API based on the swagger definitions.

Armed with these tools and this specification, you will be able to more clearly
communicate with your integrators, and have ready made clients for internal
integrations.


[swagger]: http://swagger.io/
[swaggerspec]: http://swagger.io/specification/
[swaggereditor]: http://editor.swagger.io/#!/
[swagger-codegen]: https://github.com/swagger-api/swagger-codegen
