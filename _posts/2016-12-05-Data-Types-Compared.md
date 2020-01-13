---
layout: post
title: What data types are most suitable for fast Kafka data streams? [Part Two]
tags: [java, kafka]
bigimg: /img/autos-technology-vw-multi-storey-car-park-63294.jpg
---

In my [last post](http://www.bigendiandata.com/2016-11-15-Data-Types-Compared/) I explained how important it is to format data types as byte arrays rather than other types, such as POJOs or json objects, in order to achieve minimal overhead when serializing data records to Kafka's native byte array format.

However, although serialization may be faster byte arrays, what about reading data fields in stream records?  And what about data validation?  Byte arrays contain no information about what array offsets delmite various data fields and they contain no features for validating data, effectively placing the responsibility of data validation on downstream consumers - which could be redundant and inefficient. I'm going to attempt to address these questions in this post.


# What Data Types provide the most efficient access to attributes?

Normally, stream consumers will need to access attributes for the objects being streamed. How that access is provided can have a big impact on how fast a stream processor can run. In this blog post we'll explore the following three popular data types used for streaming data in Kafka:

1. Avro 
2. POJO
3. JSON

[Avro](https://avro.apache.org/docs/current/) is a data serialization system that serializes data with a user-specified schema. The schema is written in JSON format and describes the fields and their types. Here is an example:

{% highlight json %}
	{
	    "type": "record",
	 	"name": "User",
	 	"fields": [
		    {"name": "name", "type": "string"},
		    {"name": "favorite_number",  "type": ["int", "null"]},
		    {"name": "favorite_color", "type": ["string", "null"]}
		]
	}
{% endhighlight %}

The nice thing about Avro is that it gives you an easy way to define a schema that can be used to enforce the structure of  records and can be used to serialize and deserialize records to/from Kafka native byte streams. 

Everyone's familiar with Plain Old Java Objects (POJOs) and JSON Strings, so I won't go into those.

There are three processes every Kafka stage must do. Consume a record, deserialize it, access one or more attributes in each record, optionally transform the record, serialize it, then publish it to a new topic. Lets compare our three data classes in terms of how long it takes to access an attribute. 

Lets assume Kafka is using the ByteArray serializer, and focus on the data access complexity. *What's the fastest way to access an element in that array?* Using the JMH framework, we can measure this quite easily, with the following microbenchmarks:

{% highlight java %}
	@Benchmark
    public String AvroTest(MyRecord stream_record) {
        // convert byte array to avro, then access the str1 attribute
        GenericRecord record = stream_record.recordInjection.invert(stream_record.avro_bytes).get();
        return record.get("str1").toString();
    }

    @Benchmark
    public String SubstrTest(MyRecord stream_record) {
        // Assuming a known schema, access byte array elements directly.
        return new String(stream_record.string_bytes, 0, 5);
    }

    @Benchmark
    public String PojoTest(MyRecord stream_record) throws Exception {
        // convert byte array to POJO and access str1 attribute
        ByteArrayInputStream bis = new ByteArrayInputStream(stream_record.pojo_bytes);
        ObjectInput in = new ObjectInputStream(bis);
        Object obj = in.readObject();
        RecordType record = (RecordType)obj;
        return record.getStr1();
    }
{% endhighlight %}

The benchmark I ran on my laptop gave me this result:

	Benchmark                Mode  Cnt         Score       Error  Units
	MyBenchmark.AvroTest    thrpt  200   3360845.667 ±  51103.886  ops/s
	MyBenchmark.PojoTest    thrpt  200    241485.913 ±   3748.181  ops/s
	MyBenchmark.SubstrTest  thrpt  200  20081971.452 ± 114315.726  ops/s

# Conclusion

From a data access perspective, converting byte arrays to POJO is 100x slower than directly accessing the byte array elements. Converting to Avro is 10x slower. But when speed isn't everything, Avro's schema management is much more flexible and less error prone than directly accessing byte elements.

The code and instructions for running these benchmarks are posted at [https://github.com/iandow/kafka_jmh_tests](https://github.com/iandow/kafka_jmh_tests).


# Future Work

What's not done?  You can use the KafkaAvroSerializer to send messages of Avro type to/from Kafka. That might be faster than using the Generic types that we used in our Avro example.  We should check that out too... (tbd).

# References

Official Avro docs - [https://avro.apache.org/docs/1.8.1/gettingstartedjava.html#Compiling+and+running+the+example+code](https://avro.apache.org/docs/1.8.1/gettingstartedjava.html#Compiling+and+running+the+example+code)
Good Avro to Kafka blog article - [http://aseigneurin.github.io/2016/03/04/kafka-spark-avro-producing-and-consuming-avro-messages.html](http://aseigneurin.github.io/2016/03/04/kafka-spark-avro-producing-and-consuming-avro-messages.html)
Using the KafkaAvroSerializer - [http://docs.confluent.io/1.0/schema-registry/docs/serializer-formatter.html](http://docs.confluent.io/1.0/schema-registry/docs/serializer-formatter.html)


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