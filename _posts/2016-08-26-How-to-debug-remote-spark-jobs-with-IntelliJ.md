---
layout: post
title: How To Debug Remote Spark Jobs With IntelliJ
tags: [intellij, spark]
---

Application developers often use debuggers to find and fix defects in their code. Attaching a debugger to a running application is straightforward when the runtime is local on a laptop but trickier when that code runs on a remote server. This is even more confusing for Big Data applications since they typically run in a distributed fashion across multiple remote cluster nodes.  Fortunately, for Big Data applications implemented with the Apache Spark framework, it's actually pretty easy to attach a debugger even as they run across a remote multi-node cluster.

My favorite IDE is IntelliJ. I use it to develop Spark applications that run on remote multi-node clusters. I configure maven to compile my application and all its dependencies into a single jar, then after I build my jar file I upload it to my remote cluster and run it like this:

{% highlight bash %}
$ /opt/mapr/spark/spark-1.6.1/bin/spark-submit --class com.mapr.test.BasicSparkStringConsumer /mapr/myclust1/user/iandow/my-streaming-app-1.0-jar-with-dependencies.jar /user/iandow/mystream:mytopic
{% endhighlight %}

In order to attach the IntelliJ debugger to my spark application I'll define the SPARK_SUBMIT_OPTS environment variable and run it like this:

{% highlight bash %}
$ export SPARK_SUBMIT_OPTS=-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=4000
$ /opt/mapr/spark/spark-1.6.1/bin/spark-submit --class com.mapr.test.BasicSparkStringConsumer /mapr/myclust1/user/iandow/my-streaming-app-1.0-jar-with-dependencies.jar /user/iandow/mystream:mytopic
{% endhighlight %}

After running that command, it will wait until you connect your debugger, as shown below:

{% highlight bash %}
$ /opt/mapr/spark/spark-2.0.1/bin/spark-submit --class com.mapr.demo.finserv.SparkStreamingToHive nyse-taq-streaming-1.0-jar-with-dependencies.jar /user/mapr/taq:sender_0310
Listening for transport dt_socket at address: 4000
{% endhighlight %}

Now you can configure the IntelliJ debugger like this, where 10.200.1.101 is the IP address of the remote machine where I'm running my Spark job:

![IntelliJ Debugger](http://iandow.github.io/img/IntelliJ%20debug%20config.png)

Start the debugger by clicking Debug under IntelliJ's Run menu. Once it connects to your remote Spark process you'll be off and running. Now you can set breakpoints, pause the Spark runtime, and do everything else you can normally do in a debugger.  Here's an example of what IntelliJ shows when pausing a Spark job with a breakpoint:

![IntelliJ Debugger](http://iandow.github.io/img/IntelliJ_debugger.png)

So, that's how you attach IntelliJ's debugger to a Spark application running on a remote cluster. Isn't that nice!

<br><br>
<div class="main-explain-area padding-override jumbotron">
  <img src="http://iandow.github.io/img/starbucks_coffee_cup.png" width="120" style="margin-left: 15px" align="right">
  <a href="https://www.paypal.me/iandownard" title="PayPal donation" target="_blank">
  <h1>Hope that Helped!</h1>
  <p class="margin-override font-override">
    If this post helped you out, please consider fueling future posts by buying me a cup of coffee!</p>
  </a>
  <br>
</div>