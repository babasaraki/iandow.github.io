---
layout: post
title: What data class types provide the lightest overhead for Kafka pipeline processors?
tags: [java, kafka]
bigimg: /img/highway.jpg
---

The types you choose to use to represent data can have a big impact on how fast you can stream that data through Kafka. A typical Kafka pipeline includes multiple stages that access streaming data to perform some kind of operation. Each intermediate pipeline processor will probably consume messages from one topic, perform some operation on those messages, then publish the result to a new topic. Each processor will deserialize a message from Kafka, access one or more attributes in the message, perform some operation on that data, then serialize the result, and publish it to a new topic. Essentially, the operations that happen over and over again throughout the pipeline are 1) serialization (and deserialization) and 2) accessing streaming record attributes.  Lets look at each of these individually...

# What Serializers provide the most efficient conversion to/from byte arrays?

Kafka transports bytes. When you publish records to a Kafka topic, you must specify a serializer which can convert your data objects to bytes. Kafka provides two serializers out of the box. They are StringSerializer and ByteArraySerializer. To better understand when you would you use these, consider the case where textual data (e.g. JSON) read from a file or socket, is ingested into Kafka. How should you stream this data?

1. Create and stream a JSON object with the StringSerializer for each row of text?
2. Create a POJO with attributes corresponding to all the fields you expect to ingest?
3. Create a data type with a single byte array attribute, and getter methods that return data fields by indexing into predefined positions in the byte array.

Option #1 provides a flexible schema that won't bomb out when unexpected record formats are ingested.  But parsing the incoming text can be prohibitively expensive for fast data streams.

Option #2 makes it easy to access attributes once the POJO has been instantiated, but compound data types are more costly to work with than native types because object attributes can potentially reside in inefficient memory locations.  And again, parsing is expensive and potentially prohibitive for fast data streams.  Furthermore, this approach offers no flexibility for ingesting data containing unexpected fields. And finally, this approach requires that you implement a custom serializer.

Option #3 is probably the fastest. Keeping our data in one large array has the best possible locality and since all the data is on one area of memory cache-thrashing will be kept to a minimum. It also allows us to stream byte arrays with Kafka's native ByteArray serializer. We can still facilitate access to data fields with getter methods that return a substring of byte array, however this approach does not accomodate flexible schemas. If incoming data contains unexpected fields or field lengths, ingesting the data may fail, or worse, silently ingest in a corrupt format.

As far as speed goes, the closer you work with byte arrays, the better. But for the purposes of flexible schemas, sometimes it's necessary to use data formats such as stringified JSON or Avro encoding (see below).


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


