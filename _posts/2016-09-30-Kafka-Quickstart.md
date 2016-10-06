---
layout: post
title: How to quickly get started using Kafka
tags: [azure, mapr, kafka]
---

This post describes how to quickly install Apache Kafka on a one node cluster and run some simple producer and consumer experiments.

[Apache Kafka](http://kafka.apache.org) is a distributed streaming platform. It lets you publish and subscribe to streams of data like a messaging system. You can also use it to store streams of data in a distributed cluster and process those streams in real-time.

### Step 1: Create an Ubuntu server

This post is about installing Kafka, not Ubuntu, but if you don't have an Ubuntu server currently available, then I suggest creating one in Azure or Amazon AWS.

### Step 2: Download and Install Kafka

Downloading and installing Kafka is a piece of cake. Just [download](http://kafka.apache.org/downloads.html) the latest release and untar it, like this:

```
wget http://apache.cs.utah.edu/kafka/0.10.0.1/kafka_2.11-0.10.0.1.tgz
tar -xzf kafka_2.11-0.10.0.1.tgz
cd kafka_2.11-0.10.0.1
```

Congratulations, you just installed Kafka. Zookeeper was also in that tar ball, so you have that too. Kafka uses ZooKeeper, and we'll start both services in the next step.

#### Setup Kafka to start automatically on bootup
Here's how I configure Kafka to start automatically on Ubuntu 14.04:

{% highlight bash %}
sudo su
cp -R ~/kafka_2.11-0.10.0.1 /opt
ln -s /opt/kafka_2.11-0.10.0.1 /opt/kafka
{% endhighlight %}

Copy the following init script to /etc/init.d/kafka:

<script src="https://gist.github.com/superscott/a1c67871cdd54b0c8693.js"></script>

Make the kafka service with these commands:

{% highlight bash %}
chmod 755 /etc/init.d/kafka
update-rc.d kafka defaults
{% endhighlight %}

Now you should be able to start and stop the kafka service like this:

{% highlight bash %}
sudo service kafka start
sudo service kafka status
sudo service kafka stop
{% endhighlight %}

If you want to remove the Kafka service later, run `update-rc.d -f kafka remove`. 


### Step 3: Start Kafka and Zookeeper 

If you created a Kafka daemon service as described above, then just run `sudo service kafka start`, otherwise start it manually as described below.

First start the ZooKeeper service, like this:

```bin/zookeeper-server-start.sh config/zookeeper.properties```

Then start the Kafka service, like this:

```bin/kafka-server-start.sh config/server.properties```

We just started a single Kafka broker, which is just a cluster of size one. This is fine for testing purposes, but for real applications you'll want to setup more cluster nodes with the config/server.properties file, as described [here](http://kafka.apache.org/documentation.html#quickstart_multibroker).

### Step 4: Create a topic

```bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test```

### Step 5: Send some messages

```while true; do fortune | bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test; done```

### Step 6: Start a consumer

```bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic test --from-beginning```

## Java producer and consumer examples

Here's an example of a Java consumer [[citation]](http://www.tutorialspoint.com/apache_kafka/apache_kafka_consumer_group_example.htm):

{% highlight java %}
import java.util.Properties;
import java.util.Arrays;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.ConsumerRecord;

public class ConsumerGroup {
   public static void main(String[] args) throws Exception {
      if(args.length < 2){
         System.out.println("Usage: consumer <topic> <groupname>");
         return;
      }

      String topic = args[0].toString();
      String group = args[1].toString();
      Properties props = new Properties();
      props.put("bootstrap.servers", "localhost:9092");
      props.put("group.id", group);
      props.put("enable.auto.commit", "true");
      props.put("auto.commit.interval.ms", "1000");
      props.put("session.timeout.ms", "30000");
      props.put("key.deserializer",
         "org.apache.kafka.common.serialization.StringDeserializer");
      props.put("value.deserializer",
         "org.apache.kafka.common.serialization.StringDeserializer");
      KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(props);

      consumer.subscribe(Arrays.asList(topic));
      System.out.println("Subscribed to topic " + topic);
      int i = 0;

      while (true) {
         ConsumerRecords<String, String> records = consumer.poll(100);
            for (ConsumerRecord<String, String> record : records)
               System.out.printf("offset = %d, key = %s, value = %s\n",
               record.offset(), record.key(), record.value());
      }
   }
}
{% endhighlight %}

Here's how you would compile that Java consumer:

```javac -cp "./kafka_2.11-0.10.0.0/libs/*" ConsumerGroup.java```

Here's how you would run that Java consumer:

```java -cp ".:./kafka_2.11-0.10.0.0/libs/*" ConsumerGroup test my-group```

Here's a Java example of a Kafka producer:

{% highlight java %}
import java.util.*;

import org.apache.kafka.clients.producer.*;

public class TestProducer {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put("bootstrap.servers", "localhost:9092");
        props.put("acks", "all");
        props.put("retries", 0);
        props.put("batch.size", 16384);
        props.put("linger.ms", 1);
        props.put("buffer.memory", 33554432);
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

        Producer<String, String> producer = new KafkaProducer<String, String>(props);

        Random rand = new Random();
        for(int i = 0; i < 100; i++)
             producer.send(new ProducerRecord<String, String>("my-topic", Integer.toString(i), Integer.toString(rand.nextInt())));
        producer.close();
    }
}
{% endhighlight %}

Here's how you would compile that Java producer:

```javac -cp "./kafka_2.11-0.10.0.0/libs/*" TestProducer.java```

Here's how you would run that Java producer:

```java -cp ".:./kafka_2.11-0.10.0.0/libs/*" TestProducer```


# References

[Apache Kafka Documentation](http://kafka.apache.org/documentation.html)
