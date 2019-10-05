---
layout: post
title: Deepdive S3 CORS
tags: [aws, s3, python]
bigimg: /img/blue-water-blur-close-up-1231622-2.jpg
---

For several weeks I've been trying to diagnose CORS errors in a web component I built for uploading files to AWS S3. This has been one of the hardest software defects I've had to solve in a long time so I thought it would be a good idea to share what I learned along the way.

First some background. The application I've been building is designed to allow people to upload videos with a web browser and run them through a suite of AWS machine learning services that generate data about video, audio, or text media objects, then save that data to Elasticsearch so people can search video archives using any of said ML derived metadata. My application runs on a serverless framework called the Media Insights Engine designed by my team at AWS Elemental and published to [github](https://github.com/awslabs/aws-media-insights-engine).

## DropzoneJS Upload

I chose Vue.js to implement the front-end and [DropzoneJS](http://dropzonejs.com) to provide drag'n'drop file upload functionality that looks like this:

<img src="http://iandow.github.io/img/dropzone.gif" width="70%">

## Uploading with Presigned URLs for S3

Here's what's supposed to happen in the back-end when the upload is started:

1. The web browser sends two requests to an API Gateway endpoint which acts as the point of entry to a Lambda function that returns a presigned URL which can be used in a subsequent POST to upload a file to S3. 
2. The first request is an HTTP OPTIONS method to /upload. This is called a presigned CORS requests. The browser uses this to verify that the server will allow a POST with some CORS related headers. The server responds with an empty 200 OK.
3. The second request is an HTTP POST to /upload. The Lambda function responds with said presigned URL.
4. The browser submits another HTTP OPTIONS method to the S3 endpoint to check that it will allow a POST with some CORS related headers. The server should respond with an empty 200 OK.
5. Finally the browser uses the presigned URL response from step #2 to POST to the S3 endpoint with the file data.

There are a lot of ways this can go wrong. It doesn't help that browsers will often report several different types of faults as CORS issues even when your CORS policies are perfectly fine. For example, here's the error you'll get when an API Gateway endpoint rejects a request due to IP restrictions in an API's access policy:

<img src="http://iandow.github.io/img/cors_error.png" width="70%">

You'd be in good company if you read `Maybe CORS errors` in that error and thought you had a problem with your CORS policy. But you would be wrong because the cause was an API Gateway resource policy.

<img src="http://iandow.github.io/img/api_gateway.png" width="70%">

Setting up CORS for S3 buckets is not complicated. When you actually have a real CORS problem you can often solve it by reading this [CORS troubleshooting guide](https://docs.aws.amazon.com/AmazonS3/latest/dev/cors-troubleshooting.html). When I saw CORS errors in my browser, I read that guide and then went to make changes to the CORS policy in my S3 bucket. Strangely, by making changes to the policy the problem *sometimes* went away. However, this was totally sporadic and seemingly dependant on how my S3 requests were being handled by the replicated S3 back-end. Pretty soon I realized there was no problem with the CORS policy on my S3 bucket, which by the way looks like this:

<img src="http://iandow.github.io/img/s3_cors.png" width="70%">

Even with a solid CORS policy on my S3 bucket, I frequently encountered CORS errors accompanied by HTTP 307 Temporary Redirect errors on the CORS preflight request. A CORS preflight request is an HTTP OPTIONS request that checks to see if the server understands the CORS protocol ([reference](https://developer.mozilla.org/en-US/docs/Glossary/Preflight_request)).

Here's what a CORS preflight redirect looks like:

<img src="http://iandow.github.io/img/cors_preflight_redirect.png" width="70%">

Now, look closely at the preflight redirect. Where is it directing the browser? How is the redirected URL different from the original request?

The redirected URL is a region-specific URL. ***This was an important clue.***

<img src="http://iandow.github.io/img/redirected_url.png" width="70%">


Browsers won't redirect preflight requests because [reasons](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS#Preflighted_requests). However, after doing some research about S3 redirects [here](https://docs.aws.amazon.com/AmazonS3/latest/dev/UsingBucket.html), [here](https://aws.amazon.com/premiumsupport/knowledge-center/s3-http-307-response/
), [here](https://docs.aws.amazon.com/AmazonS3/latest/dev/BucketRestrictions.html
), and [here](https://docs.aws.amazon.com/AmazonS3/latest/dev/Redirects.html
), I realized that this region-specific URL was important. 

My fix started to come together pretty quickly after realizing DropzoneJS used a statically defined URL that was not region specific for S3 buckets. I also noticed 

After changing DropzoneJS to use URL provided in the response from `get_presigned_url()`, my uploads started working reliably in EVERY region.

I also noticed that the get_presigned_url() boto3 function in my Lambda function returned different results depending on the region it was deployed to. I was able to isolate this region dependency once I learned that you can create a region-dependant boto3 client by using botocore from Python like this:

```
s3_client = boto3.client('s3', region_name='us-west-2')
```

This was a suprise to me because according to the boto3 docs there is no option to specify a region for your s3 client. Having learned about the botocore approach, I will henceforth always initialize S3 clients with the latest signature_version and addressing style, like this:

```
s3_client = boto3.client('s3', region_name='us-west-2', config = Config(signature_version = 's3v4', 
```

You can use the following Python code to test region-specific presigned URL functionality:

{% highlight python %}
import requests
import boto3
# these options come from botocore
s3_client = boto3.client('s3', region_name='us-west-2', config = Config(signature_version = 's3v4', s3={'addressing_style': 'virtual'}))
<!-- s3_client = boto3.client('s3') -->
response = s3_client.generate_presigned_post('mie01-dataplanebucket-1vbh3c018ikls','cat.jpg')
with open('/Users/ianwow/Desktop/cat.jpg', 'rb') as f:
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


# Conclusions:

* Region specific S3 endpoints are required to upload to buckets in any region other than Virginia (us-east-1)
* Always use botocore Config options to initialize Python S3 clients with a region, sig 3/4, and virtual path addressing.
* Don't assume that you have a CORS issue when browsers report CORS errors because they may not be aware of lower level issues, such as DNS resolution of S3 endpoints or API access controls.



<p>Please provide your feedback to this article by adding a comment to <a href="https://github.com/iandow/iandow.github.io/issues/15">https://github.com/iandow/iandow.github.io/issues/15</a>.</p>

