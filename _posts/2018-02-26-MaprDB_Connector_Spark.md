---
layout: post
title: The MapR-DB Connector for Apache Spark
tags: [nosql, json, spark]
---

MapR just released Python and Java support for their MapR-DB connector for Spark. It also supports Scala, but Python and Java are new.  I recorded a video to help them promote it, but I also learned a lot in the process, relating to how databases can be used in Spark. 

If you want to use a database to persist a Spark dataframe (or RDD, or Dataset), you need a piece of software the connects that databse to Spark.  [Here](https://docs.databricks.com/spark/latest/data-sources/index.html) is a list of database that can be connected to Spark. Without getting into the relative strengths of one database over another, there are a couple of capabilities you should look for when you're picking a database to use with Spark. These are characteristics of the database's Spark *connector*, not of the database:

1. Filter and Projection Pushdown.  I.e. When you select columns and use the SQL `where` clause to select rows in a table, those operations get executed on the database. All other SQL operators, like `order by` or `group by` are computed in the Spark executor.
2. Automatic Schema Inference
3. Support for RDDs, Dataframes, and Datasets
4. Bulk save

There are also those intangible nice-to-haves, like language support (python/java/scala), IDE support (intellij/netbeans/eclipse), and notebook support (jupyter/zeppelin). 


Here is a video demo that I recorded which talks about the MapR-DB connector for Spark:

<a href="https://youtu.be/9RHm1YJNaIc"><img src="http://iandow.github.io/img/ojai_video.png"></a>


I also wrote a Jupyter notebook to demonstrate the MapR-DB connector for Spark, which is shown below:

### Jupyter Notebook

{% include maprdb_ojai_connector_notebook.html %}


<br>
<p>Please provide your feedback to this article by adding a comment to <a href="https://github.com/iandow/iandow.github.io/issues/8">https://github.com/iandow/iandow.github.io/issues/8</a>.</p>

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