---
layout: post
title: Running MediaInfo as an AWS Lambda Function
tags: [aws, lambda, python, mediainfo, docker]
bigimg: /img/lens-582605_1920.jpg
---
<!-- bigimg copied with from https://pixabay.com/illustrations/lens-colorful-background-digital-582605/-->

<img src="http://iandow.github.io/img/mediainfo_logo.png" width="15%" style="margin-left: 15px" align="right" alt="MediaInfo Logo">

This post describes how to package [MediaInfo](https://mediaarea.net/en/MediaInfo) so it can be used in applications hosted by AWS Lambda. [AWS Lambda](https://aws.amazon.com/lambda/) is a cloud service from Amazon that lets you run code without the complexity of building and managing servers. MediaInfo is a very popular tool for people who do video editing, streaming, or transcoding. It tells you all about what's in an audio or video file, like how they're encoded, number of channels, bitrate, resolution, etc. Here's a screenshot for some of the data it provides:

<img src="http://iandow.github.io/img/mediainfo_screenshot.png" width="90%" alt="MediaInfo application"> 

The MediaInfo library can be published to AWS Lambda in two ways:

1. together with application code as a monolithic all-in-one Lambda function,
2. or as a Lambda layer. 

I like the Lambda layer approach because it reduces the size of the Lambda function and enables more of my application code to be displayed in the Lambda code viewer in the AWS console. Both the monolithic and layered deploy options are described [here](https://github.com/iandow/mediainfo_aws_lambda) but in this blog post I'm going to just describe the procedure for deploying MediaInfo as a Lambda layer.

# Procedure

I've created a sample application that deploys MediaInfo as an AWS Lambda layer which is used by a Lambda function to get metadata tags for a video file saved in AWS S3. The code and documentation is maintained at [https://github.com/iandow/mediainfo_aws_lambda](https://github.com/iandow/mediainfo_aws_lambda). 

## Preliminary AWS CLI Setup: 
1. Install Docker on your workstation.
2. Setup credentials for AWS CLI (see [the user guide](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html)).
3. Create IAM Role with Lambda and S3 access:
```
# Create a role with S3 and Lambda exec access
ROLE_NAME=lambda-MediaInfo_study
aws iam create-role --role-name $ROLE_NAME --assume-role-policy-document '{"Version":"2012-10-17","Statement":{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}}'
aws iam attach-role-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --role-name $ROLE_NAME
aws iam attach-role-policy --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole --role-name $ROLE_NAME
```

## Build MediaInfo library using Docker

It's kind of a pain in the ass to build MediaInfo for AWS Lambda, but don't worry I've made it easy. Just run the following commands:

```
git clone https://github.com/iandow/mediainfo_aws_lambda
cd mediainfo_aws_lambda
docker build --tag=pymediainfo-layer-factory:latest .
docker run --rm -it -v $(pwd):/data pymediainfo-layer-factory cp /packages/pymediainfo-python37.zip /data
```

Those commands do the following things:

* Install the `pymediainfo` Python wrapper for MediaInfo. But wait! We need more than a wrapper. We need the actual MediaInfo binaries, too. The next step does that.
* Download and compile the MediaInfo dynamic linked library (DLL). Make sure to compile that DLL with support for URL inputs so we can process video files from S3 without downloading them to the Lambda runtime, which only provides 512MB of writtable disk space.
* AWS Lambda functions run in an [Amazon Linux environment](https://docs.aws.amazon.com/lambda/latest/dg/current-supported-versions.html), so we'll use Docker to build the DLL in an Amazon Linux environment.
* Put the Python wrapper and DLL files in a PATH where Python runtimes for AWS Lambda expect to find them.


## Deploy the AWS Lambda function with MediaInfo Lambda layer.

1. Edit the Lambda function code to do whatever you want it to do. In this example we're using `app.py` from [https://github.com/iandow/MediaInfo_aws_lambda](https://github.com/iandow/MediaInfo_aws_lambda).
```
vi app.py
```

2. Publish the MediaInfo Python library as a Lambda layer.
```
ACCOUNT_ID=$(aws sts get-caller-identity | jq -r ".Account")
LAMBDA_LAYERS_BUCKET=lambda-layers-$ACCOUNT_ID
LAYER_NAME=pymediainfo
aws s3 mb s3://$LAMBDA_LAYERS_BUCKET
aws s3 cp pymediainfo-python37.zip s3://$LAMBDA_LAYERS_BUCKET
aws lambda publish-layer-version --layer-name $LAYER_NAME --description "pymediainfo" --content S3Bucket=$LAMBDA_LAYERS_BUCKET,S3Key=pymediainfo-python37.zip --compatible-runtimes python3.7 --license-info "This product uses MediaInfo (https://mediaarea.net/en/MediaInfo) library, Copyright (c) 2002-2020 MediaArea.net SARL."
```

3. Create the Lambda function:
```
zip app.zip app.py
```

4. Deploy the Lambda function:
```
BUCKET_NAME=pymediainfo-test-$(date +%s)
aws s3 mb s3://$BUCKET_NAME
# Upload a test video
wget https://vjs.zencdn.net/v/oceans.mp4
S3_KEY=videos/oceans.mp4
aws s3 cp oceans.mp4 s3://$BUCKET_NAME/videos/
# Create the Lambda function:
FUNCTION_NAME=pymediainfo_layered
ACCOUNT_ID=$(aws sts get-caller-identity | jq -r ".Account")
aws s3 cp app.zip s3://$BUCKET_NAME
aws lambda create-function --function-name $FUNCTION_NAME --timeout 20 --role arn:aws:iam::${ACCOUNT_ID}:role/$ROLE_NAME --handler app.lambda_handler --region us-west-2 --runtime python3.7 --environment "Variables={BUCKET_NAME=$BUCKET_NAME,S3_KEY=$S3_KEY}" --code S3Bucket="$BUCKET_NAME",S3Key="app.zip"
```

7. Attach the `pymediainfo` Lambda layer to the Lambda function:
```
LAYER=$(aws lambda list-layer-versions --layer-name $LAYER_NAME | jq -r '.LayerVersions[0].LayerVersionArn')
aws lambda update-function-configuration --function-name $FUNCTION_NAME --layers $LAYER
```

### Test the Lambda function:
Our Lambda function requires a video as input. Copy a video to S3, like this:
```
wget https://vjs.zencdn.net/v/oceans.mp4
aws s3 cp ./oceans.mp4 s3://$BUCKET_NAME/videos/oceans.mp4
```
Then invoke the Lambda function:
```
aws lambda invoke --function-name $FUNCTION_NAME /dev/stdout
```

You should see output like this (although with a much longer LogResult):
```
{
    "LogResult": "U1RBUlQgU..."
    "ExecutedVersion": "$LATEST",
    "StatusCode": 200
}
```

### Sample output

The outputfile.txt will contain metadata values for the oceans.mp4 video file, like this:
(I added line breaks in the json below, to make it more readable.)

{% highlight json %}
{
  "tracks": [
    {
      "track_type": "General",
      "count": "331",
      "count_of_stream_of_this_kind": "1",
      "kind_of_stream": "General",
      "other_kind_of_stream": [
        "General"
      ],
      "stream_identifier": "0",
      "count_of_video_streams": "1",
      "count_of_audio_streams": "1",
      "video_format_list": "AVC",
      "video_format_withhint_list": "AVC",
      "codecs_video": "AVC",
      "audio_format_list": "AAC LC",
      "audio_format_withhint_list": "AAC LC",
      "audio_codecs": "AAC LC",
      "complete_name": "/root/oceans.mp4",
      "folder_name": "/root",
      "file_name_extension": "oceans.mp4",
      "file_name": "oceans",
      "file_extension": "mp4",
      "format": "MPEG-4",
      "other_format": [
        "MPEG-4"
      ],
      "format_extensions_usually_used": "braw mov mp4 m4v m4a m4b m4p m4r 3ga 3gpa 3gpp 3gp 3gpp2 3g2 k3g jpm jpx mqv ismv isma ismt f4a f4b f4v",
      "commercial_name": "MPEG-4",
      "format_profile": "Base Media",
      "internet_media_type": "video/mp4",
      "codec_id": "isom",
      "other_codec_id": [
        "isom (isom/avc1)"
      ],
      "codec_id_url": "http://www.apple.com/quicktime/download/standalone.html",
      "codecid_compatible": "isom/avc1",
      "file_size": 23014356,
      "other_file_size": [
        "21.9 MiB",
        "22 MiB",
        "22 MiB",
        "21.9 MiB",
        "21.95 MiB"
      ],
      "duration": 46613,
      "other_duration": [
        "46 s 613 ms",
        "46 s 613 ms",
        "46 s 613 ms",
        "00:00:46.613",
        "00:00:46;12",
        "00:00:46.613 (00:00:46;12)"
      ],
      "overall_bit_rate_mode": "VBR",
      "other_overall_bit_rate_mode": [
        "Variable"
      ],
      "overall_bit_rate": 3949861,
      "other_overall_bit_rate": [
        "3 950 kb/s"
      ],
      "frame_rate": "23.976",
      "other_frame_rate": [
        "23.976 FPS"
      ],
      "frame_count": "1116",
      "stream_size": 16342,
      "other_stream_size": [
        "16.0 KiB (0%)",
        "16 KiB",
        "16 KiB",
        "16.0 KiB",
        "15.96 KiB",
        "16.0 KiB (0%)"
      ],
      "proportion_of_this_stream": "0.00071",
      "headersize": "16334",
      "datasize": "22998022",
      "footersize": "0",
      "isstreamable": "Yes",
      "encoded_date": "UTC 2013-05-03 22:51:07",
      "tagged_date": "UTC 2013-05-03 22:51:07",
      "file_last_modification_date": "UTC 2013-05-08 00:34:04",
      "file_last_modification_date__local": "2013-05-08 00:34:04"
    },
    {
      "track_type": "Video",
      "count": "378",
      "count_of_stream_of_this_kind": "1",
      "kind_of_stream": "Video",
      "other_kind_of_stream": [
        "Video"
      ],
      "stream_identifier": "0",
      "streamorder": "0",
      "track_id": 1,
      "other_track_id": [
        "1"
      ],
      "format": "AVC",
      "other_format": [
        "AVC"
      ],
      "format_info": "Advanced Video Codec",
      "format_url": "http://developers.videolan.org/x264.html",
      "commercial_name": "AVC",
      "format_profile": "Baseline@L3",
      "format_settings": "3 Ref Frames",
      "format_settings__cabac": "No",
      "other_format_settings__cabac": [
        "No"
      ],
      "format_settings__reference_frames": 3,
      "other_format_settings__reference_frames": [
        "3 frames"
      ],
      "internet_media_type": "video/H264",
      "codec_id": "avc1",
      "codec_id_info": "Advanced Video Coding",
      "duration": 46545,
      "other_duration": [
        "46 s 545 ms",
        "46 s 545 ms",
        "46 s 545 ms",
        "00:00:46.545",
        "00:00:46;12",
        "00:00:46.545 (00:00:46;12)"
      ],
      "bit_rate": 3859631,
      "other_bit_rate": [
        "3 860 kb/s"
      ],
      "maximum_bit_rate": 9263280,
      "other_maximum_bit_rate": [
        "9 263 kb/s"
      ],
      "width": 960,
      "other_width": [
        "960 pixels"
      ],
      "height": 400,
      "other_height": [
        "400 pixels"
      ],
      "sampled_width": "960",
      "sampled_height": "400",
      "pixel_aspect_ratio": "1.000",
      "display_aspect_ratio": "2.400",
      "other_display_aspect_ratio": [
        "2.40:1"
      ],
      "rotation": "0.000",
      "frame_rate_mode": "CFR",
      "other_frame_rate_mode": [
        "Constant"
      ],
      "frame_rate": "23.976",
      "other_frame_rate": [
        "23.976 (24000/1001) FPS"
      ],
      "framerate_num": "24000",
      "framerate_den": "1001",
      "frame_count": "1116",
      "color_space": "YUV",
      "chroma_subsampling": "4:2:0",
      "other_chroma_subsampling": [
        "4:2:0"
      ],
      "bit_depth": 8,
      "other_bit_depth": [
        "8 bits"
      ],
      "scan_type": "Progressive",
      "other_scan_type": [
        "Progressive"
      ],
      "bits__pixel_frame": "0.419",
      "stream_size": 22456564,
      "other_stream_size": [
        "21.4 MiB (98%)",
        "21 MiB",
        "21 MiB",
        "21.4 MiB",
        "21.42 MiB",
        "21.4 MiB (98%)"
      ],
      "proportion_of_this_stream": "0.97576",
      "writing_library": "Zencoder Video Encoding System",
      "other_writing_library": [
        "Zencoder Video Encoding System"
      ],
      "encoded_library_name": "Zencoder Video Encoding System",
      "encoded_date": "UTC 2013-05-03 22:50:47",
      "tagged_date": "UTC 2013-05-03 22:51:08",
      "codec_configuration_box": "avcC"
    },
    {
      "track_type": "Audio",
      "count": "280",
      "count_of_stream_of_this_kind": "1",
      "kind_of_stream": "Audio",
      "other_kind_of_stream": [
        "Audio"
      ],
      "stream_identifier": "0",
      "streamorder": "1",
      "track_id": 2,
      "other_track_id": [
        "2"
      ],
      "format": "AAC",
      "other_format": [
        "AAC LC"
      ],
      "format_info": "Advanced Audio Codec Low Complexity",
      "commercial_name": "AAC",
      "format_settings__sbr": "No (Explicit)",
      "other_format_settings__sbr": [
        "No (Explicit)"
      ],
      "format_additionalfeatures": "LC",
      "codec_id": "mp4a-40-2",
      "duration": 46613,
      "other_duration": [
        "46 s 613 ms",
        "46 s 613 ms",
        "46 s 613 ms",
        "00:00:46.613",
        "00:00:46:23",
        "00:00:46.613 (00:00:46:23)"
      ],
      "bit_rate_mode": "VBR",
      "other_bit_rate_mode": [
        "Variable"
      ],
      "bit_rate": 92920,
      "other_bit_rate": [
        "92.9 kb/s"
      ],
      "maximum_bit_rate": 104944,
      "other_maximum_bit_rate": [
        "105 kb/s"
      ],
      "channel_s": 2,
      "other_channel_s": [
        "2 channels"
      ],
      "channel_positions": "Front: L R",
      "other_channel_positions": [
        "2/0/0"
      ],
      "channel_layout": "L R",
      "samples_per_frame": "1024",
      "sampling_rate": 48000,
      "other_sampling_rate": [
        "48.0 kHz"
      ],
      "samples_count": "2237424",
      "frame_rate": "46.875",
      "other_frame_rate": [
        "46.875 FPS (1024 SPF)"
      ],
      "frame_count": "2185",
      "compression_mode": "Lossy",
      "other_compression_mode": [
        "Lossy"
      ],
      "stream_size": 541450,
      "other_stream_size": [
        "529 KiB (2%)",
        "529 KiB",
        "529 KiB",
        "529 KiB",
        "528.8 KiB",
        "529 KiB (2%)"
      ],
      "proportion_of_this_stream": "0.02353",
      "encoded_date": "UTC 2013-05-03 22:51:07",
      "tagged_date": "UTC 2013-05-03 22:51:08"
    }
  ]
}
{% endhighlight %}


### Clean up resources

Here's how to delete everything created above.

```
aws s3 rm s3://$BUCKET_NAME/videos/oceans.mp4
aws s3 rb s3://$BUCKET_NAME/
aws s3 rm s3://$LAMBDA_LAYERS_BUCKET/pymediainfo-python37.zip
aws s3 rb s3://$LAMBDA_LAYERS_BUCKET
rm oceans.mp4
rm -rf ./app.zip ./python/
aws lambda delete-function --function-name $FUNCTION_NAME
LAYER_VERSION=$(aws lambda list-layer-versions --layer-name pymediainfo | jq -r '.LayerVersions[0].Version')
aws lambda delete-layer-version --layer-name pymediainfo --version-number $LAYER_VERSION
aws iam detach-role-policy --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole --role-name $ROLE_NAME
aws iam detach-role-policy --policy-arn arn:aws:iam::aws:policy/AmazonS3FullAccess --role-name $ROLE_NAME
aws iam delete-role --role-name $ROLE_NAME
```

<p>Please provide your feedback to this article by adding a comment to <a href="https://github.com/iandow/iandow.github.io/issues/17">https://github.com/iandow/iandow.github.io/issues/17</a>.</p>

<br><br>
<div class="main-explain-area padding-override jumbotron">
  <a href="https://www.paypal.me/iandownard" title="PayPal donation" target="_blank">
  <h1>Hope that Helped!</h1>
  <img src="http://iandow.github.io/img/starbucks_coffee_cup.png" width="120" style="margin-left: 15px" align="right">
  <p class="margin-override font-override">
    If this post helped you out, please consider fueling future posts by buying me a cup of coffee!</p>
  </a>
  <br>
</div>