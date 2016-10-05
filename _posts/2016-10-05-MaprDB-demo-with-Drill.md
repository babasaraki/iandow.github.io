---
layout: post
title: How To Persist Kafka JSON Streams in MapR-DB and query with SQL using Apache Drill
tags: [mapr, kafka, maprdb, drill, json]
---

# Streaming data is like, "Now you see it. Now you don't!"

One of the challenges when working with streams, especially streams of fast data, is the transitory nature of the data. Streams are characterized by a Time To Live (TTL) which defines the age at which messages are deleted. Ideally, noone would want messages older than the TTL, and for many applications that's a valid assumption. For example, in IoT applications sensor data probably has little to no value after it has aged. But in other cases, such as with insurance or banking applications, record-retention laws often require data to be persisted far beyond the point at which the data has any practical value to streaming analytics. In these cases, we must be able to persist streaming data without loosing any messages even if the responsiveness of our database lags behind the throughput of our stream or our stream consumers suffer an unexpected failure.

# How can I write Responsive and Elastic stream consumers?

The Kafka API allows us to update consumer cursor positions for streams whenever we want, so if we only update the cursor after we've processed a message, then we can avoid skipping messages. And by replicating topics across a cluster of nodes (even spanning various geographic locations) we can be very resilient to failures. Resiliency? Check!

Kafka also allows us to partition our topics so we can consume messages from a stream in a distributed and concurrent manner. That allows us to scale up and down in response to changes in streaming throughput. Elasticity? Check!

# What does this look like in code?

Below, I'm going to show you a Java application the consumes from MapR Streams using the Kafka API and persists each consumed message to MapR-DB. I will also show you how to query those database tables with SQL using [Apache Drill](https://drill.apache.org/). Why Drill? Because it allows us to use ANSI-SQL to query a non-relational datastore, containing schemaless records, like JSON.
 
Here is the Java code that illustrates how to consume from a MapR Stream using the Kafka API and persist each streamed record as a JSON document in Mapr-DB:
	
[Persister.java](https://gist.github.com/iandow/f6376264c2281d1c0e3a2485e86c9f23)
[Tick.java](https://gist.github.com/iandow/92d3276e50a7e77f41e69f5c69c8563b)

In that example, I read records from a MapR Stream, like this:
{% highlight java %}
ConsumerRecords<String, byte[]> msg = consumer.poll(TIMEOUT);
...
Iterator<ConsumerRecord<String, byte[]>> iter = msg.iterator();
while (iter.hasNext()) 
    ConsumerRecord<String, byte[]> record = iter.next();
{% endhighlight %}

Then I cast them to a JSON annotated POJO class called "Tick", like this:
{% highlight java %}
Tick tick = new Tick(record.value());
{% endhighlight %}

Then I create a MapRDB Document from that object, like this:
{% highlight java %}
Document document = MapRDB.newDocument((Object)tick);
{% endhighlight %}

And I write each Document to a table in MapR-FS, like this:
{% highlight java %}
table.insertOrReplace(tick.getTradeSequenceNumber(), document);
{% endhighlight %}

# How do I query MapR-DB tables?

Now that we have data persisted in MapR-DB tables, how can I query it?  

My MapR-DB table is named `/user/mapr/ticktable`, which on the filesystem is located at `/mapr/ian.cluster.com/user/mapr/ticktable`. First I'll show how to query this table with MapR `dbshell`, then with Apache Drill.

Here's how I read it with dbshell:

{% highlight bash %}
mapr dbshell
    find /user/mapr/ticktable
{% endhighlight %}

Here's how I read it with Drill:

{% highlight bash %} 
/opt/mapr/drill/drill-1.6.0/bin/sqlline -u jdbc:drill:
{% endhighlight %}

Either of the following two SELECT statements will work:

{% highlight sql %} 
SELECT * FROM dfs.`/mapr/ian.cluster.com/user/mapr/ticktable`;
SELECT * FROM dfs.`/user/mapr/ticktable`;
{% endhighlight %} 

![SQL Result](http://iandow.github.io/img/drill_query.png)

# References

I used Java 8 and MapR 5.2 to develop this example.
 
The code I referenced was mostly copied from the following reference application, which includes documentation on how to compile and run:

[GitHub - mapr-demos/finserv-application-blueprint](https://github.com/mapr-demos/finserv-application-blueprint): Example blueprint application for processing high-speed trading data. 
 
Here are some useful links to APIs and utilities I used above:

	- [mapr dbshell](http://maprdocs.mapr.com/home/MapR-DB/JSON_DB/getting_started_json_ojai_using_maprdb_shell.html)
	- [MapRDB Javadoc](http://maprdocs.mapr.com/apidocs/maprdb_json/51/com/mapr/db/MapRDB.html)