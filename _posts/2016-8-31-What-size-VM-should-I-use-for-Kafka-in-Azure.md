I've been using Azure for hosting a 3 node MapR cluster with which I'm running a streaming application that uses Kafka, Spark Streaming, and Spark SQL APIs to process a fast data stream. My use-case requires that I be able to ingest 1.7GB of data into Kafka within 1 minute (apx 227Mbps). Since this is a relatively high I/O workload, I know I need my cluster nodes to run on Azure's premium storage tier, which uses high-performance solid-state drives (SSDs) for virtual machine disks, but within tier I wasn't sure which of the [Azure VM sizes](https://azure.microsoft.com/en-us/documentation/articles/virtual-machines-windows-sizes/) would be optimal (i.e. cheapest) for my application. So I ran a simple experiment. 

Starting with the smallest/cheapest VM sizein the DS-series, I used a simple Kafka stream producer to publish data from a 1.7GB file residing on local storage, until that dataset could be publishing withing 60 seconds. 

The results of my experiement are shown in the following table.

| VM Size       | CPU | Memory | Kafka Ingest Time | Kafka Ingest Rate |
| ------------- |-------------|-----------|
| Standard_DS4  | 8 cores | 28 GB | 1m35.806s  | 17.7 Mbps |
| Standard_DS11 | 2 cores | 14 GB | 2m3.918s   | 13.7 Mbps |
| Standard_DS12 | 2 cores | 14 GB | 2m3.918s   | 13.7 Mbps |
| Standard_DS13 | 2 cores | 14 GB | 2m3.918s   | 13.7 Mbps |
| Standard_DS14 | 2 cores | 14 GB | 2m3.918s   | 13.7 Mbps |
| ???           | tbd     | tbd   | 1m0.000s   | 28.3 Mbps | <-- GOAL !!!

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