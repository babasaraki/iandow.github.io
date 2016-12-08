---
layout: post
title: Kafka vs MapR Streams. Which is faster?
tags: [performance, kafka, mapr streams]
---

Performance is one of the most common reasons why enterprises choose MapR as their core platform for processing and storing big data. MapR consistently performs faster than any other big data platform for all kinds of applications, including YARN, distributed filesystem I/O, NoSQL database operations, and data streaming. In this post I'm focusing on the later, comparing the throughput of data streaming with MapR Streams and Apache Kafka.

MapR Streams is a cluster-based messaging system for streaming data at scale and it's integrated into the MapR Converged Data Platform. MapR Streams is similar to Kafka. Both systems use the same API for publish and subscribe, so applications written for Kafka can also run on MapR Streams. What differentiates MapR Streams are features such as global replication, security, multi-tenancy, high availability, and disaster recovery - all of which it inherits from the MapR Converged Data Platform. MapR Streams is also easier to manage than Kafka. For example, MapR Streams logically groups topics together so that policies such as time-to-live and access control can be easily configured for groups of topics.

There are speed advantages too. I've been looking at this a lot lately, trying to understand where and why MapR Streams out-performs vanilla Kafka. It has become abundantly clear that *MapR Streams can transport a much faster stream of data, with much larger message sizes, and to far more topics than what can be achieved with vanilla Kafka*. 

My comparisons focused on the following two questions:

1. How fast can I publish data into a stream?
2. How large can messages be before throughput diminishes?
3. How many topics can I publish data to?

I ran these comparisons as parameterized tests in JUnit which measure the throughput of a producer. No consumers were run during these tests. I have posted [source code and documentation](https://github.com/iandow/kafka_junit_tests) for my tests, so it should be possible to replicate results on your own gear if you are interested.

# My Setup

While I've been careful to ensure that I'm doing an apples-to-apples comparison, I wanted to compare Kafka an MapR-Streams as how they perform "off the shelf", without the burdon of tuning my test environment to perfectly optimize performance in each test scenario. So, I have pretty much stuck with the default settings except for Kafka's replication factor and synchronous send mode. I set Kafka's replication factor to 3 and acks to "all" (i.e. synchronous send mode) because MapR Streams replicates topics across a cluster by default because its underlying filesystem (MapR-FS) is distributed, and because MapR Streams defaults to synchronous send mode.

For my tests, I used a three node cluster of Ubuntu servers running on Azure with DS14 specs:

- Intel Xeon CPU E5-2660 2.2 GHz processor with 16 cores
- SSD disk storage with 64,000 MBps cached / 51,200 uncached max disk throughput
- 112GB of RAM
- virtual networking throughput between 1 and 2 Gbits/sec

Okay, now lets get to the results.

## MapR Streams achieves significantly higher throughput than Kafka.

In this study I measured how fast a single threaded producer could publish 1 million messages to single topic with 1 parition and 3x synchronous replication (i.e. the producer used the `acks=all` property). I ran that test for a variety of message sizes to see how message size affected throughput both in terms of MB/sec and messages/sec. The results show MapR Streams consistently achieving more than double the throughput of Kafka.

![throughput_MBytes_per_sec](http://iandow.github.io/img/tput-bytes.png)

## MapR Streams can handle significantly larger messages than Kafka.

MapR Streams demonstrates a much higher capacity for handling large message sizes, as shown below:

![throughput_records_per_sec](http://iandow.github.io/img/tput-bytes.png)

## MapR Streams can handle significantly more topics than Kafka.

Another major benefit that MapR Streams holds over Kafka relates how well it can handle large quantities of stream topics. I quantified this by measuring how fast a single threaded producer could publish 1 million messages to an increasingly large quantity of topics, configured with 1 partition and 3x synchronous replication. 

![throughput_topics](http://iandow.github.io/img/tput-topics.png)

# Why does MapR Streams transport data so much faster than Kafka?

These performance comparisons show such an advantage for MapR Streams that I really needed to understand why MapR Streams performed so well in order to trust them. So I spoke with a MapR Streams engineer and here's what I learned:

## Why does MapR Streams achieve more than 2x throughput than Kafka?

The Mapr Stream client has 64 flusher threads that concurrently flush data to the MapR server. This is different from  Kafka which uses the client application threads directly to flush to a Kafka broker. In the Kafka client there is only one thread that's actually working so your throughput is limited by that thread's CPU. In MapR, multiple RPCs can be sent out because we have 64 flushers working in parallel. In MapR, producer.send() does nothing but put messages onto a queue. MapR's internal flushers will process that queue in parallel. With Kafka, this is limited by the number of threads that have been created within the producer. 

Furthermore, replication in MapR is more efficient since the underlying storage (MapR-FS) is distributed. Callbacks for synchronous sends will be invoked the moment a message is written to the filesystem on one MapR node. With Kafka, those callbacks only execute after Zookeeper finishes copying messages through Zookeeper on each of the in-sync replicas. 

## Why does MapR Streams handle so many more topics than Kafka?

A topic is just metadata of a Mapr Stream, it does not introduce overhead to normal operations. The MapR Stream uses only one file for a stream no matter how many topics it has and it  inherits efficient I/O patterns from the core MapR filesystem (MapR-FS) which keeps files coherent and clean so that I/O operations can be efficiently buffered and addressed to sequential locations on disk.

On the other hand, Kafka represents each topic by at least one directory and several files per partition. The more topics/partitions Kafka has, the more files it creates. This makes it harder to buffer disk operations, perform sequential I/O, and it increases the complexity of what Zookeeper must manage.

# Other Advantages for MapR Streams 

Aside from performance, MapR Streams addresses a lot of other shortcomings with Kafka. I'll mention three, relating to replication, scaling, and mirroring:
	
- Kafka reaquires a lot of manual effort to replicate across clusters.

- Kakfa requires partitions to fit within the disk space of a single cluster node and cannot be split across machines. This is especially risky because Zookeeper could automatically assign multiple large partitions to a node that doesn't have space for them. You can move them manually, but that can become unmanagable.

- Kafka's mirroring design simply forwards messages to a mirror cluster. The offsets in the source cluster are useless in the mirror, which means consumers and producers cannot fail over from one cluster to a mirror.

# Conclusion

MapR Streams outperforms Kafka in big ways. Although performance isn't the only thing that makes MapR Streams desireable over Kafka for enterprise-grade streaming, it's a big one.

I measured throughput in a variety of cases that focused on the effects of message size and topic quantity and I saw MapR Streams transport a much faster stream of data, with much larger message sizes, and to far more topics than what we could be achieved with vanilla Kafka on a single cluster.




