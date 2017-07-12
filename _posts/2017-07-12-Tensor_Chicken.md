---
layout: post
title: Using Tensorflow on a Raspberry Pi in a Chicken Coop
tags: [tensorflow, python]
bigimg: /img/tensorflow_banner.png
---

Ever since I first heard about Tensorflow and the promises of Deep Learning I've been anxious to give it a whirl.

<img src="http://iandow.github.io/img/tensorflow_logo.png" width="33%" align="right">

Tensorflow is a powerful and easy to use library for machine learning. It was open-sourced by Google in November 2015. In less than 2 years it has become one of the most popular projects on GitHub. I was introduced to Tensorflow at the [O'Reilly Strata Data conference](https://conferences.oreilly.com/strata) in San Jose last year. During a [presentation](https://amy-jo.storage.googleapis.com/talks/tf-strata.pdf) by developer evangelists from Google I saw several really fun image processing examples that used Tensorflow to transform or identify subjects in images. That presentation was to this particular machine learning novice, nothing less than jaw dropping.

Fast forward 6 months and I've just deployed a Tensorflow application to my chicken coop. It's sort of an over-engineered attempt to detect blue jays in my nesting box and chase them away before they break my eggs. The app detects movement in the nesting box with a camera attached to a Raspberry Pi, then identifies the moving creature using an image classification model implemented in Tensorflow and posts that result to [@TensorChicken's](https://twitter.com/TensorChicken) on Twitter. 

<img src="http://iandow.github.io/img/rhode_island_red.jpg" width="33%" align="left" hspace="20">

The fact that I'm using Tensorflow on a Raspberry Pi is laughable because it's so often associated with applications that perform collosally large computations across hundreds of servers. But Tensorflow is flexible and it can be used at scale, or not.


# How do you install Tensorflow on a Raspberry Pi?

Long gone are the days where embedded computers were difficult to use. Single-board computers like the Raspberry Pi are suprisingly powerful and usable in ways that are familiar to any Linux head. Turns out, it's really easy to install Tensorflow on the Raspberry Pi. Here are the commands I used to install Tensorflow on my Pi:

{% highlight bash %}
sudo apt-get update
sudo apt-get install python3-pip python3-dev
wget https://github.com/samjabrahams/tensorflow-on-raspberry-pi/releases/download/v1.1.0/tensorflow-1.1.0-cp34-cp34m-linux_armv7l.whl
sudo pip3 install tensorflow-1.1.0-cp34-cp34m-linux_armv7l.whl
sudo pip3 uninstall mock
sudo pip3 install mock
{% endhighlight %}

For more information about installing Tensorflow on the Pi, check these out:
- [https://github.com/samjabrahams/tensorflow-on-raspberry-pi](https://github.com/samjabrahams/tensorflow-on-raspberry-pi)
- [https://codelabs.developers.google.com/codelabs/tensorflow-for-poets/#0](https://codelabs.developers.google.com/codelabs/tensorflow-for-poets/#0)
- [https://svds.com/tensorflow-image-recognition-raspberry-pi/](https://svds.com/tensorflow-image-recognition-raspberry-pi/)


# How do you detect motion and capture images from a Raspberry Pi webcam?

I setup a webcam on my Pi with a nice lightweight application called [Motion](https://github.com/Motion-Project/motion). It works with any Linux-supported video camera and provides cool features to capture images and movies and trigger scripts when motion is detected. The camera I'm using is the [Raspberry Pi Camera Module V2 - 8 Megapixel, 1080p](https://www.amazon.com/dp/B01ER2SKFS/ref=cm_sw_r_tw_dp_x_NGAzzbTRQ64WJ).

Here are the commands I ran on the Pi to setup the webcam:

{% highlight bash %}
sudo apt-get update
sudo apt-get upgrade
sudo apt-get remove libavcodec-extra-56 libavformat56 libavresample2 libavutil54
wget https://github.com/ccrisan/motioneye/wiki/precompiled/ffmpeg_3.1.1-1_armhf.deb
sudo dpkg -i ffmpeg_3.1.1-1_armhf.deb
sudo apt-get install curl libssl-dev libcurl4-openssl-dev libjpeg-dev libx264-142 libavcodec56 libavformat56 libmysqlclient18 libswscale3 libpq5
wget https://github.com/Motion-Project/motion/releases/download/release-4.0.1/pi_jessie_motion_4.0.1-1_armhf.deb
sudo dpkg -i pi_jessie_motion_4.0.1-1_armhf.deb
sudo vi /etc/motion/motion.conf
{% endhighlight %}

There are a lot of options in the motion.conf file. For your reference here is my [motion.conf](https://gist.github.com/iandow/1abc620626af601bf73f529e49b3a7b4) file.

I also had to add "bcm2835-v4l2" to `/etc/modules`. Then after a reboot the webcam appeared on Motion's built-in HTTP server on port 8081 (for example: `http://10.1.1.2:8081`).

For more information about setting up a Raspberry Pi webcam server, check out [https://pimylifeup.com/raspberry-pi-webcam-server/](https://pimylifeup.com/raspberry-pi-webcam-server/).


# Building the Tensorflow model

My Tensorflow image classification model is derived from the Inception-v3 model. This model can be easily retrained to recognize groups of new images. I copied about 5000 images captured by the Motion service to my laptop and manually saved them in directories named according to the categories I intended to be used as their label. For example, all the images of an empty nest were saved in a directory called, `empty_nest`.  I also saved all the label names in a plain text file called `retrained_labels-chickens.txt`.

To incorporate my custom labels into the Inception-v3 model I ran the following command. This took about 30 minutes to complete on my laptop.

{% highlight bash %}
python retrain.py  --bottleneck_dir=bottlenecks-chickens   --how_many_training_steps=500   --model_dir=inception   --summaries_dir=training_summaries-chickens/basic   --output_graph=retrained_graph-chickens.pb   --output_labels=retrained_labels-chickens.txt   --image_dir=chicken_photos
{% endhighlight %}


# Testing the model

To test the model, I ran commands like the following one to verift that it produced sensible labels for images of my nesting box:

{% highlight bash %}
python3 /home/pi/tf_files/label_image-chickens.py test_image.png
{% endhighlight %}

Here is my [label_image-chickens.py](https://gist.github.com/iandow/a3745b95d2b80689f6fb12b1b8f9fc9e) script.

# Analyzing the model with Tensorboard

The `retrain.py` script outputs model performance data that can be analyzed in Tensorboard, which can be really useful to help understand how your model works (from a neurel network perspective) and how accurately your model makes predictions. But really I like Tensorboard most because it makes charts which help me beautify blog articles:

<img src="http://iandow.github.io/img/tensorboard_histogram.png" width="30%" align="center">
<img src="http://iandow.github.io/img/tensorboard_chart.png" width="30%" align="center">
<img src="http://iandow.github.io/img/tensorboard_histogram2.png" width="30%" align="center">

# Running the app

The model I generated in the previous step was contained in a file about 84 MBs large. [Tensorflow models can be compressed](https://www.youtube.com/watch?v=EnFyneRScQ8) but since the Raspberry Pi is so powerful I just left it as-is. Once the model file `retrained_graph-chickens.pb` was copied to the Pi I automated image classification by invoking the following bash script from one of the "on_motion_detect" properties defined in `motion.conf`. This script can be run automatically via Motion or manually from the shell. 

{% highlight bash %}
#!/bin/bash
sntp -s time.google.com
sleep 5
ls -1tr /home/pi/motion/*.jpg | tail -n 1 | while read line; do
date
echo "Tweeting file '$line'";
CLASSIFICATION=`python3 /home/pi/tf_files/label_image-chickens.py $line | head -n 3`;
PUBLICIP=`curl -s ifconfig.co | tr '.' '-'`
MESSAGE=`echo -e "${CLASSIFICATION}\nLive video: ${PUBLICIP}.ptld.qwest.net:8081"`
MEDIA_ID=`twurl -H upload.twitter.com -X POST "/1.1/media/upload.json" --file $line --file-field "media" | jq -r '.media_id_string'`;
twurl "/1.1/statuses/update.json?tweet_mode=extended" -d "media_ids=$MEDIA_ID&status=$MESSAGE";
done
{% endhighlight %}

The above script sends tweets with a utility called `twurl`. To install it I just ran `sudo gem install twurl`. It also requires that you create an app on [https://apps.twitter.com/](https://apps.twitter.com/) and authorize access via keys defined in `~/.twurlrc`. See [twurl docs](https://github.com/twitter/twurl) for more information. 

Here's how everything fits together:

<img src="http://iandow.github.io/img/tensorchicken_flow_diagram.png" width="66%" align="center">
<img src="http://iandow.github.io/img/tensorchicken_tweet.png" width="33%" align="center">

# Thinking beyond APIs, what are the challenges with Deep Learning for business?

This application was not very hard to build. Tensorflow, motion detection, and automatic tweeting are all things you can sort out pretty easily, but things change if take it try to deploy on a bigger scale. Imagine a high-tech chicken farm where potentially hundreds of chickens are continuously monitored by smart cameras looking predators, animal sicknesses, and other environmental threats. In scenarios like this, you'll quickly run into challenges dealing with the enormity of raw data. You don't want to disgard old data because you might need it in order to retrain future models. Not only can it be hard to reliably archive image data but it's also challenging to apply metadata to each image and save that information in a searchable database. There are other challenges as well:
 
- How do you deal with the enormity of raw data streams? 
- How do you reliably archive raw data and make it searchable?
- Where do you run computationally difficult Machine Learning workloads?
- As you retrain and improve Tensorflow models, how do you do version control and A/B testing?

These challenges are frequently encountered by people trying to operationalize applications that use machine learning and Big Data in production. Like any self respecting wizard you can try to figure these things out yourself, but they'll come a point  you'll find yourself wanting things that are outside the scope of any machine learning API. That's when you become my favorite person to talk to!  

<img src="http://iandow.github.io/img/mapr-red-background-logo.png" width="33%" align="right">

At [MapR](http://www.mapr.com), we sell a [Converged Data Platform](https://mapr.com/products/mapr-converged-data-platform/) that is designed to improve how data is managed and how applications access data. People like MapR because we provide better security, easier management, higher resiliance to failure, and faster performance than any other Big Data platform. An application running on MapR has direct access to data stored in files, tables, or streams. That data can include:

- structured and unstructured data,
- data in cold storage and real-time data in streams,
- and data on-premise, data in the cloud, or data at the IoT edge.

MapR integrates key technologies, including a vast Big Data filesystem, a NoSQL database, and a distributed streaming engine into its patented Converged Data Platform. MapR uses open APIs such as HDFS, HBase, Kafka, POSIX, and NFS because it makes it easy for users to easily harness the power of MapR's underlying platform.

So next time you're planning to deploy infrastructure Big Data or code for Deep Learning, contact me and lets talk shop! 

![MapR Converged Data Platform](http://iandow.github.io/img/the-mapr-converged-data-platform-stack.png)


# Conclusion

The science and math behind the Deep Learning is mindbogglingly sophisticated but Tensorflow has made it approachable by novice software programmers such as myself. It's crazy to think that only a few years ago an image classification application like I built for my chicken coop were unheard-of because the APIs for neurel networks simply weren't advanced enough. It's well known that we advance technology by standing on the shoulders of giants and as I watch tweets flow into [@TensorChicken's](https://twitter.com/TensorChicken) I can't help but reflect on the centuries of work which has lead some of the smartest people of our time to evolve Deep Learning to where it is today. 

Tensorflow was open-sourced by Google in 2015. In less than 2 years it has become one of the most popular projects on GitHub. It's API is simple, it's capabilities are vast, and it's supported by a passionate and growing community of developers trying to improve it. I think Tensorflow is really going to benefit humanity - and my chickens - in a big way.


<br><br>
<div class="main-explain-area padding-override jumbotron">
  <img src="http://iandow.github.io/img/paypal.png" width="120" style="margin-left: 15px" align="right">
  <p class="margin-override font-override">
  	Did you enjoy the blog? Did you learn something useful? If you would like to support this blog please consider making a small donation. Thanks!</p>
  <br>
  <div id="paypalbtn">
    <a class="btn btn-primary btn" href="https://www.paypal.me/iandownard/3.5">Donate via PayPal</a>
  </div>
</div>