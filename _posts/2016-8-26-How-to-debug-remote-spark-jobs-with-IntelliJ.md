Application developers often use debuggers to assist with application development and fix problems in their code. Typically, developers run and debug their applications locally on their workstation, however one of the challenges with developing big data applications is that they're typically intended to be run on a multi-node cluster. My favorite IDE is IntelliJ, and I'm currently using it to develop a Spark streaming application. I've configured maven to compile my application and all its dependencies into a single jar. After I build my jar file, I manually rsync it to my remote cluster where I normally run it like this:

{% highlight bash %}
$ /opt/mapr/spark/spark-1.6.1/bin/spark-submit --class com.mapr.test.BasicSparkStringConsumer /mapr/myclust1/user/iandow/my-streaming-app-1.0-jar-with-dependencies.jar /user/iandow/mystream:mytopic
{% endhighlight %}

If I want to attach the IntelliJ debugger to it, then I run it like this:

{% highlight bash %}
$ export SPARK_SUBMIT_OPTS=-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=4000
$ /opt/mapr/spark/spark-1.6.1/bin/spark-submit --class com.mapr.test.BasicSparkStringConsumer /mapr/myclust1/user/iandow/my-streaming-app-1.0-jar-with-dependencies.jar /user/iandow/mystream:mytopic
{% endhighlight %}

In IntelliJ, my debugger is configured as follows:

![IntelliJ Debugger](http://iandow.github.io/img/IntelliJ%20debug%20config.png)

So, that's how you attach IntelliJ's debugger to a spark application running on a remote cluster. Isn't that nice!