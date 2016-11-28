---
layout: post
title: What data types are most suitable for fast Kafka data streams?
tags: [java, kafka]
bigimg: /img/highway.jpg
---

The data types you choose to use to represent data can have a big impact on how fast you can stream that data through Kafka. A typical Kafka pipeline includes multiple stages that access streaming data to perform some kind of operation. Each stage will typically need to consume messages from one topic, perform some operation on those messages, then publish the result to a new topic. The computational complexity for each message will be the cumulative cost to perform the following tasks:

1. Convert the message from byte array to a complex data type (e.g. POJO). This is called "deserialization".
2. Access one or more attributes in the deserialized object
3. Perform some operation on that data
4. Construct a new message based on results from the previous step, convert that message to a byte array, and publish to a new topic. This is called "serialization".

Step 3 will vary depending on what your application does, but there are only a couple different ways developers usually implement steps 1, 2, and 4. These steps correspond to serialization (and deserialization) and accessing message properties (i.e. data fields). In the following two sections we will look at how different data types effect these patterns for serialization and accessing data fields.

# What Serializers provide the most efficient conversion to/from byte arrays?

Kafka transports bytes. When you publish records to a Kafka topic, you must specify a serializer which can convert your data objects to bytes. Kafka provides two serializers out of the box. They are StringSerializer and ByteArraySerializer. To better understand when you would you use these, consider the case where the following textual data is ingested into Kafka. Each row is a seperate message and the information in each message is encoded into fixed-length fields defined by a custom schema. These records describe ticks on the New York Stock Exchange whose schema is defined [here](http://www.nyxdata.com/Data-Products/Daily-TAQ).

    080449201DAA T 00000195700000103000N0000000000000004CT1000100710071007 
    080449201DAA T 00000131000000107000N0000000000000005CT10031007 
    080449201DAA T 00000066600000089600N0000000000000006CT10041005 
    080449201DAA T 00000180200000105100N0000000000000007CT100310051009 
    080449201DAA T 00000132200000089700N0000000000000008CT100410051005 
    080449201DAA T 00000093500000089400N0000000000000009CT10031007 
    080449201DAA T 00000075100000105400N0000000000000010CT100410081006 
    080449201DAA T 00000031300000088700N0000000000000011CT1004100810081007 

How should you stream this data?

1. Create and stream a JSON object with the StringSerializer for each row of text?
2. Create a POJO with attributes corresponding to all the fields you expect to ingest?
3. Create a data type with a single byte array attribute, and getter methods that return data fields by indexing into predefined positions in the byte array.

Option #1's pseudocode looks like this:

{% highlight java %}
// raw_data is a string read from a file, socket,
// or a string dequeued from a Kafka topic
String raw_data = reader.readLine();
JSONObject json_data = new JSONObject();
// parse raw data into a JSON object
json_data.put("date", raw_data.substring(0,9));
json_data.put("symbol", raw_data.substring(9,16));
json_data.put("price", raw_data.substring(16,26));
json_data.put("volume", raw_data.substring(26,39));
// Processing and data transformation happens here. Note, we can access data fields easily with the dot operator
//   json_data.get(...)
// Now we publish a new message to a new topic
producer.send(new ProducerRecord<String, String>(key, json_data.toString()));
{% endhighlight %}

This is generally not a good approach because there is no data validation, it creates a lot of JSON objects, and performs too many string operations. Parsing strings like can be prohibitively expensive for fast data streams.

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


