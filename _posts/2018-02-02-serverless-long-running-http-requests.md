---
layout: post
title: Pattern, Serverless Long Running HTTP Requests in AWS
categories: [API_Gateway, AWS, HTTP, AWS_Lambda, Serverless]
---

![http://www.99serverless.com/wp-content/uploads/2017/11/Serverless-Application-Architecture.png](http://www.99serverless.com/wp-content/uploads/2017/11/Serverless-Application-Architecture.png)


The serverless way to deploy web applications in AWS usually involves hooking up API Gateway with Lambda. If you done this, chances are you have ran into a built-in constraint: `the timeout`. These products are designed to run microservices, which are expected to be micro, duration-wise as well 😆. However, there are some use cases for which our web app needs to handle long runnings tasks. In this article, I will share with you an HTTP pattern to accomplish that while still being serverless.

<!--more-->


## Intro

First things first. Lambda functions will timeout after a max. of `5 min`; API Gateway requests will timeout after `29 sec`. Find out more about service limits: [Lambda](https://docs.aws.amazon.com/lambda/latest/dg/limits.html) and [API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/limits.html).

There are plenty of use cases I can see that would need more than 29 s. But that’s not important right now. We have first to understand our pattern’s goal:

> We want to respond to the request quickly and perform the long running task on the background.

Simple, right ? This is also called **Asynchronous processing**.

Let’s it break it down into three parts: i) initial request/response; ii) background processing; iii) polling.


## Initial Request/Response


The goal of this step is to receive the request, start executing the long running job via non-block (asynchronous) call, and finally create a persistent data object to hold the information about the running job. After all that, we will make use of a nice HTTP status code to produce our response:


> [Section 6.3.3 of RFC 7231](https://tools.ietf.org/html/rfc7231)
> > The 202 (Accepted) status code indicates that the request has been accepted for processing, but the processing has not been completed.

That’s precisely what we are doing here. The last piece of information we need to include is the URI Locationof our job. We do this by adding the Location header to the HTTP response, which will look like:

```
HTTP 202
Location: jobs/1234
```

All of this can be easily executed in under 29s. 😁


## Background Processing
You might be wondering where we will execute the long running job. We want to be serverless, so let’s use another Lambda function. As mentioned in the last step, this function will be called asynchronously, detached from the original Lambda. That means, we can now take advantage of the full 5 min Lambda timeout. That time should be enough for us to execute our long running task.

Once processing is completed, the function must update the status of `Job` in the database. Something like `my_job.status = 'DONE'`.



# Polling
So how will the HTTP client know when the job is completed ? Simple, the client will poll for the job status from time to time. It will send a GET request to the location returned in the first step:

```
GET jobs/1234
{"status": "RUNNING"}
```

Clients should send requests with care so not to overload the server with too many requests. A good strategy is to perform polls with [Exponential Backoff](https://en.wikipedia.org/wiki/Exponential_backoff).

An alternative to polling is to setup the create job call with an HTTP hook, so that the client can get notified when the processing completes.


## Architecture
If we were going to use, say, DynamoDB for our data layer, here’s how our full serverless architecture would look like:

![http://www.99serverless.com/wp-content/uploads/2017/11/Serverless-Application-Architecture.png](http://www.99serverless.com/wp-content/uploads/2017/11/Serverless-Application-Architecture.png)


Thanks to [Cloudcraft](https://cloudcraft.co/) for this nice diagram 🙂


## That’s It
Post on the comments section below if  have any questions.

Liked it ? Share this with your friends.
