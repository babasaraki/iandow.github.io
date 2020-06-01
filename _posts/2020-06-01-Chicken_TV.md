---
layout: post
title: Chicken TV
tags: [video, chickens]
---

About 3 weeks ago one of my chickens, a Plymouth Rock, went broody. So, I immediately bought a dozen fertilized eggs from a farmer near Portland Oregon and stuck them under her. The eggs should hatch on Saturday June 6. I can't wait to see how many hatch and what breeds they'll be. It's amazing to watch the hen on her clutch. I love the sounds she makes too. 

<!-- CSS  -->
<link href="https://vjs.zencdn.net/7.2.3/video-js.css" rel="stylesheet">
<!-- HTML -->
<video id='chicken-tv' class="video-js vjs-default-skin" width="640" height="360" poster="http://iandow.github.io/img/chicken_tv_poster.png" controls>
<source type="application/x-mpegURL" src="https://hctigsi3ocwd57.data.mediastore.us-east-1.amazonaws.com/Ian_gopro_test_a/main_720p30.m3u8">
</video>

I'll do my best to keep that stream up and running but any number of other things might bring it down, like precarious extension chords, curious kids, etc.

<img src="http://iandow.github.io/img/chicken_incubator.jpg" width="20%" style="margin-left: 15px" align="right" alt="Egg incubator">

Coincidentally, my son's kindergarten class also just hatched some chicks. They used an incubator that carefully controls humidity, temperature, and egg movement so conditions are perfect for embryo development. 

<img src="http://iandow.github.io/img/chicken_tv_poster.png" width="20%" style="margin-right: 15px" align="right" alt="Hen on a nest">

It's amazing how you don't have to do any of that if you use a hen! Mother nature just takes care of it all! All you have to do is keep out predictors like crows, raccoons, and such.

I hope you enjoy watching this as much as I do. I hope you enjoy ***Chicken TV***!


### Specs

To setup this video feed, I have a GoPro Hero8 camera in the chicken coop sending 720p video to an RTMP endpoint that I configured using AWS MediaLive. I also configured MediaLive to save my video stream on AWS MediaStore. MediaStore generates an m3u8 manifest file for the stream, which I specified in the configuration for the Javascript video player (video-js) that you see on this web page.

<!-- JS code -->
<!-- If you'd like to support IE8 (for Video.js versions prior to v7) -->
<script src="https://vjs.zencdn.net/ie8/ie8-version/videojs-ie8.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/videojs-contrib-hls/5.14.1/videojs-contrib-hls.js"></script>
<script src="https://vjs.zencdn.net/7.2.3/video.js"></script>

<script>
var player = videojs('hls-example');
player.play();
</script>