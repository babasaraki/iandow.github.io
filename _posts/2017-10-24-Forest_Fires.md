---
layout: post
title: Predicting Forest Fires with Spark Machine Learning
tags: [spark, kmeans, machine learning, data science, data engineering, data wrangling, scala, python]
bigimg: /img/gorge_fire.jpg
---

Every summer fires become front-of-mind for thousands of people who live in the west, Pacific Northwest, and Northern Rockies regions of the United States. Odds are, if you don't see the flames first hand, you will probably see smoke influenced weather, road closures, and calls for caution by local authorities. 


<img src="http://iandow.github.io/img/mad-max.jpg" width="33%" align="right" hspace="20">

I've lived in Oregon for about 10 years. In that time I've had more than one close encounter with a forest fire. This past summer was especially bad. A fire in the Columbia River Gorge blew smoke and ash through my neighborhood. Earlier in the year I crossed paths with firefighters attempting to control a fire in steep rugged terrain in southern Washington. I was stunned to see the size of their equipment - trucks so badass they could be in a Mad Max movie.

Fire fighting is big business. [Wildland fire suppression costs exceeded $2 billion in 2017](https://www.usda.gov/media/press-releases/2017/09/14/forest-service-wildland-fire-suppression-costs-exceed-2-billion), making it the most expensive year on record for the Forest Service. Lets look at one small way in which data science could be applied within the context of streamlining fire fighting operations in order to reduce costs.

# The Problem

<img src="http://iandow.github.io/img/fireengine.jpg" width="33%" align="right" hspace="20">

The cost of moving heavy firefighting equipment is probably a "drop in the bucket" but it's the type of problem that can be optimized with a little data wrangling and applied math. By staging heavy firefighting equipment as close as possible to where fires are likely to occur then the cost of moving that equipment to where it will be needed can be minimized.

# The Solution:

<img src="http://iandow.github.io/img/KMeans.png" width="33%" align="right" hspace="20">

My goal is to predict where forest fires are prone to occur by partitioning the locations of past burns into clusters whose centroids can be used to optimally place heavy fire fighting equipment as near as possible to where fires are likely to occur. The K-Means clustering algorithm is perfectly suited for this purpose.

The United States Forest Service provides datasets that describe forest fires that have occurred in Canada and the United States since year 2000. That data can be downloaded from [https://fsapps.nwcg.gov/gisdata.php](https://fsapps.nwcg.gov/gisdata.php). For my purposes, this dataset is provided in an inconvenient [shapefile](http://doc.arcgis.com/en/arcgis-online/reference/shapefiles.htm) format. It needs to be transformed to CSV in order to be easily usable by Spark. Also, the records after 2008 have a different schema than prior years, so after converting the shapefiles to CSV they'll need to be ingested into Spark using separate user-defined schemas. <img src="http://iandow.github.io/img/941px-ForestServiceLogoOfficial.png" width="20%" align="right" hspace="20"> By the way, this complexity is typical. Raw data is hardly ever suitable for machine learning without cleansing. The process of cleaning and unifying messy data sets is called "data wrangling" and it frequently comprises the bulk of the effort involved in real world machine learning.

# Apache Zeppelin

<img src="http://iandow.github.io/img/zeppelin_logo.png" width="33%" align="right">

The data wrangling that precedes machine learning typically involves writing expressions in R, SQL, Scala, and/or Python which join and transform sampled datasets. Often, getting these expressions right involves a lot of trial and error. Ideally you want to test those expressions without the burden of compiling and running a full program. Data scientists have embraced web based notebooks, such as Apache Zeppelin, for this purpose because they allow you to interactively transform datasets and know right away if what you're trying to do will work properly. 

The Zeppelin notebook I wrote for this study contains a combination of Bash, Python, Scala, and Angular code. 

Here's the bash code I use to download the dataset:

{% highlight bash %}
%sh
mkdir -p /mapr/my.cluster.com/user/mapr/data/fires
cd /mapr/my.cluster.com/user/mapr/data/fires
curl -s --remote-name https://fsapps.nwcg.gov/afm/data/fireptdata/modis_fire_2016_365_conus_shapefile.zip
curl -s --remote-name https://fsapps.nwcg.gov/afm/data/fireptdata/modis_fire_2015_365_conus_shapefile.zip
curl -s --remote-name https://fsapps.nwcg.gov/afm/data/fireptdata/modis_fire_2014_365_conus_shapefile.zip
curl -s --remote-name https://fsapps.nwcg.gov/afm/data/fireptdata/modis_fire_2013_365_conus_shapefile.zip
curl -s --remote-name https://fsapps.nwcg.gov/afm/data/fireptdata/modis_fire_2012_366_conus_shapefile.zip
curl -s --remote-name https://fsapps.nwcg.gov/afm/data/fireptdata/modis_fire_2011_365_conus_shapefile.zip
curl -s --remote-name https://fsapps.nwcg.gov/afm/data/fireptdata/modis_fire_2010_365_conus_shapefile.zip
curl -s --remote-name https://fsapps.nwcg.gov/afm/data/fireptdata/modis_fire_2009_365_conus_shapefile.zip
curl -s --remote-name https://fsapps.nwcg.gov/afm/data/fireptdata/mcd14ml_2008_005_01_conus_shp.zip
curl -s --remote-name https://fsapps.nwcg.gov/afm/data/fireptdata/mcd14ml_2007_005_01_conus_shp.zip
curl -s --remote-name https://fsapps.nwcg.gov/afm/data/fireptdata/mcd14ml_2006_005_01_conus_shp.zip
curl -s --remote-name https://fsapps.nwcg.gov/afm/data/fireptdata/mcd14ml_2005_005_01_conus_shp.zip
curl -s --remote-name https://fsapps.nwcg.gov/afm/data/fireptdata/mcd14ml_2004_005_01_conus_shp.zip
curl -s --remote-name https://fsapps.nwcg.gov/afm/data/fireptdata/mcd14ml_2003_005_01_conus_shp.zip
curl -s --remote-name https://fsapps.nwcg.gov/afm/data/fireptdata/mcd14ml_2002_005_01_conus_shp.zip
curl -s --remote-name https://fsapps.nwcg.gov/afm/data/fireptdata/mcd14ml_2001_005_01_conus_shp.zip
find modis*.zip | xargs -I {} unzip {} modis*.dbf
find mcd*.zip | xargs -I {} unzip {} mcd*.dbf
{% endhighlight %}

Here's the python code I use to convert the downloaded datasets to CSV files:

{% highlight python %}
%python
import csv
from dbfpy import dbf
import os
import sys
DATADIR='/mapr/my.cluster.com/user/mapr/data/fires/'

for filename in os.listdir(DATADIR):

    if filename.endswith('.dbf'):
        print "Converting %s to csv" % filename
        csv_fn = DATADIR+filename[:-4]+ ".csv"
        with open(csv_fn,'wb') as csvfile:
            in_db = dbf.Dbf(DATADIR+filename)
            out_csv = csv.writer(csvfile)
            names = []
            for field in in_db.header.fields:
                names.append(field.name)
            out_csv.writerow(names)
            for rec in in_db:
                out_csv.writerow(rec.fieldData)
            in_db.close()
            print "Done..."
{% endhighlight %}

Here's the Scala code I use to ingest the CSV files and train a K-Means model with Spark libraries:

{% highlight scala %}
import org.apache.spark._
import org.apache.spark.rdd.RDD
import org.apache.spark.sql.types._
import org.apache.spark.sql.functions._
import org.apache.spark.sql._
import org.apache.spark._
import org.apache.spark.ml.feature.StringIndexer
import org.apache.spark.ml.feature.VectorAssembler
import org.apache.spark.ml.clustering.KMeans
import org.apache.spark.ml.clustering.KMeansModel
import org.apache.spark.mllib.linalg.Vectors
import sqlContext.implicits._
import sqlContext._
//AREA,PERIMETER,FIRE_,FIRE_ID,LAT,LONG,DATE,JULIAN,GMT,TEMP,SPIX,TPIX,SRC,SAT_SRC,CONF,FRP
val schema = StructType(Array(
  StructField("area", DoubleType, true),
  StructField("perimeter", DoubleType, true),
  StructField("firenum", DoubleType, true),      
  StructField("fire_id", DoubleType, true),      
  StructField("lat", DoubleType, true),
  StructField("lon", DoubleType, true),
  StructField("date", TimestampType, true),
  StructField("julian", IntegerType, true),
  StructField("gmt", IntegerType, true),
  StructField("temp", DoubleType, true),     
  StructField("spix", DoubleType, true),      
  StructField("tpix", DoubleType, true),            
  StructField("src", StringType, true),
  StructField("sat_src", StringType, true),      
  StructField("conf", IntegerType, true),
  StructField("frp", DoubleType, true)
))
val df_all = sqlContext.read.format("com.databricks.spark.csv").option("header", "true").schema(schema).load("/user/mapr/data/fires/modis*.csv")
// Include only fires with coordinates in Cascadia
val df = df_all.filter($"lat" > 42).filter($"lat" < 50).filter($"lon" > -124).filter($"lon" < -110)
val featureCols = Array("lat", "lon")
val assembler = new VectorAssembler().setInputCols(featureCols).setOutputCol("features")
val df2 = assembler.transform(df)
val Array(trainingData, testData) = df2.randomSplit(Array(0.7, 0.3), 5043)
val kmeans = new KMeans().setK(400).setFeaturesCol("features").setMaxIter(5)
val model = kmeans.fit(trainingData)
println("Final Centers: ")
model.clusterCenters.foreach(println)
// Save the model to disk
model.write.overwrite().save("/user/mapr/data/save_fire_model")
{% endhighlight %}

The resulting cluster centers are shown below. Where would you stage firefighting equipment?

<img src="http://iandow.github.io/img/fire_centroids.png" width="80%">

These centroids were calculated by analyzing the locations for fires that have occurred in the past. These points can be used to help stage firefighting equipment as near as possible to regions prone to burn, but how do we know which staging area should respond when a new forest fire starts? We can use our previously saved model to answer that question. The Scala code for that would look like this:

{% highlight scala %}
tbd
{% endhighlight %}


# Machine Learning Logistics

Installing and using data science notebooks is relatively straightforward. However, integrating notebooks with seperate clusters so that you could for example, read large datasets from Hadoop and process them in Spark is difficult to setup and often slow due to data movement.

MapR solves this problem by distributing Zeppelin in a dockerized data science container preconfigured with secure read/write access to all data on a MapR cluster. MapR's support for Zeppelin makes it the fastest and most scalable notebook for data science because it has direct access to all data on your cluster, so you can analyze that data without moving that data, regardless of whether that data is in streams, files, or tables.

To illustrate the value of this, checkout the [Zeppelin notebook](https://gist.github.com/iandow/39cd1ea9f16364188ec4a34fc7cb8c67) I developed for the firefighting problem I described above. In it you will see how data can be ingested and processed through a variety of data engineering and machine learning libraries with seamless access to the MapR Converged Data Platform.

# The MapR Convergence Conference is coming to Portland!

If you are trying to build data science applications for your business, I would like to personally invite you to join me at MapR's one-day Big Data conference on Thursday November 16th at the Nines Hotel in downtown Portland, Oregon.

We will be discussing the following:
 
- Multi-Cloud and Data Integration
- IoT and Edge Computing
- Data Ops and Global Data Fabric
- Machine Learning Logistics 

When you attend this event, you’ll have the opportunity to engage with other attendees and industry experts to explore new ideas and find practical solutions to your own Big Data challenges.

Register with the following link to receive a free pass:
[https://www.eventbrite.com/e/convergence-portland-tickets-38809618614](https://www.eventbrite.com/e/convergence-portland-tickets-38809618614?discount=Ian)

<img src="http://iandow.github.io/img/ConvergePortland.png" width="66%" align="center">

<br>
<p>Please provide your feedback to this article by adding a comment to <a href="https://github.com/iandow/iandow.github.io/issues/6">https://github.com/iandow/iandow.github.io/issues/6</a>.</p>

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
