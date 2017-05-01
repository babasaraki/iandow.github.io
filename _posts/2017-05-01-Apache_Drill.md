---
layout: post
title: How to combine relational and NoSQL datasets with Apache Drill
tags: [data discovery, data integration, apache drill]
---

It is rarely the case that enterprise data science applications can operate on data which is entirely contained within a single database system. Take for instance a company which wants to use data sources across its enterprise to develop marketing campaigns or recommendation engines that more accurately target the concerns of key customer groups. To develop this kind of application you would be faced with a set of heterogeneous datasets such as CRM data, customer support data, and payment records that would need to be migrated and transformed into a common format simply to perform analysis. This can be challenging when your datasets are large and the schemas change. The solution to this challenge is to use a database technology, such as Apache Drill, that can quickly provide results to queries without data movement or transformation. In this article I'm going to illustrate how Apache Drill can be used to query heterogeneous datasets easily and quickly. 

## What is Apache Drill?

Apache Drill is a Unix service that unifies access to data across a variety of data formats and sources. It can run on a single node or on multiple nodes in a clustered environment. It supports a variety of NoSQL databases and file systems, including HBase, MongoDB, HDFS, Amazon S3, Azure Blob Storage, and local text files like json or csv files. Drill provides the following user friendly features:
* Drill supports industry standard APIs such as ANSI SQL and JDBC
* Drill enables you to perform queries without predefining schemas
* Drill enables you to JOIN data in different formats from multiple datastores.

## Using the RDBMS Storage Plugin to access MySQL

Drill is designed to work with any relational datastore that provides a JDBC driver. Drill is actively tested with Postgres, MySQL, Oracle, MSSQL and Apache Derby. To setup a JDBC connector follow these three steps:

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
SELECT tbl1.name, tbl2.address, tbl3.name FROM `dfs.tmp`.`./names.json` as tbl1 JOIN `dfs.tmp`.`./addressunitedstates.json` as tbl2 ON tbl1.id=tbl2.id JOIN ianmysql.cars.`car` as tbl3 ON tbl1.id=tbl3.customerid
{% endhighlight %}

# Conclusion

I hope it's now apparent that it doesn't matter how your data is formatted or where its stored, Apache Drill enables you to explore datasets easily and quickly in standard SQL. 

