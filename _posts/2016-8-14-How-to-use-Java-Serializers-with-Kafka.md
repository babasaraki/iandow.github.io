Apache Kafka is a distributed pub-sub messaging system that scales horizontally and has built-in message durability and delivery guarantees. It's a cluster-based technology and has evolved from its origins at LinkedIn to become the defacto standard messaging system enterprises use to move massive amounts of data through transformation pipelines.

I've been using Kafka recently for some self-discovery type projects. I've used it to stream packet data from tcpdump processes so that I can visualize network performance stats in R Studio, and I've used it to process a massive stream of data reprepresenting real-time stock transactions on the New York Stock exchange. One of the things I've stuggled with is achieving the throughput I need to keep up with these real-time data streams. There are a lot of different use-case specific parameters and approaches to performance optimizations for Kafka, but the one parameter I want to talk about here is the data *serializer*.

So what are serializers? Serializers define how objects can be translated to a byte-stream format. Byte streams are the universal language that operating systems use for I/O, such as for reading or writting objects to a file or database. Serialization is necessary in order to replicate application state across nodes in a cluster. Java provides a default serializer for every object, descibed [here](https://docs.oracle.com/javase/7/docs/platform/serialization/spec/serial-arch.html). You can define your own custom serliazer, as described [here](http://thecodersbreakfast.net/index.php?post/2011/05/12/Serialization-and-magic-methods).

Kafka stores and transports byte arrays in its queue. The String and Byte array serializers are provided by Kafka out-of-the-box, but if you use them for objects which are not Strings or byte arrays, you will be using Java's default serializer for your objects. 
The problem with that is that by using the default java serializer for Kafka you may create highly unportable serialization that other languages may have trouble decoding. 

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

In my use-cases, maximizing throuhgput has been far more important than preserving the convenience of accessing member variables in cleanly parsed and populated POJOs. If your goal is similar, I would suggest you stick with the byte array serializers rather than writting custom serializers for passing POJOs through your Kafka data pipeline.

