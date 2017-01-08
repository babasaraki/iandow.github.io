---
layout: post
title: What's wrong with using small batch sizes in Kafka?
tags: [java, kafka]
bigimg: /img/autos-technology-vw-multi-storey-car-park-63294.jpg
---

# What is Kafka's batch size?

Kafka producer clients will buffer unsent records for each partition. These buffers are of a size specified by the batch.size config. You can achieve higher throughput by increasing the batch size, but there is a tradeoff between more batching and increased end-to-end latency. The larger your batch size, the more cumulative time messages will spend waiting in the send buffer. So, you want to batch to increase throughput, but you don't want to batch too much lest you cause unwanted latency.

The effects of batching on throughput and latency are really well illustrated by this blog: [http://blog.l1x.me/post/2015/03/02/high-performance-kafka-for-analytics.html](http://blog.l1x.me/post/2015/03/02/high-performance-kafka-for-analytics.html), but that blog only looks at message sizes up to 1KB. 

# So, what's wrong with using small batch sizes?

Batching increases latency because the producer will delay sending a message until it fills its send buffer (or the linger.ms timer expires). However, larger messages seem to be disproporationately delayed  by small batch sizes.  In the following graph, I measured end-to-end latency for a wide range of message sizes using a batch size of 16KB.  The step up in latency is due to the batch size being too small.

![latency-batch-16kb](http://iandow.github.io/img/latency-batch-32kb.png)

When I increased batch size to 32KB, end-to-end latency was much improved, as shown below:

![latency-batch-16kb](http://iandow.github.io/img/latency-batch-32kb.png)

# Conclusion:

If you're sending large messages in Kafka, you might be surprised to find how much you can improve performance simply by increasing you producer batch size. 
