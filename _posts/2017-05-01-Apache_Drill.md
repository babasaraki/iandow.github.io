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
  <p class="margin-override font-override">
  	<img src="http://iandow.github.io/img/paypal.png" width="33%" align="right">
  	Did you learn something useful from this blog? Has it saved you time? If so, perhaps you would like to buy me a beer!</p>
  <div id="paypalbtn">
    <a class="btn btn-primary btn" href="https://www.paypal.me/iandownard/3.5">Donate via PayPal</a>
  </div>
</div>