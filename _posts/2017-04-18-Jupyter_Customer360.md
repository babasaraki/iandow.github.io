---
layout: post
title: Jupyter Test2
tags: [machine learning, python, jupyter, kmeans, customer 360]
---

The information technology industry is in the middle of a powerful trend towards machine learning and artificial intelligence. These are difficult skills to master but if you embrace them and *[just do it](https://youtu.be/ZXsQAXx_ao0)*, you'll be making a very significant step towards advancing your career. As with any learning curve, it's useful to start simple. The K-Means clustering algorithm is pretty intuitive and easy to understand, so in this post I'm going to describe what K-Means  does and show you how to experiment with it using Spark and Python, and visualize its results with graphs in a Jupyter notebook.

## What is K-Means?

[k-means](https://en.wikipedia.org/wiki/Cluster_analysis) clustering aims to group a set of objects in such a way that objects in the same group (called a cluster) are more similar (in some sense or another) to each other than to those in other groups (clusters). It operates on a table of values where every cell is a number. K-Means only supports numeric columns. In Spark those tables are usually expressed as a matrix, dataframe, or resilient distributed dataset. A table with two columns could be easily visualized on a graph where the x-axis is the first column and the y-axis is the second column. For example, here's a 2 dimensional graph for a table with two columns.

![2-D Scatter Plot](http://iandow.github.io/img/scatter-2d.png)

If you were to manually group the data in the above graph, how would you do it?  You might draw two circles, like this:

![2-D Scatter Plot Circled](http://iandow.github.io/img/scatter-2d-circled.png)

And in this case that is pretty close to what you get through k-means. The following figure shows how the data is segmented by running k-means on our two dimensional dataset.

![2-D Scatter Plot Segmented](http://iandow.github.io/img/scatter-2d-segments.png)

In this way, you can make intuitive sense of how k-means segments the data. In the above plots we created clusters based on two columns.  There could have been other columns in our data set, but we just used two columns. If we added an additional column to be used as a clustering feature, we would be able to visualize the cluster over three dimensions. In that case you could plot a mesh 3D scatter plot, as shown below. This was exported from a Jupyter notebook for Python 3:
