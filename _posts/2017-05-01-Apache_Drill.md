---
layout: post
title: How to combine relational and NoSQL datasets with Apache Drill
tags: [data discovery, data integration, apache drill]
---

It is rarely the case that enterprise data science applications can operate on data which is entirely contained within a single database system. Take for instance a company which wants to build a Customer 360 application that uses data sources across its enterprise to develop marketing campaigns or recommendation engines that more accurately target the concerns of key customer groups. To develop this kind of application you would be faced with a set of heterogeneous datasets such as CRM data, customer support data, and payment records that would need to be migrated and transformed into a common format simply to perform analysis. This can be challenging when your datasets are large or their schemas change, or they're stored as schemaless CSV or JSON files. 

The solution to this challenge is to use a database technology such as Apache Drill that can quickly process SQL queries across disparate datasets without moving or transforming those datasets. In this article I'm going to illustrate how Apache Drill can be used to query heterogeneous datasets easily and quickly.

## What is Apache Drill?

<img src="http://iandow.github.io/img/apache-drill-logo-400_0.png" width="33%" align="right">

[Apache Drill](https://drill.apache.org/) is a Unix service that unifies access to data across a variety of data formats and sources. It can run on a single node or on multiple nodes in a clustered environment. It supports a variety of NoSQL databases and file systems, including HBase, MongoDB, HDFS, Amazon S3, Azure Blob Storage, and local text files like json or csv files. Drill provides the following user friendly features:

* Drill supports industry standard APIs such as ANSI SQL and JDBC
* Drill enables you to perform queries without predefining schemas
* Drill enables you to JOIN data in different formats from multiple datastores.

## Connecting Drill to MySQL

Drill is primarily designed to work with nonrelational datasets, but it can also work with any relational datastore through a JDBC driver. To setup a JDBC connection follow these three steps:

1. Install Drill, if you do not already have it installed.
2. Copy your database's JDBC driver into the jars/3rdparty directory. (You'll need to do this on every node.)
3. Add a new storage configuration to Drill through the web ui. 

I'm using a 3 node MapR cluster for my development environment, so I already have Drill installed on each node. Drill is included in the standard MapR distribution, although MapR does not officially support the RDBMS plugin.

The MySQL JDBC connector can be downloaded from [https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.41.tar.gz](https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-5.1.41.tar.gz). So I copied that to `/opt/mapr/drill/drill-1.9.0/jars/3rdparty/` on each of my nodes.

I defined the MySQL storage configuration from the Drill web ui, as show below:

