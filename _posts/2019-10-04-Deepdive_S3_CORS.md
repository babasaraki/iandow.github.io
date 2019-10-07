---
layout: post
title: Deep Dive into CORS Configs on AWS S3
tags: [aws, s3, python]
bigimg: /img/blue-water-blur-close-up-1231622-2.jpg
---

For several weeks I've been trying to diagnose Cross-Origin Resource Sharing (CORS) errors in a web component I built for uploading files to AWS S3. This has been one of the hardest software defects I've had to solve in a long time so I thought it would be a good idea to share what I learned along the way.

First some background. The application I've been building is designed to allow people to upload videos with a web browser and run them through a suite of AWS machine learning services that generate data about video, audio, or text media objects, then save that data to Elasticsearch so people can search video archives using any of said ML derived metadata. My application runs on a serverless framework called the [Media Insights Engine](https://github.com/awslabs/aws-media-insights-engine) designed by my team at AWS Elemental.

## Drag-and-Drop Uploads with DropzoneJS

I chose Vue.js to implement the front-end and [DropzoneJS](http://dropzonejs.com) to provide drag-and-drop file upload functionality, as shown below. My Vue.js component for Dropzone was derived from [vue-dropzone](https://github.com/rowanwins/vue-dropzone).

<img src="http://iandow.github.io/img/dropzone.gif" width="70%">

## Uploading to S3 with Presigned URLs

Here's what's supposed to happen in my application when a user uploads a file:

1. <img src="http://iandow.github.io/img/upload1.png" width="30%" style="margin-left: 15px" align="right"> <img src="http://iandow.github.io/img/upload2.png" width="30%" style="margin-left: 15px" align="right"> The web browser sends two requests to an API Gateway endpoint which acts as the point of entry to a Lambda function that returns a presigned URL which can be used in a subsequent POST to upload a file to S3. 
2. The first request is an HTTP OPTIONS method to my `/upload` endpoint. This is called a "preflight CORS request". The browser uses this to verify that the server (an API Gateway endpoint in my case) understands the CORS protocol. The server should respond with an empty 200 OK.
3. The second request is an HTTP POST to `/upload`. The prescribed Lambda function then responds with a presigned URL.
4. <img src="http://iandow.github.io/img/upload3.png" width="30%" style="margin-left: 15px" align="right"> <img src="http://iandow.github.io/img/upload4.png" width="30%" style="margin-left: 15px" align="right"> The browser submits another preflight CORS request to verify that the S3 endpoint understands the CORS protocol. Again, the S3 endpoint should respond with an empty 200 OK.
5.  Finally the browser uses the presigned URL response from step #3 to POST to the S3 endpoint with the file data.

## Configuring CORS on an S3 Bucket

Before using presigned URLs to upload to S3 you have to define a CORS policy on the S3 bucket so web clients loaded in one domain (e.g. localhost or cloudfront) can interact with resources in the S3 domain. 
Setting a CORS policy on an S3 bucket is not complicated but if you get it wrong you can often solve it with the suggestions mentioned in this [CORS troubleshooting guide](https://docs.aws.amazon.com/AmazonS3/latest/dev/cors-troubleshooting.html). Here's the CORS policy I used on my S3 bucket:

<img src="http://iandow.github.io/img/s3_cors.png" width="70%">

## What can go wrong?

There are a lot of different ways I found to break things (this happens to be my specialty). Sometimes I would neglect to configure a CORS policy on my S3 bucket. This would cause S3 to block my CORS preflight request with an HTTP 403 error like this:

<img src="http://iandow.github.io/img/cors_error2.png" width="70%">

I would also occasionally get the same error when I put the wrong CIDR block on the API Gateway endpoint for the Lambda function I used to get presigned URLs. 

<img src="http://iandow.github.io/img/api_gateway.png" width="70%">

Even with a solid CORS policy on my S3 bucket and access policies in API Gateway, I always encountered an `HTTP 307 Temporary Redirect` error on the CORS preflight request sent to the S3 endpoint in any region other than Virginia (us-east-1). A ***CORS preflight request*** is an HTTP OPTIONS request that checks to see if the server understands the CORS protocol ([reference](https://developer.mozilla.org/en-US/docs/Glossary/Preflight_request)). Here's what it looks like when a server redirects a CORS preflight request to a different endpoint:

<img src="http://iandow.github.io/img/cors_preflight_redirect.png" width="70%">

Now, look closely at the preflight redirect. Where is it directing the browser? How is the redirected URL different from the original request?

The redirected URL is a region-specific URL. ***This was an important clue.***

Browsers won't redirect preflight requests because [reasons](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#Preflighted_requests). After doing some research about S3 usage [here](https://docs.aws.amazon.com/AmazonS3/latest/dev/UsingBucket.html), [here](https://aws.amazon.com/premiumsupport/knowledge-center/s3-http-307-response/
), [here](https://docs.aws.amazon.com/AmazonS3/latest/dev/BucketRestrictions.html
), and [here](https://docs.aws.amazon.com/AmazonS3/latest/dev/Redirects.html
), I realized that my DropzoneJS component needed to use a region-specific S3 endpoint for CORS preflight requests. The default S3 endpoint is only valid for buckets created in Virginia!

## Creating presigned URLs the right way!

The solution to my problems started coming together after realizing my DropzoneJS implementation used a statically defined URL that worked in Virginia (us-east-1) but not for any other region. I also noticed that the `get_presigned_url()` boto3 function in my Lambda function returned different results depending on the region it was deployed to. I was able to isolate this region dependency once I learned that you can create a region-dependent S3 client by using [botocore.client.Config](https://botocore.amazonaws.com/v1/documentation/api/latest/reference/config.html) from Python like this:

```
s3_client = boto3.client('s3', region_name='us-west-2')
```

This was a surprise to me because according to the [boto3](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/s3.html) docs there is no option to specify a region for your S3 client. Having learned about the botocore approach, ***I will now always initialize S3 clients with a region name, the latest signature_version, and virtual host style addressing***, like this:

```
s3_client = boto3.client('s3', region_name='us-west-2', config = Config(signature_version = 's3v4', 
```

My uploads started working reliably in every region after changing the S3 client to use a region-specific configuration and changing DropzoneJS to use the URL provided in the response from `get_presigned_url()`.

### Sample code

You can use the following code to generate region-specific presigned URLs in a Python interpreter:

{% highlight python %}
import requests
import boto3
from botocore.config import Config

s3_client = boto3.client('s3', region_name='us-west-2', config = Config(signature_version = 's3v4', s3={'addressing_style': 'virtual'}))

response = s3_client.generate_presigned_post('mie01-dataplanebucket-1vbh3c018ikls','cat.jpg')
with open('/Users/myuser/Desktop/cat.jpg', 'rb') as f:
     files = {'file': ('cat.jpg', f)}
     requests.post(response['url'], data=response['fields'], files=files)
{% endhighlight %}

Here's what my `/upload` Lambda function looks like now:

{% highlight python %}
@app.route('/upload', cors=True, methods=['POST'], content_types=['application/json'])
def upload():
    region = os.environ['AWS_REGION']
    s3 = boto3.client('s3', region_name=region, config = Config(signature_version = 's3v4', s3={'addressing_style': 'virtual'}))
    # limit uploads to 5GB
    max_upload_size = 5368709120
    try:
        response = s3.generate_presigned_post(
            Bucket=(app.current_request.json_body['S3Bucket']),
            Key=(app.current_request.json_body['S3Key']),
            Conditions=[["content-length-range", 0, max_upload_size ]],
            ExpiresIn=3600
        )
    except ClientError as e:
        logging.info(e)
        raise ChaliceViewError(
            "Unable to generate pre-signed S3 URL for uploading media: {error}".format(error=e))
    except Exception as e:
        logging.info(e)
        raise ChaliceViewError(
            "Unable to generate pre-signed S3 URL for uploading media: {error}".format(error=e))
    else:
        print("presigned url generated: ", response)
        return response
{% endhighlight %}


# Takeaways:

Here are the key points to remember about uploading to S3 using presigned URLs:

* Always use region specific S3 endpoints when trying to upload to S3. ([reference](https://docs.aws.amazon.com/AmazonS3/latest/dev/Redirects.html))
* Always use botocore Config options to initialize Python S3 clients with a region, sig 3/4, and virtual path addressing. ([reference](https://docs.aws.amazon.com/AmazonS3/latest/dev/RESTAuthentication.html))
* Don't assume that you have a CORS issue when browsers report CORS errors because they may not be aware of lower level issues, such as DNS resolution of S3 endpoints or API access controls.



<p>Please provide your feedback to this article by adding a comment to <a href="https://github.com/iandow/iandow.github.io/issues/15">https://github.com/iandow/iandow.github.io/issues/15</a>.</p>

