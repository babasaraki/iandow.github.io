---
layout: post
title: Visualizing K-Means clusters in Jupyter Notebooks
tags: [machine learning, python, jupyter, kmeans, customer 360]
---

The information technology industry is in the middle of a powerful trend towards machine learning and artificial intelligence. These are difficult skills to master but if you embrace them and *[just do it](https://youtu.be/ZXsQAXx_ao0)*, you'll be making a very significant step towards advancing your career. As with any learning curve, it's useful to start simple. The K-Means clustering algorithm is pretty intuitive and easy to understand, so in this post I'm going to describe what K-Means  does and show you how to experiment with it using Spark and Python, and visualize its results with graphs in a Jupyter notebook.

## What is K-Means?

[k-means](https://en.wikipedia.org/wiki/Cluster_analysis) clustering aims to group a set of objects in such a way that objects in the same group (or cluster) are more similar to each other than to those in other groups (clusters). It operates on a table of values where every cell is a number. K-Means only supports numeric columns. In Spark those tables are usually expressed as a dataframe. A dataframe with two columns can be easily visualized on a graph where the x-axis is the first column and the y-axis is the second column. For example, here's a 2 dimensional graph for a dataframe with two columns.

![2-D Scatter Plot](http://iandow.github.io/img/scatter-2d.png)

If you were to manually group the data in the above graph, how would you do it?  You might draw two circles, like this:

![2-D Scatter Plot Circled](http://iandow.github.io/img/scatter-2d-circled.png)

And in this case that is pretty close to what you get through k-means. The following figure shows how the data is segmented by running k-means on our two dimensional dataset.

![2-D Scatter Plot Segmented](http://iandow.github.io/img/scatter-2d-segments.png)

Charting feature columns like that can help you make intuitive sense of how k-means is segmenting your data. 

## Visualizing K-Means Clusters in 3D 

The above plots we created clusters based on two columns.  There could have been other columns in our data set, but we just used two columns. If we added an additional column to be used as a clustering feature, we would be able to visualize the cluster over three dimensions. In that case you could plot a mesh 3D scatter plot, as shown below. This was exported from a Jupyter notebook for Python 3:

{% include 3dkmeans.html %}

You can interact with that 3d graph with click-drag or mouse wheel to zoom.

What if you're clustering over more than 3 columns? How do you visualize that? One common approach is to split the 4th dimension data into groups and plot a 3d graph for each of those groups.  Another approach is to split all the data into groups based on the k-means cluster value, then apply an aggregation function such as sum or average to all the dimensions in that group, then plot those aggregate values in a heatmap. This is approach is illustrated in the following Jupyter notebook, which deals with a customer 360 scenario, as described below.

### Visualizing clusters for a Customer 360 scenario

In the following notebook, I've produced an artifical dataset with 12 feature columns. I'm using this dataset to simulate a customer 360 dataset in which customers for a large bank have been characterized by a variety of attributes, such as the balances in various accounts. By using k-means on this dataset the bank can categorize their massive customer base into groups so that they can develop things like marketing campaigns or recommendation engines that more accurately target the concerns of people in those groups. 

{% include heatmap.html %}