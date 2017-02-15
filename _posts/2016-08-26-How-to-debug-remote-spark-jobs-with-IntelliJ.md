---
layout: post
title: How To Debug Remote Spark Jobs With IntelliJ
tags: [intellij, spark]
---

Application developers often use debuggers to assist with application development and fix problems in their code. Typically, developers run and debug their applications locally on their workstation, however one of the challenges with developing big data applications is that they're designed to be run on a multi-node cluster. The presents a challenge for debugging because you have to attach a debugger to a process that's running on a remote machine. Fortunately, it's actually not that hard. 

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

Start the debugger by clicking Debug under IntelliJ's Run menu. Once it connects to your remote Spark process you'll be off and running. Now you can set breakpoints, pause the Spark runtime, and do everything else you can normally do in a debugger.  Here's an example of what IntelliJ shows when pausing a Spark with a breakpoint:

![IntelliJ Debugger](http://iandow.github.io/img/IntelliJ_debugger.png)

So, that's how you attach IntelliJ's debugger to a spark application running on a remote cluster. Isn't that nice!