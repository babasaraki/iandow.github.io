---
layout: post
title: How To Install Mapr In Azure
tags: [azure, mapr]
bigimg: /img/clouds.jpg
---

Last week MapR released a new version of their Converged Data Platform. Today I installed it on Azure, and kept notes on all the commands I used. It's possible to automate this installation, and I'm pretty sure MapR has documented how to do that, but until I find that doc, here are the commands I use. This is what I like to call, "Automation for Dummy's", meaning, you can just copy and paste (some but not all of) these commands.  Needless to say, you should know what these commands do before you blindly copy and paste.


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

#####################
# Add SSH public keys
#####################

for NODENAME in nodea nodeb nodec; do
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no $ADMIN_USER@$NODENAME.westus.cloudapp.azure.com "sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config"
scp -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no ~/.ssh/id_rsa-azure.pub $ADMIN_USER@$NODENAME.westus.cloudapp.azure.com:~/.ssh
scp -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no ~/.ssh/id_rsa-azure.pub $ADMIN_USER@$NODENAME.westus.cloudapp.azure.com:~/.ssh
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no $ADMIN_USER@$NODENAME.westus.cloudapp.azure.com "cat ~/.ssh/id_rsa-azure.pub >> ~/.ssh/authorized_keys"
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no $ADMIN_USER@$NODENAME.westus.cloudapp.azure.com "sudo service ssh restart"
done


##################################
# (optional) Configure swap space
##################################
for NODENAME in nodea nodeb nodec; do
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no $ADMIN_USER@$NODENAME.westus.cloudapp.azure.com sudo sed -i 's/ResourceDisk.EnableSwap=n/ResourceDisk.EnableSwap=y/g' /etc/waagent.conf
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no $ADMIN_USER@$NODENAME.westus.cloudapp.azure.com sudo sed -i 's/ResourceDisk.SwapSizeMB=0/ResourceDisk.SwapSizeMB=5120/g' /etc/waagent.conf
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no $ADMIN_USER@$NODENAME.westus.cloudapp.azure.com sudo service walinuxagent restart
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no $ADMIN_USER@$NODENAME.westus.cloudapp.azure.com "sudo swapon -s"
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no $ADMIN_USER@$NODENAME.westus.cloudapp.azure.com "sudo dd if=/dev/zero of=/mnt/swapfile bs=1400M count=1"
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no $ADMIN_USER@$NODENAME.westus.cloudapp.azure.com "sudo chmod 600 /mnt/swapfile"
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no $ADMIN_USER@$NODENAME.westus.cloudapp.azure.com "sudo mkswap /mnt/swapfile"
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no $ADMIN_USER@$NODENAME.westus.cloudapp.azure.com "sudo swapon /mnt/swapfile"
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no $ADMIN_USER@$NODENAME.westus.cloudapp.azure.com "sudo swapon -s"
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no $ADMIN_USER@$NODENAME.westus.cloudapp.azure.com "sudo free -m"
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no $ADMIN_USER@$NODENAME.westus.cloudapp.azure.com "sudo su -c \"echo '/mnt/swapfile   none    swap    sw    0   0' >> /etc/fstab\""
done

#############################
# Set the admin user password
#############################
for NODENAME in nodea nodeb nodec; do
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no $ADMIN_USER@$NODENAME.westus.cloudapp.azure.com "echo -e \"${ADMIN_PASSWORD}\n${ADMIN_PASSWORD}\" | sudo passwd mapr"; 
done
unset ADMIN_PASSWORD
{% endhighlight %}

Next, run the mapr-setup.sh utility on one of your nodes. *This must be run manually*, so ssh login to a cluster node and do this:

{% highlight bash %}
wget http://package.mapr.com/releases/installer/mapr-setup.sh -P /tmp
chmod 700 /tmp/mapr-setup.sh
echo -e "\n\n\n" | sudo bash /tmp/mapr-setup.sh
{% endhighlight %}

When the script prompts you for anything, just accept the defaults.

Once the mapr-setup.sh script ends, open https://nodea.westus.cloudapp.azure.com:9443 and login with the username you specified with `$ADMIN_USER`.

On the "Configure Nodes" page of the webui installer, specify the internal IPs for the node names, and specify /dev/sdc for the target installation disk. *DO NOT* use the internal DNS when you specify the node names. Use the internal IPs, instead. Azure generates really long internal DNS names that can cause the MapR installer to fail if you choose to use the use the optional MySQL service.

The webui installer will run for about 30 minutes before it completes.

If you like saving money, you'll probably want to only run these cluster machines when you actually need them. I usually power off my VMs at the end of my work day.  Here's a useful one-liner to start and stop a series of VMs:

{% highlight bash %}
azure login
for NODENAME in nodea nodeb nodec; do azure vm start --resource-group iansandbox $NODENAME & 
done
{% endhighlight %}



### When you're done with the VMs, cleanup like this:

{% highlight bash %}
azure network vnet delete --resource-group $RESOURCE_GROUP --name $RESOURCE_GROUP-vnet
azure network vnet subnet delete --resource-group $RESOURCE_GROUP --vnet-name $RESOURCE_GROUP-vnet --name $RESOURCE_GROUP-subnet -q
for NODENAME in nodea nodeb nodec; do
azure vm delete --resource-group $RESOURCE_GROUP --name $NODENAME -q
azure network nic delete --resource-group $RESOURCE_GROUP --name $NODENAME-nic -q
azure network public-ip delete --resource-group $RESOURCE_GROUP --name $NODENAME-publicip -q
done;
{% endhighlight %}

<br><br>
<div class="main-explain-area padding-override jumbotron">
  <img src="http://iandow.github.io/img/starbucks_coffee_cup.png" width="120" style="margin-left: 15px" align="right">
  <a href="https://www.paypal.me/iandownard" title="PayPal donation" target="_blank">
  <h1>Hope that Helped!</h1>
  <p class="margin-override font-override">
    If this post helped you out, please consider fueling future posts by buying me a cup of coffee!</p>
  </a>
  <br>
</div>