Last week MapR released a new version of their Converged Data Platform. Today I installed it on Azure, and kept notes on all the commands I used. It's possible to automate this installation, and I'm pretty sure MapR has documented how to do that, but until I find that doc, here are the commands I use. This is what I like to call, "Automation for Dummy's", meaning, you can just copy and paste (some but not all of) these commands.  Needless to say, you should know what these commands do before you blindly copy and paste.


{% highlight bash %}
####################################
# Setup preliminary group assets
####################################
azure login
azure config mode arm
azure group create --location westus --name hadoopsummit

azure network nsg create --resource-group hadoopsummit --location westus --name hadoopsummit
azure network nsg rule create --resource-group hadoopsummit --nsg-name hadoopsummit --name AllowAll-from_me --priority 100 --source-address-prefix `curl ifconfig.co`/32 --destination-port-range 0-65535
	
azure storage account create --resource-group hadoopsummit --location westus --sku-name LRS --kind Storage hadoopsummit
azure network vnet create --resource-group hadoopsummit --location westus --name hadoopsummit-vnet
azure network vnet subnet create --resource-group hadoopsummit --vnet-name hadoopsummit-vnet --address-prefix 10.1.1.0/24 --name hadoopsummit-subnet

####################################
# Provision the VMs
####################################
for NODENAME in nodea nodeb nodec; do
azure network public-ip create --resource-group hadoopsummit --location westus --domain-name-label $NODENAME --name $NODENAME-publicip
azure network nic create --resource-group hadoopsummit --location westus --subnet-vnet-name hadoopsummit-vnet --subnet-name hadoopsummit-subnet --public-ip-name $NODENAME-publicip --network-security-group-name hadoopsummit --name $NODENAME-nic 
azure vm create --resource-group hadoopsummit --location westus --os-type linux --nic-name $NODENAME-nic --vnet-name hadoosummit-vnet  --vnet-subnet-name hadoopsummit-subnet --storage-account-name hadoopsummit --image-urn canonical:UbuntuServer:14.04.4-LTS:latest --vm-size Standard_DS11 --ssh-publickey-file ~/.ssh/id_rsa-azure.pub --admin-username mapr --name $NODENAME
azure vm disk attach-new --resource-group HADOOPSUMMIT --vm-name $NODENAME 100
done



####################################################
# When you're done with the VMs, cleanup like this.
####################################################
azure network vnet delete --resource-group hadoopsummit --name hadoopsummit-vnet
azure network vnet subnet delete --resource-group hadoopsummit --vnet-name hadoopsummit-vnet --name hadoopsummit-subnet -q
azure vm delete --resource-group HADOOPSUMMIT --name nodea -q
azure vm delete --resource-group HADOOPSUMMIT --name nodeb -q
azure vm delete --resource-group HADOOPSUMMIT --name nodec -q
azure network nic delete --resource-group HADOOPSUMMIT --name nodea-nic -q
azure network nic delete --resource-group HADOOPSUMMIT --name nodeb-nic -q
azure network nic delete --resource-group HADOOPSUMMIT --name nodec-nic -q
azure network public-ip delete --resource-group HADOOPSUMMIT --name nodea-publicip -q
azure network public-ip delete --resource-group HADOOPSUMMIT --name nodeb-publicip -q
azure network public-ip delete --resource-group HADOOPSUMMIT --name nodec-publicip -q


#########################
# Install the Oracle JDK
#########################
for NODENAME in nodea nodeb nodec; do
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no mapr@$NODENAME.westus.cloudapp.azure.com sudo apt-get install python-software-properties
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no mapr@$NODENAME.westus.cloudapp.azure.com sudo add-apt-repository ppa:webupd8team/java
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no mapr@$NODENAME.westus.cloudapp.azure.com sudo apt-get update
done
# this command won't work in the for loop
echo -e "Y\ny\n" | ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no mapr@nodea.westus.cloudapp.azure.com sudo apt-get install oracle-java8-installer
echo -e "Y\ny\n" | ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no mapr@nodeb.westus.cloudapp.azure.com sudo apt-get install oracle-java8-installer
echo -e "Y\ny\n" | ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no mapr@nodec.westus.cloudapp.azure.com sudo apt-get install oracle-java8-installer


# (optional) On each azure node, make sure swap space is enabled
for NODENAME in nodea nodeb nodec; do
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no mapr@$NODENAME.westus.cloudapp.azure.com sudo sed -i 's/ResourceDisk.EnableSwap=n/ResourceDisk.EnableSwap=y/g' /etc/waagent.conf
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no mapr@$NODENAME.westus.cloudapp.azure.com sudo sed -i 's/ResourceDisk.SwapSizeMB=0/ResourceDisk.SwapSizeMB=5120/g' /etc/waagent.conf
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no mapr@$NODENAME.westus.cloudapp.azure.com sudo service walinuxagent restart
done


# make sure you can ssh from one node to another (manually). If not, you're likely to get an error during the webui installer.

for NODENAME in nodea nodeb nodec; do
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no mapr@$NODENAME.westus.cloudapp.azure.com "sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd_config"
scp -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no ~/.ssh/id_rsa-azure.pub mapr@$NODENAME.westus.cloudapp.azure.com:~/.ssh
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no mapr@$NODENAME.westus.cloudapp.azure.com "cat ~/.ssh/id_rsa-azure.pub >> ~/.ssh/authorized_keys"
ssh -i ~/.ssh/id_rsa-azure -oStrictHostKeyChecking=no mapr@iannodec.westus.cloudapp.azure.com "sudo service ssh restart"
done

########################
# Setup swap space 1.4GB
########################
SSH onto each node and run these commands:
sudo swapon -s
sudo dd if=/dev/zero of=/mnt/swapfile bs=1400M count=1
sudo chmod 600 /mnt/swapfile
sudo mkswap /mnt/swapfile
sudo swapon /mnt/swapfile
sudo swapon -s
free -m
echo "/mnt/swapfile   none    swap    sw    0   0" >> /etc/fstab

# Set the mapr user password on each node
passwd mapr

# You must run mapr-setup.sh manually, so ssh login to nodea and do this:
wget http://package.mapr.com/releases/installer/mapr-setup.sh -P /tmp
chmod 700 /tmp/mapr-setup.sh
sudo su

bash /tmp/mapr-setup.sh

{% endhighlight %}

After the mapr-setup.sh script ends, it should advise you to open a URL, like https://nodea.westus.cloudapp.azure.com:9443 and login with mapr.

In the webui installer, specify the internal IPs for the node names, and specify /dev/sdc for the target installation disk.

The webui installer will run for about 30 minutes before it completes.


If you like saving money, you'll probably want to only run these cluster machines when you actually need them. I usually power off my VMs at the end of my work day.  Here's a useful one-liner to start and stop a series of VMs:

{% highlight bash %}
azure login
for NODENAME in nodea nodeb nodec; do azure vm start --resource-group iansandbox $NODENAME & done
{% endhighlight %}

