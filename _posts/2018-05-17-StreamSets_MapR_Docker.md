---
layout: post
title: Using StreamSets and MapR together in Docker
tags: [streamsets, mapr, docker, data pipelines]
bigimg: /img/cool-background-3.png
---

In this post I demonstrate how to integrate StreamSets with MapR in Docker. This is made possible by the MapR persistent application client container (PACC). The fact that ***any*** application can use MapR simply by mapping `/opt/mapr` through Docker volumes is really powerful! Installing the PACC is a piece of cake, too. 

# Introduction

I use [StreamSets](http://streamsets.com) a lot for creating and visualizing data pipelines. I recently discovered that I've been installing StreamSets the hard way, meaning I've been downloading their tar installer, but now I'm using Docker and I'm liking the isolation and reproducibility it provides. 

To use StreamSets with MapR, the mapr-client package needs to be installed on the StreamSets host. ***Alternatively*** (emphasized because this is important) you can run a separate CentOS Docker container which has the mapr-client package installed, then you can share `/opt/mapr` as a docker volume with the StreamSets container. I like this approach because the MapR installer (which you can download [here](https://mapr.com/download/)) can configure a mapr-client container for me! MapR calls this container the [Persistent Application Client Container (PACC)](https://mapr.com/products/persistent-application-client-container/).

Here is the procedure I used to create and configure the PACC and StreamSets in Docker:

# Start the MapR Client in Docker

Here's a short video showing how to create, configure, and run the PACC:

<a href="https://asciinema.org/a/jD6NnO7UDBCxbdARMYQ8mf0Wr"><img src="https://asciinema.org/a/jD6NnO7UDBCxbdARMYQ8mf0Wr.png" width="60%"></a>

For more information about creating the PACC image, see [https://maprdocs.mapr.com/home/AdvancedInstallation/CreatingPACCImage.html](https://maprdocs.mapr.com/home/AdvancedInstallation/CreatingPACCImage.html).

Here are the steps I used for creating the PACC:

{% highlight bash %}
wget http://package.mapr.com/releases/installer/mapr-setup.sh -P /tmp
/tmp/mapr-setup.sh docker client
vi /tmp/docker_images/client/mapr-docker-client.sh
  # Set these properties:
  # MAPR_CLUSTER=nuc.cluster.com
  # MAPR_CLDB_HOSTS=10.0.0.10
  # MAPR_MOUNT_PATH=/mapr
  # MAPR_DOCKER_ARGS="-v /opt/mapr --name mapr-client"
/tmp/docker_images/client/mapr-docker-client.sh
{% endhighlight %}

# Start StreamSets in Docker

Start the StreamSets docker container with the following command. 

```
docker run --restart on-failure -it  -p 18630:18630 -d --volumes-from mapr-client --name sdc streamsets/datacollector
```

Normally we would need to install the MapR client on the StreamSets host, but since we've mapped /opt/mapr from the PACC via docker volumes, the StreamSets host already has it!

Now you need to go to StreamSet's package manager and install the MapR libraries:

<img src="http://iandow.github.io/img/streamsets_mapr_install.png" width="70%">

You'll see several MapR packages in StreamSets.

- MapR 6.0.0
- MapR 6.0.0 MEP 4
- MapR Spark 2.1.0 MEP 3

You'll want to install the first one, "MapR 6.0.0". That package lets you use MapR filesystem, MapR-DB, and MapR Streams.  If you want Hive and cluster mode execution, then install "MapR 6.0.0 MEP 4" as well as "MapR 6.0.0". If you want Spark, then also install "MapR Spark 2.1.0 MEP 3".

For more details on why the MapR package was split up like this, see this particular commit [https://github.com/streamsets/datacollector/commit/9452a03489ddf8ae2af81be9afaa904c7e766a55#diff-fd75725ca8cdddff01e7533e9b740e44](https://github.com/streamsets/datacollector/commit/9452a03489ddf8ae2af81be9afaa904c7e766a55#diff-fd75725ca8cdddff01e7533e9b740e44)

After you install the package, don't forget to run the setup-MapR script and all that jazz as described in the setup guide.

You'll be prompted to restart StreamSets. After it's restarted, run these commands finish the MapR setup:

{% highlight bash %}
docker exec -u 0 -it sdc /bin/bash
export SDC_HOME=/opt/streamsets-datacollector-3.2.0.0/
export SDC_CONF=/etc/sdc
echo "export CLASSPATH=\`/opt/mapr/bin/mapr classpath\`" >> /opt/streamsets-datacollector-3.2.0.0/libexec/sdc-env.sh
/opt/streamsets-datacollector-3.2.0.0/bin/streamsets setup-mapr
{% endhighlight %}

Restart StreamSets again from the gear menu. 

<img src="http://iandow.github.io/img/streamsets_restart.png" width="60%">

When it comes up you will be able to use MapR in StreamSets data pipelines. Here's a basic pipeline example that saves the output of tailing a file to a file on MapR-FS:

<img src="http://iandow.github.io/img/streamsets_tail.png" width="60%">
<img src="http://iandow.github.io/img/streamsets_maprfs1.png" width="60%">
<img src="http://iandow.github.io/img/streamsets_maprfs2.png" width="60%">


<br>
<p>Please provide your feedback to this article by adding a comment to <a href="https://github.com/iandow/iandow.github.io/issues/11">https://github.com/iandow/iandow.github.io/issues/11</a>.</p>

<br><br>
<div class="main-explain-area padding-override jumbotron">
  <img src="http://iandow.github.io/img/paypal.png" width="120" style="margin-left: 15px" align="right">
  <p class="margin-override font-override">
  	Did you enjoy the blog? Did you learn something useful? If you would like to support this blog please consider making a small donation. Thanks!</p>
  <br>
  <div id="paypalbtn">
    <a class="btn btn-primary btn" href="https://www.paypal.me/iandownard/5">Donate via PayPal</a>
  </div>
</div>
