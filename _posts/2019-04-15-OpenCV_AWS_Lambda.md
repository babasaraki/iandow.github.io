---
layout: post
title: Running OpenCV as an AWS Lambda Function
tags: [computer vision, docker, kafka, mapr, twitter]
bigimg: /img/black-color-dark-1171480.jpg
---

This post describes how to package the OpenCV python library so it can be used in applications that run in AWS Lambda. AWS Lambda is a Function-as-a-Service (FaaS) offering from Amazon that lets you run code without the complexity of building and maintaining the underlying infrastructure. OpenCV is one of the larger python libraries. Packaging it together with application code in a monolithic zip file will work, but deploying it as an AWS Lambda Layer has the following advantages:

* Lambda Layers allow libraries to be shared across many functions without duplicating code.
* Lambda Layers enable AWS Lambda functions to be smaller. Smaller packages can be packaged and uploaded faster and also enables the web-based code editor in AWS Lambda to work (it only works for applications less than 3MB). 

The relative sizes of an AWS Lambda function packaged with OpenCV vs an AWS Lambda function that uses OpenCV via a Lambda layer, is shown below:

<img src="https://raw.githubusercontent.com/iandow/opencv_aws_lambda/master/images/lambda_function_sizes.png">

# Procedure

To illustrate this process, I've created a sample application that deploys OpenCV as an AWS Lambda layer which is used by a Lambda function to grayscale an image in AWS S3 and save it back to AWS S3. Sample code and documentation is provided at [https://github.com/iandow/opencv_aws_lambda](https://github.com/iandow/opencv_aws_lambda).

## Preliminary AWS CLI Setup: 
1. Install Docker on your workstation.
2. Setup credentials for AWS CLI (see http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html).
3. Create IAM Role with Lambda and S3 access:
```
# Create a role with S3 and Lambda exec access
ROLE_NAME=lambda-opencv_study
aws iam create-role --role-name $ROLE_NAME --assume-role-policy-document '{"Version":"2012-10-17","Statement":{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}}'
aws iam attach-role-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --role-name $ROLE_NAME
aws iam attach-role-policy --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole --role-name $ROLE_NAME
```

## Build OpenCV library using Docker

AWS Lambda functions run in an [Amazon Linux environment](https://docs.aws.amazon.com/lambda/latest/dg/current-supported-versions.html), so libraries should be built for Amazon Linux. You can build Python-OpenCV libraries for Amazon Linux using the provided Dockerfile, like this:

```
git clone https://github.com/iandow/opencv_aws_lambda
cd opencv_aws_lambda
docker build --tag=python-opencv-factory:latest .
docker run --rm -it -v $(pwd):/data python-opencv-factory cp /packages/cv2-python36.zip /data
```

## Deploy the AWS Lambda function with OpenCV Lambda layer.

1. Edit the Lambda function code to do whatever you want it to do. In this example we're using app.py from https://github.com/iandow/opencv_aws_lambda.
```
vi app.py
```

2. Publish the OpenCV Python library as a Lambda layer.
```
LAMBDA_LAYERS_BUCKET=lambda-layers-$ACCOUNT_ID
LAYER_NAME=cv2
aws s3 mb s3://$LAMBDA_LAYERS_BUCKET
aws s3 cp cv2-python36.zip s3://$LAMBDA_LAYERS_BUCKET
aws lambda publish-layer-version --layer-name $LAYER_NAME --description "Open CV" --content S3Bucket=$LAMBDA_LAYERS_BUCKET,S3Key=cv2-python36.zip --compatible-runtimes python3.6
```

3. Create the Lambda function:
```
zip app.zip app.py
```

4. Deploy the Lambda function:
```
# Create the Lambda function:
FUNCTION_NAME=opencv_layered
ACCOUNT_ID=$(aws sts get-caller-identity | jq -r ".Account")
BUCKET_NAME=ianwow
aws s3 cp app.zip s3://$BUCKET_NAME
aws lambda create-function --function-name $FUNCTION_NAME --timeout 10 --role arn:aws:iam::$ACCOUNT_ID:role/$ROLE_NAME --handler app.lambda_handler --region us-west-2 --runtime python3.6 --environment "Variables={BUCKET_NAME=$BUCKET_NAME,S3_KEY=$S3_KEY}" --code S3Bucket="$BUCKET_NAME",S3Key="app.zip"
```

7. Attach the cv2 Lambda layer to our Lambda function:
```
LAYER=$(aws lambda list-layer-versions --layer-name $LAYER_NAME | jq -r '.LayerVersions[0].LayerVersionArn')
aws lambda update-function-configuration --function-name $FUNCTION_NAME --layers $LAYER
```

### Test the Lambda function:
Our Lambda function requires an image as input. Copy an image to S3, like this:
```
aws s3 cp ./my_image.jpg s3://ianwow/images/my_image.jpg
```
Then invoke the Lambda function:
```
aws lambda invoke --function-name $FUNCTION_NAME --log-type Tail outputfile.txt
cat outputfile.txt
```

You should see output like this:
```
{"statusCode": 200, "body": "{\"message\": \"image saved to s3://ianwow/my_image-gray.jpg\"}"}
```

```
aws s3 cp ./my_image.jpg s3://ianwow/my_image-gray.jpg
open my_image-gray.jpg
```

<img src="https://github.com/iandow/opencv_aws_lambda/blob/master/images/my_image.jpg" width="200"> <img src="https://raw.githubusercontent.com/iandow/opencv_aws_lambda/master/images/my_image-gray.jpg" width="200">

### Clean up resources
```
aws s3 rm s3://ianwow/my_image-gray.jpg
rm my_image-gray.jpg
rm -rf ./app.zip ./python/
aws lambda delete-function --function-name $FUNCTION_NAME
LAYER_VERSION=$(aws lambda list-layer-versions --layer-name cv2 | jq -r '.LayerVersions[0].Version')
aws lambda delete-layer-version --layer-name cv2 --version-number $LAYER_VERSION
aws iam detach-role-policy --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole --role-name $ROLE_NAME
aws iam detach-role-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --role-name $ROLE_NAME
aws iam delete-role --role-name $ROLE_NAME
```


<p>Please provide your feedback to this article by adding a comment to <a href="https://github.com/iandow/iandow.github.io/issues/14">https://github.com/iandow/iandow.github.io/issues/14</a>.</p>

<div class="main-explain-area padding-override jumbotron">
  <img src="http://iandow.github.io/img/mustache-udnie.cropped.jpg" width="120" style="margin-left: 15px" align="right">
  <p class="margin-override font-override" style="margin-left: 15px">
  	I’m fundraising for <a href="https://us.movember.com">Movember</a>! Please consider donating $5 to help me support Men's health: <a href="https://mobro.co/iandownard">https://mobro.co/iandownard</a>
  </p>
  <img src="http://iandow.github.io/img/movember.jpg" width="60" style="margin-left: 30px" align="right">
  <br>
  <div id="paypalbtn">
    <a class="btn btn-primary btn" href="https://mobro.co/iandownard">Donate to Movember</a>
  </div>
</div>
