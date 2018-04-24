---
layout: post
title: Building A Serverless Image Processing SaaS
categories: [API_Gateway, AWS, Flask, AWS_Lambda, Python, SaaS, Serverless, Zappa]
---

[http://www.99serverless.com/wp-content/uploads/2017/11/architecture.png](http://www.99serverless.com/wp-content/uploads/2017/11/architecture.png)


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


{% highlight python %}
@app.route('/<s3_key>', methods=['GET'])
def transform(s3_key):
    ops = request.args.to_dict()

    img_filename = s3.download(BUCKET, s3_key)

    if img_filename:
        output_img = image_transform(img_filename, ops)
    else:
        abort(404)

    with open(output_img, 'rb') as fp:
        mime_type = get_mime_type(output_img)
        return send_file(
                    io.BytesIO(fp.read()),
                    attachment_filename=output_img,
                    mimetype=mime_type
               )
{% endhighlight %}
