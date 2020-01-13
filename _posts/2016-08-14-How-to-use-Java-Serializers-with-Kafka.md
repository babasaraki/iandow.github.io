---
layout: post
title: How To Use Java Serializers With Kafka
tags: [java, kafka]
---

Apache Kafka is a distributed pub-sub messaging system that scales horizontally and has built-in message durability and delivery guarantees. It's a cluster-based technology and has evolved from its origins at LinkedIn to become the defacto standard messaging system enterprises use to move massive amounts of data through transformation pipelines.

I've been using Kafka recently for some self-discovery type projects. I've used it to stream packet data from tcpdump processes so that I can visualize network performance stats in R Studio, and I've used it to process a massive stream of data representing real-time stock transactions on the New York Stock exchange. One of the things I've struggled with is achieving the throughput I need to keep up with these real-time data streams. There are a lot of different use-case specific parameters and approaches to performance optimizations for Kafka, but the one parameter I want to talk about here is the data *serializer*.

So what are serializers? Serializers define how objects can be translated to a byte-stream format. Byte streams are the universal language that operating systems use for I/O, such as for reading or writing objects to a file or database. Serialization is necessary in order to replicate application state across nodes in a cluster. Java provides a default serializer for every object, described [here](https://docs.oracle.com/javase/7/docs/platform/serialization/spec/serial-arch.html). You can define your own custom serializer, as described [here](http://thecodersbreakfast.net/index.php?post/2011/05/12/Serialization-and-magic-methods).

Kafka stores and transports byte arrays in its queue. The String and Byte array serializers are provided by Kafka out-of-the-box, but if you use them for objects which are not Strings or byte arrays, you will be using Java's default serializer for your objects. If the JVM is unable to serialize your object using the default serializer, you will get a run-time error, like this:


	Exception in thread "Thread-2" org.apache.kafka.common.errors.SerializationException: 
	Can't convert key of class java.time.LocalTime to class class 
	org.apache.kafka.common.serialization.StringSerializer specified in key.serializer

Even if the default serializer works for your objects, you should still be careful using it because Java's serializer may not be compatible with the default serializers in other languages. In other words, by using the default Java serializer for Kafka you may create unportable serialization that other languages may have trouble decoding. 

You can create your own serializer for Kafka. A good example of that is [here](http://niels.nu/blog/2016/kafka-custom-serializers.html). 

Here's an excerpt for how I configure a Kafka producer:


{% highlight java %}
    private void configureProducer() {
    Properties props = new Properties();
    props.put("key.serializer",
            "org.apache.kafka.common.serialization.StringSerializer");
    props.put("value.serializer",
            "org.apache.kafka.common.serialization.ByteArraySerializer");

    producer = new KafkaProducer<String, String>(props);
}
{% endhighlight %}

In this example, I'm using the StringSerializer for the key, because my key is simply a String. However, for speed reasons, I'm publishing and consuming data in a byte array format. Even though my data has fields (in fact my objects are JSON objects represented as byte arrays) I'm pushing the task of parsing those fields as far down the data pipeline as possible. So for example, when I consume this data in Spark, only then will I parse the data (in conceivably very small buckets or windows). By minimizing the amount of processing done for data transformations such as converting JSON byte arrays into POJOs, we can maximize the throughput of data through Kafka.

In my use-cases, maximizing throughput has been far more important than preserving the convenience of accessing member variables in cleanly parsed and populated POJOs. If your goal is similar, I would suggest you stick with the byte array serializers rather than writing custom serializers for passing POJOs through your Kafka data pipeline.

## Further Reading

I've blogged extensively about performance optimization for Kafka stream pipelines and how serialization of various data types can significantly slow things down if you don't do it right. To read more, check out my blog on [What data types are most suitable for fast Kafka data streams](http://www.bigendiandata.com/2016-12-05-Data-Types-Compared/).

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