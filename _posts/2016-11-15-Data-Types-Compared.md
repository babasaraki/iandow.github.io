---
layout: post
title: What data types are most suitable for fast Kafka data streams? [Part One]
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

1. Would you create and stream a JSON object with the StringSerializer for each row of text?
2. Would you create a POJO with attributes corresponding to all the fields you expect to ingest?
3. Would you create a data type with a single byte array attribute, and getter methods that return data fields by indexing into predefined positions in the byte array?

## Option #1

For option #1 we would represent each record as a org.json.simple.JSONObject type. Parsing and processing would look like this:

{% highlight java %}
// consume records from kafka as strings
ConsumerRecords<String, String> records = consumer.poll();
for (ConsumerRecord<String, String> record : records) {
    JSONObject json_data = new JSONObject();
    // parse raw data into a JSON object
    json_data.put("date", record.substring(0,9));
    json_data.put("symbol", record.substring(9,16));
    json_data.put("price", record.substring(16,26));
    json_data.put("volume", record.substring(26,39));
    // Processing and data transformation happens here. 
    // We can access data fields easily with the dot operator
    //   json_data.get(...)
    // Then we publish a new message to a new topic, like this:
    producer.send(new ProducerRecord<String, String>(key, json_data.toString()));
}
{% endhighlight %}

Although the JSONObject is generic and schemaless, there is no data validation. Furthermore, this approach creates a lot of JSON objects and performs far too many string operations. Parsing strings like this can be prohibitively expensive for fast data streams.

## Option #2

In option #2 we'll represent our data with a POJO or Java bean that would look something like this:

{% highlight java %}
public class Tick implements Serializable {
    private String date;
    private String symbol;
    private String price;
    private String volume;

    public String getDate() {
        return date;
    }
    public void setDate(String date) {
        this.date = date;
    }
    public String getSymbol() {
        return date;
    }
    public void setSymbol(String date) {
        this.date = date;
    }
    // getters and setters go here
}
{% endhighlight %}

Using that data structure, data processing in Kafka could look like this:

{% highlight java %}
// consume records from Kafka as byte arrays
ConsumerRecords<String, byte[]> records = consumer.poll();
for (ConsumerRecord<String, byte[]> record : records) {
    // convert each record value to POJO
    ByteArrayInputStream input_bytes = new ByteArrayInputStream(record.value());
    Object obj = new ObjectInputStream(input_bytes).readObject();
    // Parsing data fields happens automatically when we cast to the POJO type, here:
    Tick tick = (Tick)obj;
    // Processing and data transformation happens here. 
    // We can access data fields easily with the dot operator
    //   tick.getPrice()...
    // Convert output message to bytes
    ByteArrayOutputStream bos = new ByteArrayOutputStream();
    ObjectOutput out = new ObjectOutputStream(bos);
    out.writeObject(tick);
    byte[] value = bos.toByteArray();
    // Now we can publish a new message back to Kafka, like this:
    producer.send(new ProducerRecord<String, byte[]>(key, value));
{% endhighlight %}

Option #2 makes it easy to access attributes once the POJO has been instantiated, but compound data types are more costly to work with than native types because object attributes can potentially reside in inefficient memory locations.  And just as in option #1, parsing with string operations is expensive and potentially prohibitive for fast data streams.  Although the attributes of our POJO provide some level of data validation, it offers no flexibility for ingesting data that might contain unexpected or incomplete fields. Finally, this approach requires that you convert the POJO to/from bytes when consumeing or publishing to Kafka, otherwise you’ll get a SerializationException when Kafka tries to convert it to bytes using whatever serializer you specified (unless you wrote a custom serializer).

## Option #3
The data type for option #3 might look like this:

{% highlight java %}
public class Tick implements Serializable {
    private byte[] data;
    public Tick(byte[] data) {
        this.data = data;
    }
    public Tick(String data) {
        this.data = data.getBytes(Charsets.ISO_8859_1);
    }
    public byte[] getData() {
        return this.data;
    }
    @JsonProperty("date")
    public String getDate() {
        return new String(data, 0, 9);
    }
    @JsonProperty("symbol")
    public String getDate() {
        return new String(data, 10, 16);
    }
    @JsonProperty("price")
    public String getDate() {
        return new String(data, 16, 10);
    }
    @JsonProperty("volume")
    public String getDate() {
        return new String(data, 26, 13);
    }
{% endhighlight %}

Option #3 is probably the fastest. Keeping our data in one large array has the best possible locality and since all the data is on one area of memory cache-thrashing will be kept to a minimum. It also allows us to stream byte arrays with Kafka's native ByteArray serializer. We can still facilitate access to data fields with getter methods that return a substring of byte array, however this approach does not accomodate flexible schemas. One disadvantage of this approach is if incoming data contains unexpected fields or field lengths, ingesting the data may fail, or worse, silently ingest in a corrupt format. 

## Measuring Performance with JUnit 

Kafka fundmentally transports data as bytes arrays. So the method of convertting your data types to/from byte arrays plays a big part in how fast your Kafka pipeline can run. Specifically I'm looking at how fast the following three data types can be converted to/from byte arrays:

1. POJOs
2. JSON objects
3. JSON annotated byte array objects

We theorized that the overhead of byte serialization to/from Kafka can be minimized by utilizing byte arrays as your principle data type format. It's obvious right? Because there's no faster way to convert a data object to Kafka's byte array format than by formatting the data object a byte array to begin with.

To measure how well those data types perform in Kafka I implemented a JUnit test (see [TypeFormatSpeedTest.java](https://github.com/iandow/kafka_junit_tests/blob/master/src/test/java/com/mapr/sample/TypeFormatSpeedTest.java)) which measures how fast data can be read from a stream, parsed, and written back to another stream. When I ran it on a DS11 server in Azure, I saw the following results:

    [testByteSpeed] 523537 records streamed per second
    [testPojoSpeed] 70735 records streamed per second
    [testJsonSpeed] 28479 records streamed per second

This means it's more than 7x faster to stream data formatted as a byte array than as a POJO, and it's 18x faster to stream data formatted as byte arrays than as an org.json.simple.JSONObject.

## Conclusion

When you need built-in data validation it may be desireable to use other POJOs, JSON, or Avro encoded data types, but as far as serialization speed goes, it will always be faster to format your data types as byte arrays than anything else. 

To download the code used for this study, clone the [https://github.com/iandow/kafka_junit_tests](https://github.com/iandow/kafka_junit_tests) repository.



<br><br>
<div class="main-explain-area padding-override jumbotron">
  <img src="http://iandow.github.io/img/paypal.png" width="120" style="margin-left: 15px" align="right">
  <p class="margin-override font-override">
    Did you enjoy the blog? Did you learn something useful? If you would like to support this blog please consider making a small donation. Thanks!</p>
  <br>
  <div id="paypalbtn">
    <a class="btn btn-primary btn" href="https://www.paypal.me/iandownard/3.5">Donate via PayPal</a>
  </div>
</div>