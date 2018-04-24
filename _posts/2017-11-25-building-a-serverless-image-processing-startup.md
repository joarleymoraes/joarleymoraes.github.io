---
layout: post
title: Building A Serverless Image Processing SaaS
categories: [API_Gateway, AWS, Flask, AWS_Lambda, Python, SaaS, Serverless, Zappa]
---

![http://www.99serverless.com/wp-content/uploads/2017/11/architecture.png](http://www.99serverless.com/wp-content/uploads/2017/11/architecture.png)


If you google “Image Processing SaaS” you will find many image processing services. Some of them you might have already used, or at least heard of, such as Imgix, Bitline, Cloudinary, etc. [Here’s a more comprehensive list](https://gist.github.com/cheeaun/6385645). In this tutorial, I will share with you how you can build your own and serverless service, using AWS and [Zappa](https://github.com/Miserlou/Zappa). So I’m happy to announce **imgy**, a tiny image processing service we will be building in this tutorial. 😀

<!--more-->


## TL;DR

![https://raw.githubusercontent.com/joarleymoraes/imgy/master/docs/demo.gif](https://raw.githubusercontent.com/joarleymoraes/imgy/master/docs/demo.gif)



## Our Scope

Most features of all these services orbit around a implementation of a real-time image transformation API, a simple RESTful web service with a single operation that looks like the following:

`GET https://example.com/api/image.png?w=300&h=300`


For example, in the above HTTP request, the API receives the input `image.png`, dynamically makes image modifications (in this case, changes its dimensions to 300px) and returns the modified image, as HTTP (binary) response.

Very simple, right ?

Of course, we can have many image processing modifiers, such as scaling, format conversion, quality compression, etc, but we will get to that later. For now what you need to understand is that the API is a single method/resource.


## Prerequisites

This tutorial assumes you know at least the basics of some AWS products, mainly CloudFormation, CloudFront, S3, APIGateway, Lambda. It’s also assumed you have some Python and Flask knowledge. I also won’t dive into much details of why you should be building serverless applications. There are plenty of articles out there on that.


## Tech Stack

In this tutorial we will use:

- AWS
- Python 3.6
- Zappa
- Flask
- Wand/ImageMagick


## Architecture

So we want to build this thing serverleslly, right ? We will use some of the AWS product arsenal for that. And here’s what the general architecture looks like:

Thanks to [Cloudcraft](https://cloudcraft.co/) for this nice diagram 🙂

![http://www.99serverless.com/wp-content/uploads/2017/11/architecture.png](http://www.99serverless.com/wp-content/uploads/2017/11/architecture.png)


- **API Gateway**: API Gateway is responsible for handling incoming HTTP requests. Its main job is to proxy the requests to Lambda, and forward the response back to the user.
- **CloudFront**: It works as a CDN (Content Delivery Network), which, in plain English, means it will cache responses (resulting images) and deliver according to geographic location of the user. A CloudFront distribution is attached by default to the API Gateway deployment, BUT guess what, it’s not designed for caching 😒, [as you can see here](https://forums.aws.amazon.com/message.jspa?messageID=646291). So we will need to include our own CloudFront distribution.
- **Lambda**: This is where our business logic lives. All the code handling the HTTP request, which is basically transforming the input image and generating a binary image response. This will be written in Python 3.6 and take advantage of the fact that Lambda instances comes with ImageMagick built-in. To be more precise we will be using [Wand](http://docs.wand-py.org/en/0.4.4/), “a simple [ImageMagick](http://www.imagemagick.org/script/index.php) binding for Python”. Lambda will scale automatically according to the demand.
- **S3**: One of the assumption of our services is that the input images we want to process is available at S3. That means images must be uploaded directly to S3 prior the request. This is actually very scalable serverless architecture, in the sense that we won’t have the headache of managing an upload server, since S3 handles that beautifully for us.


## Coding


The full code for **imgy** is available [HERE](https://github.com/joarleymoraes/imgy).  And here are some of the important aspects of it.


## HTTP Handler

This is coded in Flask, which does most of the heavy lifting for us in terms of HTTP handling. So we are basically downloading the source image (given by `s3_key`) from S3, then applying all specified operations (`ops`) coming from the query string. After that, we convert the image to binary and attach it to the HTTP response via `send_file` helper from Flask. Got part of that snippet from [here](https://blog.zappa.io/posts/serving-binary-data-through-aws-api-gateway-automatically-with-zappa).

{% gist 600564bc497cf11a6d8e4c153ea8303f %}

## Wand for the Win

Below you can see the image modification part using Wand. This code can be easily extended to include more operations, as you can stack up new image modifiers. Notice they are independent of each other, that’s how ImageMagick is designed.


{% gist afa74580ecad1b31b773e0624482518e %}


## Supported Transformations

All transformations below are currently supported and they can be applied independently of each other:

- `w`: sets image width
- `h`: sets image height
- `fm`: sets image format, e.g.: png, jpeg, etc. All supported by ImageMagick.
- `q`: sets compression quality, in case it’s lossy format. Value must be between 1 to 100.

## Binary Response

Until recently, it was very clumsy to support binary responses in API Gateway; and this is key to make our solution to work. Hopefully, [AWS improved the support last year](https://aws.amazon.com/about-aws/whats-new/2016/11/binary-data-now-supported-by-api-gateway/)  and it’s much simpler to integrate 👏 👏. Zappa took advantage of this and added binary support on its version 0.36.0.

## Caching

Now, let’s add the necessary headers to our response so that CloudFront is able to cache images. First, we will use the decorator `@app.after_request` on all responses. Then we need to tell to the CloudFront that they should cache the images for `CACHE_MAX_AGE` seconds. Finally we set the cache as `public`, rather than private. That’s it, we are all set for caching.


{% gist 05b1783e65f3f5487441ff39dc4010e4 %}


You are able to verify this is working when you see a header in the response that is set as `x-cache: Hit from cloudfront`, instead of `x-cache: Miss from cloudfront`. Plus you can check the Lambda function logs and you will notice the function is not invoked after caching.


## CORS

One important feature imgy should have is the ability to support CORS (Cross-Origin Resource Sharing) requests. Meaning, we want to let incoming requests from an external domain. For example, if we were to embed the following image in a HTML, it would not work if CORS was disabled.


{% gist b9f176fbae6e94236a101cb6e9766e62 %}

CORS support is achieved by simply adding `flask-cors` extension.


Let’s Deploy
Configure first your environment. At zappa_settings.py you SHALL change:

- `s3_bucket`: the S3 bucket where we will store our (zappa) deployment packages. You don’t need to create in AWS, Zappa will do it for you.
NOTE: make sure the default AWS CLI profile has ADMIN permission.

You MAY also configure:

`aws_region`: the AWS region to where you want to deploy the app
At `imgy/settings.py` you SHALL change:

`imgy_bucket`: the S3 bucket from where we will get input images.
You MAY also configure:

`imgy_cache_max_age`: Define the cache Control header max-age in seconds.
To install let’s perform some commands:


```
virtualenv -p python3.6 venv
source venv/bin/activate
pip install -r requirements.txt
deploy zappa
```

## Adding CloudFront (optional)

You may add a custom CloudFront distribution, which will add a caching layer to your service. Be aware that this step will take about 15-20 minutes to finish. At the end the CloudFront URL will be generated, and use that instead of the API Gateways’.

`./add_cloud_front.sh <api_id>`


E.g.: If the API Gateway URL is `https://vk05slewjg.execute-api.us-west-2.amazonaws.com/api`, then your id is `vk05slewjg`.

NOTE: you might need to update awscli to latest version: `sudo pip install awscli --upgrade --user`. Install in you OS environment, not in the virtualenv.


## UnDeploy
To remove all AWS components created:

```
./remove_cloud_front.sh
undeploy zappa
```


## Live Demo
API Gateway URL:

[https://vk05slewjg.execute-api.us-west-2.amazonaws.com/api/cloud.png?w=100&h=100&fm=jpg&q=50](https://vk05slewjg.execute-api.us-west-2.amazonaws.com/api/cloud.png?w=100&h=100&fm=jpg&q=50)

CloudFront URL:

[https://d3htvglddphhs8.cloudfront.net/cloud.png?w=100&h=100&fm=jpg&q=50](https://d3htvglddphhs8.cloudfront.net/cloud.png?w=100&h=100&fm=jpg&q=50)

## That’s It
Post your comments below if you run into any issues or have questions.

Liked it ? Share this with your friends.



