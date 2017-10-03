---
layout: post
title: Joining streams and NoSQL tables for Customer 360 analytics in Spark.
tags: [spark, streaming, sql, machine learning, customer 360, master data management]
---

*"MapR-DB is the perfect database for Customer 360 applications"*. That's the tag line I used to describe a demo I created for MapR for the Strata Data Conference in New York, September 2017. I worked hard to satisfy both the marketing need for eye candy and the product management need for developer enablement. Often, the needs of marketing and product management are at odds because marketeers want to capture leads with flashy demos but product managers want to show off technical features appeal to discriminating tastes of developers in the know. It was hard to reconcile these opposing viewpoints while building my Customer 360 demo for MapR at Strata, but with the help of the (bokeh)[http://bokeh.pydata.org] data visualization framework and the MapR's database connector for Spark, I was able to achieve demonstrable elegance in both the front-end and back-end. 

![Call Center Analytical Portal](http://iandow.github.io/img/customer360_bokeh.gif)

My efforts to build flashiness in the front-end flash were vindicated more than once when a conference attendee asked if they could take a photo of my app. (Insert a mental fist pump here). Demo's as booth provocations need to be flashy, but in this case my back-end code was equally elegant. In essence, I showed a webapp that loaded RDDs in Spark from both Kafka and MapR-DB, then I joined those RDDs to perform churn predictions using a machine learning model and again saved that result to MapR-DB (so it could be immediately accessible by my production apps, which in this case was a notional customer service portal). The clickstream contains user clicks flowing through a Kafka topic. The CRM tables in MapR-DB contain properties for customers.  Here's what that data flow looked like:

![Clickstream Animation](http://iandow.github.io/img/clickstream_animation.gif)

The three advantages I'm trying to convey with this demo were

1. Flexible schema makes it easier to persist customer data
2. Operating streams and database services on one cluster makes it possible to integrate and analyze that data faster in Spark.
3. Operating database services and machine learning on one cluster makes it possible for production applications to instantly access analytical insights.

## Customer 360 databases must support flexible schemas.

MapR-DB as the perfect database for Customer 360 because it allows you to store data for customers with different properties in the same table.  Why does that matter? Well, for customer 360 you're trying to integrate as many datasets as possible. For example you might be trying to ingest all the data your customers expose publicly on social media, or you might be trying to ingest data they expose through your mobile app, but not all customers may use social media, and not all customer may use your mobile app. Nevertheless, in a NoSQL database you can store data for all customers in one table even if different columns are used for each customer.  The sparsity in columnar data is not a problem at all for MapR-DB. 

## Converging streams and database eliminates data movement.

Furthermore, MapR platform provides both distributed streaming and distributed database so we can load real-time data (like clickstreams) and table data (like CRM data) into Spark without moving data. That's really important because anytime you move data, especially Big Data, the analysis of that data becomes much slower.  In the world of Big Data, data movement is BAD!

## Production applications get smarter when they share a common database with Spark

The third major talking point relates to how we can take an output, like churn prediction, from ML processes and load it back into the master CRM data tables in MapR-DB so those insights become instantly accessible by production applications. (In MapR, I hear a lot of people call these "operational applications", but I like to just call them production apps).

# Using Zeppelin for Customer 360

To illustrate those value propositions, I wrote Zeppelin notebook which accomplishes the following tasks in Spark:

1. Read clickstream data from MapR Streams using Spark Streaming and the Kafka API
2. Output performance data for web traffic using Spark SQL
3. Combine clickstream data with master CRM data using the Spark connector to MapR-DB
4. Predict churn risk users in the clickstream with Spark ML.

You can see all this in full living color here:

https://www.zepl.com/viewer/notebooks/bm90ZTovL2lhbmRvdy8zZDQyOTgyN2Y3NWQ0ODFmYTVlZDRmYzgxZGE1M2M1NC9ub3RlLmpzb24

The code excerpts below show how those Spark tasks were implemented:

### How to load an RDD from Mapr-DB:

{% highlight scala %}
// MapR-DB table returned as an RDD where each row is a JSON document
val rdd = sc.loadFromMapRDB("/mapr/my.cluster.com/tmp/crm_data").select("name", "address", "first_visit", "zip", "sentiment", "persona", "churn_risk");
println("Number of records loaded from MapR-DB: " + rdd.count)
val stringrdd = rdd.map(x => x.getDoc.asJsonString())
val crm_df = sqlContext.read.json(stringrdd)
{% endhighlight %}

### Here's how to load an RDD from a topic in MapR Streams using the Kafka API:

{% highlight scala %}
case class Click(user_id: Integer, datetime: String, os: String, browser: String, response_time_ms: String, product: String, url: String) extends Serializable
val schema = StructType(Array(
      StructField("user_id", IntegerType, true),
      StructField("datetime", TimestampType, true),
      StructField("os", StringType, true),
      StructField("browser", StringType, true),
      StructField("response_time_ms", StringType, true),
      StructField("product", StringType, true),
      StructField("url", StringType, true)
    ))
val groupId = "clickstream_reader"
val offsetReset = "earliest"
val pollTimeout = "5000"
val Array(topicc) = Array("/tmp/clickstream:weblog")
val brokers = "kafkabroker.example.com:9092" // not needed for MapR Streams, needed for Kafka
val ssc = new StreamingContext(sc, Seconds(2))
val topicsSet = topicc.split(",").toSet
val kafkaParams = Map[String, String](
  ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG -> brokers,
  ConsumerConfig.GROUP_ID_CONFIG -> groupId,
  ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG ->
    "org.apache.kafka.common.serialization.StringDeserializer",
  ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG ->
    "org.apache.kafka.common.serialization.StringDeserializer",
  ConsumerConfig.AUTO_OFFSET_RESET_CONFIG -> offsetReset,
  ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG -> "false",
  "spark.kafka.poll.time" -> pollTimeout
)
val consumerStrategy = ConsumerStrategies.Subscribe[String, String](topicsSet, kafkaParams)
val messagesDStream = KafkaUtils.createDirectStream[String, String](
  ssc, LocationStrategies.PreferConsistent, consumerStrategy
)
val valuesDStream = messagesDStream.map(_.value())
valuesDStream.foreachRDD { (rdd: RDD[String], time: Time) =>
  // There exists at least one element in RDD
  if (!rdd.isEmpty) {
    val count = rdd.count
    println("count received " + count)
    val spark = SparkSession.builder.config(rdd.sparkContext.getConf).getOrCreate()
    import spark.implicits._

    import org.apache.spark.sql.functions._
    val df: Dataset[Click] = spark.read.schema(schema).json(rdd).as[Click]
    df.show(20,false)
    //The reason to use the createOrReplaceTempView( tableName ) method for a DataFrame is so that in addition to being able to use the Spark provided methods of a DataFrame, you can also issue SQL queries via the SparkSession.sql( sqlQuery ) method which use the DataFrame as a SQL table.
    df.createOrReplaceTempView("weblog_snapshot")
    spark.sql("select count(*) from weblog_snapshot").show
  }
}
ssc.start()
ssc.awaitTerminationOrTimeout(10 * 1000)
ssc.stop(stopSparkContext = false, stopGracefully = false)
{% endhighlight %}

### Here's how to join the RDDs we previously loaded from MapR Streams and MapR-DB:

{% highlight scala %}
val joinedDF = spark.sql("SELECT weblog_snapshot.datetime, weblog_snapshot.os, weblog_snapshot.browser, weblog_snapshot.response_time_ms,weblog_snapshot.product,weblog_snapshot.url, crm_table.*, case when crm_table.churn_risk >= 20 then 1 else 0 end as churn_label from weblog_snapshot JOIN crm_table ON weblog_snapshot.user_id == crm_table.user_id")
{% endhighlight %}

### Here's how to save an RDD to a file or to MapR-DB:

{% highlight scala %}
// Save an RDD as a JSON file on MapR-FS
predictions_df.write.mode(SaveMode.Overwrite).format("json").save("predictions.json")
// Load an RDD from a JSON file on MapR-FS
val rdd = sc.textFile("/mapr/my.cluster.com/user/mapr/predictions.json")
// Convert an RDD[String] into JSON for the MapR-DB OJAI Connector
val maprd = rdd.map(str => MapRDBSpark.newDocument(str))
// Persist an RDD with the MapR-DB OJAI Connector
maprd.saveToMapRDB("/tmp/realtime_churn_predictions", createTable = false, bulkInsert = true, idFieldPath = "id")
{% endhighlight %}



<br>
<p>Please provide your feedback to this article by adding a comment to <a href="https://github.com/iandow/iandow.github.io/issues/5">https://github.com/iandow/iandow.github.io/issues/5</a>.</p>

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
