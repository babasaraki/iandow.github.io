---
layout: post
title: How to deploy a 3 node Kafka cluster in Azure
tags: [azure, kafka, cluster]
---

In an earlier post I described how to setup a single node Kafka cluster in Azure so that you can quickly familiarize yourself with basic Kafka operations. However, most real world Kafka applications will run on more than one node to take advantage of Kafka's replication features for fault tolerance. So here I'm going to provide a script you can use to deploy a multi-node Kafka cluster in Azure.

The following script will deploy a 3 node Kafka cluster in Azure.


{% highlight bash linenos %}
RESOURCE_GROUP=iansandbox
ADMIN_USER=mapr
# This next property tells bash to avoid putting the password declaration in its history
HISTCONTROL=ignoreboth
 ADMIN_PASSWORD='changeme!'
####################################
# Setup preliminary group assets
####################################
azure login
azure config mode arm
azure group create --location westus --name $RESOURCE_GROUP

azure network nsg create --resource-group $RESOURCE_GROUP --location westus --name $RESOURCE_GROUP
azure network nsg rule create --resource-group $RESOURCE_GROUP --nsg-name $RESOURCE_GROUP --name AllowAll-from_me --priority 100 --source-address-prefix `curl ifconfig.co`/32 --destination-port-range 0-65535
  
azure storage account create --resource-group $RESOURCE_GROUP --location westus --sku-name LRS --kind Storage $RESOURCE_GROUP
azure network vnet create --resource-group $RESOURCE_GROUP --location westus --name $RESOURCE_GROUP-vnet
azure network vnet subnet create --resource-group $RESOURCE_GROUP --vnet-name $RESOURCE_GROUP-vnet --address-prefix 10.1.1.0/24 --name $RESOURCE_GROUP-subnet

####################################
# Provision the VMs
####################################
for NODENAME in nodea nodeb nodec; do
azure network public-ip create --resource-group $RESOURCE_GROUP --location westus --domain-name-label $NODENAME --name $NODENAME-publicip
azure network nic create --resource-group $RESOURCE_GROUP --location westus --subnet-vnet-name $RESOURCE_GROUP-vnet --subnet-name $RESOURCE_GROUP-subnet --public-ip-name $NODENAME-publicip --network-security-group-name $RESOURCE_GROUP --name $NODENAME-nic 
azure vm create --resource-group $RESOURCE_GROUP --location westus --os-type linux --nic-name $NODENAME-nic --vnet-name hadoosummit-vnet  --vnet-subnet-name $RESOURCE_GROUP-subnet --storage-account-name $RESOURCE_GROUP --image-urn canonical:UbuntuServer:14.04.4-LTS:latest --vm-size Standard_DS11 --ssh-publickey-file ~/.ssh/id_rsa-azure.pub --admin-username mapr --name $NODENAME
azure vm disk attach-new --resource-group $RESOURCE_GROUP --vm-name $NODENAME 1000
done

#########################
# Install the Oracle JDK
#########################
for NODENAME in nodea nodeb nodec; do
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no $ADMIN_USER@$NODENAME.westus.cloudapp.azure.com sudo apt-get install python-software-properties
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no $ADMIN_USER@$NODENAME.westus.cloudapp.azure.com sudo add-apt-repository ppa:webupd8team/java
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no $ADMIN_USER@$NODENAME.westus.cloudapp.azure.com sudo apt-get update
done

for NODENAME in nodea nodeb nodeb; do 
echo -e "Y\ny\n" | ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no $ADMIN_USER@$NODENAME.westus.cloudapp.azure.com sudo apt-get install oracle-java8-installer; 
# (Yes, you really do need to run this command twice.)
echo -e "Y\ny\n" | ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no $ADMIN_USER@$NODENAME.westus.cloudapp.azure.com sudo apt-get install oracle-java8-installer; 
done

################
# Install Kafka
################
for NODENAME in nodea nodeb nodeb; do 
ssh mapr@$NODENAME.westus.cloudapp.azure.com wget http://apache.cs.utah.edu/kafka/0.10.0.1/kafka_2.11-0.10.0.1.tgz
ssh mapr@$NODENAME.westus.cloudapp.azure.com tar -xzvf kafka_2.11-0.10.0.1.tgz
ssh mapr@$NODENAME.westus.cloudapp.azure.com sudo mv kafka_2.11-0.10.0.1 /opt
ssh mapr@$NODENAME.westus.cloudapp.azure.com sudo ln -s /opt/kafka_2.11-0.10.0.1 /opt/kafka
ssh mapr@$NODENAME.westus.cloudapp.azure.com "sudo wget https://gist.githubusercontent.com/iandow/9efb351a7d15592583508b1e5be55184/raw/066e97b4613a54af696bdb99eb2398b698f68582/kafka && sudo mv kafka /etc/init.d/kafka && chmod 755 /etc/init.d/kafka && update-rc.d kafka defaults
"
done

######################
# Update Kafka config
######################
let BROKER_ID=0
for NODENAME in nodea nodeb nodeb; do 
ssh mapr@$NODENAME.westus.cloudapp.azure.com sudo sed -i 's/zookeeper.connect=localhost:2181/zookeeper.connect=nodea:2181,nodeb:2181,nodec:2181/g' /opt/kafka/config/server.properties
ssh mapr@$NODENAME.westus.cloudapp.azure.com sudo sed -i 's/broker.id=0/broker.id=$BROKER_ID/g' /opt/kafka/config/server.properties
let BROKER_ID=$BROKER_ID+1
done

##########################
# Update Zookeeper config
##########################
for NODENAME in nodea nodeb nodeb; do 
ssh mapr@$NODENAME.westus.cloudapp.azure.com "sudo echo -e 'server.1=nodea:2888:3888
server.2=nodeb:2888:3888
server.3=nodec:2888:3888
initLimit=5
syncLimit=2' >> /opt/kafka/config/zookeeper.properties"
done

############################
# Start Zookeeper and Kafka
############################
for NODENAME in nodea nodeb nodeb; do 
ssh mapr@$NODENAME.westus.cloudapp.azure.com sudo service kafka start
done
{% endhighlight %}

Now lets validate that it's working, like this:

Create a topic.
    
    /opt/kafka/bin/kafka-topics.sh --create --zookeeper nodea:2181,nodeb:2181,nodec:2181 --replication-factor 3 --partitions 1 --topic kafkatest

This will create a kafkatest-0 folder in the Kafka logdir (e.g. /tmp/kafka-logs) on all the three machines.

Fire up a producer on one machine:

    ssh mapr@nodea.westus.cloudapp.azure.com sudo /opt/kafka/bin/kafka-console-producer.sh --broker-list nodea:9092,nodeb:9092,nodec:9092 --topic kafkatest

...and a consumer on another:

    ssh mapr@nodeb.westus.cloudapp.azure.com sudo /opt/kafka/bin/kafka-console-consumer.sh --zookeeper nodea:2181,nodeb:2181,nodec:2181 --topic kafkatest --from-beginning

On the producer console enter some text.

    this is a test
    bla bla _kafka_is_working_ bla bla
    
Now on the consumer you should immediately see the text you entered.
 
# Conclusion

We just built a three node Kafka cluster in Azure.  We created a topic that was replicated across all three nodes in our cluster and proved its functionality by sending and consuming messages from the topic. You could further demonstrate the fault tolerance of that replication by failing each (but not all) of the cluster nodes and observing that the consumer can still read all the messages in our topic.