![mysql storage plugin](http://iandow.github.io/img/mysql-storage-plugin.png)

If you see the error "Please retry: error (unable to create/ update storage)" when you try to save the Storage Plugin then you've probably specified an invalid URL for the MySQL service or the credentials are invalid, or something else is preventing the Drill Unix service from connecting to your MySQL service.

After you click the button to enable that plugin, then you should see your MySQL tables by running `show schemas` in Drill, as shown below:

	mapr@nodec:~$ /opt/mapr/drill/drill-*/bin/sqlline -u jdbc:drill:
	apache drill 1.9.0
	0: jdbc:drill:> show schemas;
	+------------------------------+
	|         SCHEMA_NAME          |
	+------------------------------+
	| INFORMATION_SCHEMA           |
	| cp.default                   |
	| dfs.default                  |
	| dfs.root                     |
	| dfs.tmp                      |
	| hbase                        |
	| ianmysql.cars                |
	| ianmysql.information_schema  |
	| ianmysql.mysql               |
	| ianmysql.performance_schema  |
	| ianmysql                     |
	| sys                          |
	+------------------------------+
	12 rows selected (1.195 seconds)



Here are a couple examples showing how to join JSON files with MySQL tables using a SQL JOIN in Drill:

{% highlight sql %}
SELECT tbl1.name, tbl2.address FROM `dfs.tmp`.`./names.json` as tbl1 JOIN `dfs.tmp`.`./addressunitedstates.json` as tbl2 ON tbl1.id=tbl2.id;
{% endhighlight %}
{% highlight sql %}
SELECT tbl1.name, tbl2.address, tbl3.name FROM `dfs.tmp`.`./names.json` as tbl1 JOIN `dfs.tmp`.`./addressunitedstates.json` as tbl2 ON tbl1.id=tbl2.id JOIN ianmysql.cars.`car` as tbl3 ON tbl1.id=tbl3.customerid
{% endhighlight %}

## Connecting to Drill from Ubuntu

In order to use Drill from Python, R, or any other programming language you have to install an ODBC driver. The official [Apache Drill docs](https://drill.apache.org/docs/installing-the-driver-on-linux/) describe how to install on CentOS or Red Hat Linux but they do not cover Ubuntu, so I will. Here's how to install the MapR ODBC driver on Ubuntu 14.04:

First download and install the latest MapR ODBC rpm, like this:

{% highlight bash %}
wget http://package.mapr.com/tools/MapR-ODBC/MapR_Drill/MapRDrill_odbc_v1.3.0.1009/maprdrill-1.3.0.1009-1.x86_64.rpm
rpm2cpio maprdrill-1.3.0.1009-1.x86_64.rpm | cpio -idmv
sudo mv opt/mapr/drill /opt/mapr/drillodbc
cd /opt/mapr/drillodbc/Setup
cp mapr.drillodbc.ini ~/.mapr.drillodbc.ini
cp odbc.ini ~/.odbc.ini
cp odbcinst.ini ~/.odbcinst.ini
# Edit the properties in those ini files according to your needs
# Put the following exports also in ~/.bashrc
export ODBCINI=~/.odbc.ini
export MAPRDRILLINI=~/.mapr.drillodbc.ini
export LD_LIBRARY_PATH=/usr/local/lib:/opt/mapr/drillodbc/lib/64:/usr/lib64
{% endhighlight %}

Then update those .ini files, accordingly. Here is how I setup my ini files to connect to Drill:

### odbc.ini:

	[ODBC]
	Trace=yes
	Tracefile=/tmp/trace.txt

	[ODBC Data Sources]
	MapR Drill 64-bit=MapR Drill ODBC Driver 64-bit

	[drill64]
	# This key is not necessary and is only to give a description of the data source.
	Description=MapR Drill ODBC Driver (64-bit) DSN

	# Driver: The location where the ODBC driver is installed to.
	Driver=/opt/mapr/drillodbc/lib/64/libdrillodbc_sb64.so

	# The DriverUnicodeEncoding setting is only used for SimbaDM
	# When set to 1, SimbaDM runs in UTF-16 mode.
	# When set to 2, SimbaDM runs in UTF-8 mode.
	#DriverUnicodeEncoding=2

	# Values for ConnectionType, AdvancedProperties, Catalog, Schema should be set here.
	# If ConnectionType is Direct, include Host and Port. If ConnectionType is ZooKeeper, include ZKQuorum and ZKClusterID
	# They can also be specified on the connection string.
	# AuthenticationType: No authentication; Basic Authentication
	ConnectionType=Direct
	HOST=nodea
	PORT=31010
	ZKQuorum=[Zookeeper Quorum]
	ZKClusterID=[Cluster ID]
	AuthenticationType=No Authentication
	UID=[USERNAME]
	PWD=[PASSWORD]
	DelegationUID=
	AdvancedProperties=CastAnyToVarchar=true;HandshakeTimeout=5;QueryTimeout=180;TimestampTZDisplayTimezone=utc;ExcludedSchemas=sys,INFORMATION_SCHEMA;NumberOfPrefetchBuffers=5;
	Catalog=DRILL
	Schema=
	
### odbcinst.ini:
	
	[ODBC Drivers]
	MapR Drill ODBC Driver 32-bit=Installed
	MapR Drill ODBC Driver 64-bit=Installed

	[MapR Drill ODBC Driver 32-bit]
	Description=MapR Drill ODBC Driver(32-bit)
	Driver=/opt/mapr/drillodbc/lib/32/libdrillodbc_sb32.so

	[MapR Drill ODBC Driver 64-bit]
	Description=MapR Drill ODBC Driver(64-bit)
	Driver=/opt/mapr/drillodbc/lib/64/libdrillodbc_sb64.so

### mapr.drillodbc.ini
	
	[Driver]
	DisableAsync=0
	DriverManagerEncoding=UTF-16
	ErrorMessagesPath=/opt/mapr/drillodbc/ErrorMessages
	LogLevel=0
	LogPath=[LogPath]
	SwapFilePath=/tmp
	ODBCInstLib=/usr/lib/x86_64-linux-gnu/libodbcinst.so.1.0.0

Finally, if you've installed and configured the ODBC driver correctly, then the command 

{% highlight bash %}
python -c 'import pyodbc; print(pyodbc.dataSources()); print(pyodbc.connect("DSN=drill64", autocommit=True))'
{% endhighlight %}

should output information like this:

	```{'ODBC': '', 'drill64': '/opt/mapr/drillodbc/lib/64/libdrillodbc_sb64.so'}
	<pyodbc.Connection object at 0x7f5ec4a20200>```

## Connecting to Drill from Mac OS

[Drill Explorer](https://drill.apache.org/docs/drill-explorer-introduction/) is desktop GUI for Linux, Mac OS, and Windows that's useful for browsing data sources and previewing the results of SQL queries. People commonly use it to familiarize themselves with data sources and prototype SQL queries, then use another tool for actually analyzing that data in production. 

Download and install Drill Explorer and the iODBC driver for Mac OS from [here](https://drill.apache.org/docs/installing-the-driver-on-mac-os-x). Follow these instructions to setup the ODBC driver:

* [https://drill.apache.org/docs/installing-the-driver-on-mac-os-x/]https://drill.apache.org/docs/installing-the-driver-on-mac-os-x/
* [https://drill.apache.org/docs/configuring-odbc-on-mac-os-x/](https://drill.apache.org/docs/configuring-odbc-on-mac-os-x/)

Now you should have applications available in /Applications/iODBC/ and /Applications/DrillExplorer.app/. Before you can connect Drill Explorer to Drill, first create a new connection from the iODBC Data Source Administrator.  Instructions for configuring ODBC connections are at [https://drill.apache.org/docs/testing-the-odbc-connection](https://drill.apache.org/docs/testing-the-odbc-connection). Here's what my configuration looks like:

<img src="http://iandow.github.io/img/iodbc_admin.png" width="33%">
<img src="http://iandow.github.io/img/iodbc_admin_setup.png" width="33%">

Once you connect Drill Explorer using the iODBC configuration you created, you should see all the available data sources, and preview the results of SQL queries, as shown below:

<img src="http://iandow.github.io/img/drillexplorer.png" width="66%" >

## Connecting to Drill from Jupyter

You can also use the ODBC connection you configured above to programmatically query Drill data sources. MapR blogged about [how to use Drill from Python, R, and Perl](https://mapr.com/blog/using-drill-programmatically-python-r-and-perl/). I used those instructions to setup an ODBC connection in a Jupyter notebook for data exploration with Python, as shown below:

{% include drill_demo.html %}

## Connecting to Drill from R Studio

You can also connect to Drill from R Studio using the ODBC connection you configured above. Here's how I set that up:

1. Download and install the RODBC driver:

	{% highlight sql %}
	brew update
	brew install unixODBC
	wget "https://cran.r-project.org/src/contrib/RODBC_1.3-13.tar.gz"
	R CMD INSTALL RODBC_1.3-13.tar.gz
	{% endhighlight %}

	Reference: [http://stackoverflow.com/questions/28081640/installation-of-rodbc-on-os-x-yosemite](http://stackoverflow.com/questions/28081640/installation-of-rodbc-on-os-x-yosemite)

2. Set your LD_LIBRARY_PATH:

	{% highlight sql %}
	export LD_LIBRARY_PATH=/usr/local/iODBC/lib/
	{% endhighlight %}

3. Open R Studio and submit a SQL query

	{% highlight r %}
	library(RODBC)
	ch<-odbcConnect("Sample MapR Drill DSN")
	sqlQuery(ch, 'use dfs.tmp')
	sqlQuery(ch, 'show files')
	{% endhighlight %}

![R Studio](http://iandow.github.io/img/rstudio_drill.png)

For more information on using Drill with R, check out the following references:

* [http://stackoverflow.com/questions/28081640/installation-of-rodbc-on-os-x-yosemite](http://stackoverflow.com/questions/28081640/installation-of-rodbc-on-os-x-yosemite)
* [https://community.mapr.com/thread/9942](https://community.mapr.com/thread/9942)

# Conclusion

What I've tried to show above is that it doesn't matter how your data is formatted or where its stored, Apache Drill enables you to easily query and combine datasets from a wide variety of databases and file systems. This makes Apache Drill an extremely valuable tool for applications such as Customer 360 applications that require access to disparate datasets and the ability to combine those datasets with low latency SQL JOINs.

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