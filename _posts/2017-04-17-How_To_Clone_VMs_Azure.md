---
layout: post
title: How To Clone Virtual Machines in Azure
tags: [azure]
bigimg: /img/dna-1811955_1280.jpg
---

I use Azure a lot to create virtual machines for demos and application prototypes. It often takes me a long time to setup these rigs, so once I finally get things the way I like them I really don't want to duplicate that effort. Fortunately, Azure lets us clone VMs. Unfortunately, it's not that easy to figure out if you've never done it before. In my case, it's even harder because the applications I build are designed to run on Hadoop clusters. At a minimum I need to provision 3 VMs (that's a 3-node cluster) with tens of GBs of data in secondary disk storage, and more hostname based service configurations than you can shake a stick at. To set this up every time I need to present a demo would be completely unsustainable. It's FAR easier to setup a single demo rig then clone it as needed. So, that's what I'm going to describe how to do here. 

## Here's how I clone a 3-node cluster:

Once you have a rig that you want to clone, your first step is to deallocate those VMs, then generalize and capture their disk images. Here's how I typically do that for a three node cluster:

{% highlight bash %}
RG1=iansandbox3
for NODENAME in nodea nodeb nodec; do azure vm deallocate --resource-group $RG1 --name $NODENAME & done
for NODENAME in nodea nodeb nodec; do azure vm generalize --resource-group $RG1 --name $NODENAME & done
for NODENAME in nodea nodeb nodec; do azure vm capture --resource-group $RG1 --name $NODENAME --vhd-name-prefix $NODENAME --template-file-name $NODENAME.json & done
{% endhighlight %}

If you're going to clone your VMs to a new resource group, then you'll need to copy said disk images to a new storage account in that new resource group. Here is how I copy both the data and OS disk images from one storage account to another using the Azure CLI:

{% highlight bash %}
RG1=iansandbox3
STORAGE1=iansandbox3
CONTAINER1=system
STORAGEKEY1=`azure storage account keys list -g $RG1 $STORAGE1 | grep key1 | awk '{print $3}'`
RG2=iansandbox4
STORAGE2=iansandbox4
STORAGEKEY2=`azure storage account keys list -g $RG2 $STORAGE2 | grep key1 | awk '{print $3}'`
CONTAINER2=baseimages

# Create the destination storage container
azure storage container create --container $CONTAINER2 --account-name $STORAGE2 --account-key $STORAGEKEY2

# First list the file:
azure storage blob list --container $CONTAINER1 --account-name $STORAGE1 --account-key $STORAGEKEY1
azure storage blob show --container $CONTAINER1 --account-name $STORAGE1 --account-key $STORAGEKEY1 --blob Microsoft.Compute/Images/vhds/nodea-dataDisk-0.49735bce-3c31-46ed-b5fa-8be620fb23fd.vhd
    
# Then copy it (I'm copying both a data disk and an os disk here):
azure storage blob copy start --account-name $STORAGE1 --account-key $STORAGEKEY1 --source-container $CONTAINER1 --source-blob Microsoft.Compute/Images/vhds/nodea-dataDisk-0.49735bce-3c31-46ed-b5fa-8be620fb23fd.vhd --dest-account-name $STORAGE2 --dest-account-key $STORAGEKEY2 --dest-container $CONTAINER2

# Then check the status of the copy like this:
azure storage blob copy show --account-name $STORAGE1 --account-key $STORAGEKEY1 $CONTAINER1 Microsoft.Compute/Images/vhds/nodea-dataDisk-0.49735bce-3c31-46ed-b5fa-8be620fb23fd.vhd
{% endhighlight %}

Even if the status in that last step says "success", indicating that all the bytes have been copied, the image may not be ready for another several minutes. You'll know you tried to provision too soon if provisioning creates the VM, but when you navigate to the VM in the azure portal it says provisioning failed because the disk images were still in "Pending" status.  This is clearly a bug in the Azure CLI which I'll talk more about in the next section.

# Troubleshooting

One of the most common problems I have when cloning a VM happens when I try to provision a VM before the disk images have been copied to my destination storage account.  I typically save a master copy of disk images for my VMs in a storage account that contains nothing but those disk images. That way, when I need a new demo rig I just create a new storage account (usually in a new resource group) and copy those baseline disk images to that new group with a command like this:

{% highlight bash %}
azure storage blob copy start [options] [sourceUri] [destContainer]
{% endhighlight %}

Obviously, you can't use those disk images until the copy operations have completed. However, if you check the status of a blob copy like this:

{% highlight bash %}
azure storage blob copy show [options] ...
{% endhighlight %}

you should see whether the copy is "Pending" or "Successful". In my experience, that command always just says “Successful”. I know this is erroneous because if I try to create a VM from my new vhd images it fails with a DiskImageNotReady error because the copy is in fact, still "Pending".  If I continue to wait from minutes to hours, depending on how far the distance between my source and destination storage account, then the copy will finally finish and VM provisioning will succeed. Let me show you what this looks like:

You can start copying a disk image with a command like this:

{% highlight bash %}
azure storage blob copy start --account-name $STORAGE1 --account-key $STORAGEKEY1 --source-container $CONTAINER1 --source-blob Microsoft.Compute/Images/vhds/nodec-dataDisk-0.9588f46c-9198-47f4-ac9d-8c6d71e31470.vhd --dest-account-name $STORAGE2 --dest-account-key $STORAGEKEY2 --dest-container $CONTAINER2
{% endhighlight %}

That copy operation is asynchronous. You can check its status with a command like this:

    azure storage blob copy show --account-name $STORAGE1 --account-key $STORAGEKEY1 $CONTAINER1 Microsoft.Compute/Images/vhds/nodec-dataDisk-0.9588f46c-9198-47f4-ac9d-8c6d71e31470.vhd
    info:    Executing command storage blob copy show
    + Getting storage blob information
    data:    Copy ID                               Progress                     Status
    data:    ------------------------------------  ---------------------------  -------
    data:    67193eed-02fd-45b2-8693-efc8ad140526  1073741824512/1073741824512  success
    info:    storage blob copy show command OK

The progress there is clearly wrong. It did not copy 1073741824512 out of 1073741824512 bytes in only a few seconds. This is a bug with version 1 of the Azure CLI. Fortunately, the bug has been fixed with version 2, as I describe below.

## How else can you see blob copy status?

How else can you see blob copy status? I'll show you two ways.  

### Use the latest version of the Azure CLI.

I highly recommend moving to CLI version 2 which was re-written in Python. The problem I mentioned has been fixed in that version. You can download it from [https://docs.microsoft.com/en-us/cli/azure/install-azure-cli](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli).

Here's how to check the status of a blob copy with Azure CLI version 2

{% highlight bash %}
az storage blob show -c baseimages --account-name $STORAGE2 --account-key $STORAGEKEY2 -n iantest
{% endhighlight %}

Gives this output:

{% highlight json %}
"copy": {
      "completionTime": null,
      "id": "2e067681-f835-48db-9444-5d844fabeb9b",
      "progress": "55729283072/1073741824512",
      "source": "https://smolskidemo.blob.core.windows.net/system/Microsoft.Compute/Images/vhds/nodec-dataDisk-0.9588f46c-9198-47f4-ac9d-8c6d71e31470.vhd?sr=b&spr=https&sp=r&sv=2016-05-31&sig=CqgOO6WetnEefBXY/UkqYltZJHVDAGKoXIMoxRJLYf4%3D&se=2017-04-05T16%3A56%3A51Z",
      "status": "pending",
      "statusDescription": null
    },
{% endhighlight %}

See that progress status?  It says 55729283072 out of 1073741824512?  That's what you should see. And you should continue to see that first number increasing.  So, don't use version 1 of the Azure CLI. Use version 2.

### Use the Azure REST API

The other option is to use the Azure REST API. Use the 
[Get-Blob-Properties](https://docs.microsoft.com/en-us/rest/api/storageservices/fileservices/Get-Blob-Properties) REST call and checking the `x-ms-copy-status` field.  If you've never used the Azure REST API, I recommend installing Postman and setting up a GET request as shown below. You'll need to determine your Bearer value in the authorization header, but I don't want to go into that right now, so just search for it.

{% highlight json %}
GET - https://iansandbox3.blob.core.windows.net/baseimages?comp=list&restype=container
Authorization header:
	Bearer xxxx[REDACTED]xxxx
Content-Type header:
	application/json
{% endhighlight %}


## Conclusion

In this post I described how to use the Azure CLI to clone VMs by generalizing and copying their disk images to new resource groups. Here are the *power moves* that advanced Azure users will do which most people overlook when trying to create and troubleshoot provisioning errors based on generalized disk images.

1. Use Azure CLI V2 (not V1)
2. Use Azure JSON templates to provision VMs and other infrastructure
3. Use Resource Explorer [http://resources.azure.com](http://resources.azure.com) for troubleshooting
4. Use the Azure REST API for troubleshooting

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