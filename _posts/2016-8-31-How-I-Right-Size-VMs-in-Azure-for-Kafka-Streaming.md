---
layout: post
title: How to "Right Size" VMs in Azure for Kafka Streaming
tags: [azure, kafka]
---

I've been using Azure for hosting a 3 node MapR cluster with which I'm running a streaming application that uses Kafka and Spark to process a fast data stream. My use case requires that I be able to ingest 1.7 GB of data into Kafka within 1 minute (approximately 227 Mbps). Since this is a relatively high I/O workload, I know I need my cluster nodes to run on Azure's premium storage tier which uses high-performance solid-state drives (SSDs) for virtual machine disks, but within that tier I wasn't sure which of the [Azure VM sizes](https://azure.microsoft.com/en-us/documentation/articles/virtual-machines-windows-sizes/) would be optimal (i.e. cheapest) for my application. So I ran a simple experiment. 

![Azure VM Sizes](http://iandow.github.io/img/Azure%20VM%20sizes.png)

Starting with the smallest/cheapest VM size in the DS-series, I timed how long it took a simple Kafka stream producer to publish data in a 1.7 GB file residing on local storage to a single topic, and increased the VM sizes until the producer could complete within 60 seconds.

The results of my study are shown in the following table:

| VM Size       | Price   |CPU      | Memory | Kafka Ingest Time | Kafka Ingest Rate |
| ------------- |---------|----------|--------|-------------------|-------------------|
| GOAL          |         |          |        | 60 s              | 227 Mbps          |
| Standard_DS11 | $145/mo | 2 cores  | 14 GB  | 124 s             | 110 Mbps          |
| Standard_DS12 | $290/mo | 4 cores  | 28 GB  | 64 s              | 213 Mbps          |
| Standard_DS4  | $458/mo | 8 cores  | 28 GB  | 96 s              | 142 Mbps          |
| Standard_DS13 | $580/mo | 8 cores  | 56 GB  | 84 s              | 162 Mbps          |
| Standard_DS14 | $1147/mo| 16 cores | 112 GB | 77 s              | 177 Mbps          |


Here's the one-liner I used for resizing my cluster:

{% highlight bash %}
azure login
for NODENAME in nodea nodeb nodec; do azure vm set --resource-group iansandbox --vm-size Standard_DS4 --name $NODENAME & done
{% endhighlight %}

Azure can typically take between 3 and 5 minutes to resize a VM. So, it's useful to resize VMs in parallel, like the above command does.


## How do we size unused cluster nodes?

If I run all my Kafka consumers and producers on only one node in a multi-node cluster, does it matter what compute power resides on my other nodes?

For example, if I'm producing data from local file on "nodea" with a process running on nodea, and consuming said messages from Kafka processes also running on nodea, does it matter what size the other two nodes in my cluster are?

The answer is, no. It doesn't matter. If your processing is limited to one node, then the compute power of other nodes won't affect how fast your workloads run. The factors in play are compute power for the node where you're running Kafka consumers and producer, and the I/O bandwidth of cluster storage.


# Conclusion:

This unscientific study was a quick and dirty attempt to determine which Azure VM size would be the cheapest configuration for my Kafka streaming workload. There are a lot of ways to tune Kafka applications to get the most out of your hardware and there are more precise ways to measure performance than what I did but my results allowed me to make an educated guess on which VM sizes would be a good starting point for my application.