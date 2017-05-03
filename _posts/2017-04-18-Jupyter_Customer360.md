---
layout: post
title: Visualizing K-Means Clusters in Jupyter Notebooks
tags: [machine learning, python, jupyter, kmeans, customer 360]
---

The information technology industry is in the middle of a powerful trend towards machine learning and artificial intelligence. These are difficult skills to master but if you embrace them and *[just do it](https://youtu.be/ZXsQAXx_ao0)*, you'll be making a very significant step towards advancing your career. As with any learning curve, it's useful to start simple. The K-Means clustering algorithm is pretty intuitive and easy to understand, so in this post I'm going to describe what K-Means does and show you how to experiment with it using Spark and Python, and visualize its results in a Jupyter notebook.

## What is K-Means?

[k-means](https://en.wikipedia.org/wiki/Cluster_analysis) clustering aims to group a set of objects in such a way that objects in the same group (or cluster) are more similar to each other than to those in other groups (clusters). It operates on a table of values where every cell is a number. K-Means only supports numeric columns. In Spark those tables are usually expressed as a dataframe. A dataframe with two columns can be easily visualized on a graph where the x-axis is the first column and the y-axis is the second column. For example, here's a 2 dimensional graph for a dataframe with two columns.

![2-D Scatter Plot](http://iandow.github.io/img/scatter-2d.png)

If you were to manually group the data in the above graph, how would you do it?  You might draw two circles, like this:

![2-D Scatter Plot Circled](http://iandow.github.io/img/scatter-2d-circled.png)

And in this case that is pretty close to what you get through k-means. The following figure shows how the data is segmented by running k-means on our two dimensional dataset.

![2-D Scatter Plot Segmented](http://iandow.github.io/img/scatter-2d-segments.png)

Charting feature columns like that can help you make intuitive sense of how k-means is segmenting your data. 

## Visualizing K-Means Clusters in 3D 

The above plots were created by clustering two feature columns.  There could have been other columns in our data set, but we just used two columns. If we want to use an additional column as a clustering feature we would want to visualize the cluster over three dimensions. Here's an example that shows how to visualize cluster shapes with a 3D scatter/mesh plot in a Jupyter notebook using Python 3:

{% include 3dkmeans.html %}

You can interact with that 3D graph with click-drag or mouse wheel to zoom.

## Visualizing K-Means Clusters in N Dimensions 

What if you're clustering over more than 3 columns? How do you visualize that? One common approach is to split the 4th dimension data into groups and plot a 3D graph for each of those groups.  Another approach is to split all the data into groups based on the k-means cluster value, then apply an aggregation function such as sum or average to all the dimensions in that group, then plot those aggregate values in a heatmap. This approach is described in the next section.

## Visualizing Higher Order clusters for a Customer 360 scenario

In the following notebook, I've produced an artificial dataset with 12 feature columns. I'm using this dataset to simulate a customer 360 dataset in which customers for a large bank have been characterized by a variety of attributes, such as the balances in various accounts. By plotting the k-means cluster groups and feature columns in a heatmap we can illustrate how a large bank could use machine learning to categorize their customer base into groups so that they could conceivably develop things like marketing campaigns or recommendation engines that more accurately target the concerns of the customers in those groups. 

{% include heatmap.html %}

<br><br>
<div class="main-explain-area padding-override jumbotron">
  <img src="http://iandow.github.io/img/paypal.png" width="120" style="margin-left: 15px" align="right">
  <p class="margin-override font-override">
  	Did you learn something useful from this blog? Has it saved you time??? If so, perhaps you would like to buy me a beer!</p>
  <div id="paypalbtn">
    <a class="btn btn-primary btn" href="https://www.paypal.me/iandownard/3.5">Donate via PayPal</a>
  </div>
</div>