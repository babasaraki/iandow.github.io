This is a short post. I'm developing a Spark streaming application in IntelliJ, and running it on a remote cluster. Normally I run it like this:

{% highlight bash %}
$ /opt/mapr/spark/spark-1.6.1/bin/spark-submit --class com.mapr.test.BasicSparkStringConsumer /mapr/myclust1/user/iandow/my-streaming-app-1.0-jar-with-dependencies.jar /user/iandow/mystream:mytopic
{% endhighlight %}


But if I want to attach the IntelliJ debugger to it, then I run it like this:

{% highlight bash %}
$ export SPARK_SUBMIT_OPTS=-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=4000
$ /opt/mapr/spark/spark-1.6.1/bin/spark-submit --class com.mapr.test.BasicSparkStringConsumer /mapr/myclust1/user/iandow/my-streaming-app-1.0-jar-with-dependencies.jar /user/iandow/mystream:mytopic
{% endhighlight %}

In IntelliJ, my debugger is configured as follows:

![IntelliJ Debugger](http://iandow.github.io/img/IntelliJ%20debug%20config.png)
