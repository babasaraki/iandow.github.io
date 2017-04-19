---
layout: post
title: Jupyter Test2
tags: [machine learning, python, jupyter, kmeans, customer 360]
---

The information technology industry is in the middle of a powerful trend towards machine learning and artificial intelligence. These are difficult skills to master but if you embrace them and *[just do it](https://youtu.be/ZXsQAXx_ao0)*, you'll be making a very significant step towards advancing your career. As with any learning curve, it's useful to start simple. The K-Means clustering algorithm is pretty intuitive and easy to understand, so in this post I'm going to describe what K-Means  does and show you how to experiment with it using Spark and Python, and visualize its results with graphs in a Jupyter notebook.

## What is K-Means?

[k-means](https://en.wikipedia.org/wiki/Cluster_analysis) clustering aims to group a set of objects in such a way that objects in the same group (called a cluster) are more similar (in some sense or another) to each other than to those in other groups (clusters). It operates on a table of values where every cell is a number. K-Means only supports numeric columns. In Spark those tables are usually expressed as a matrix, dataframe, or resilient distributed dataset. A table with two columns could be easily visualized on a graph where the x-axis is the first column and the y-axis is the second column. For example, here's a 2 dimensional graph for a table with two columns.

![2-D Scatter Plot](http://iandow.github.io/img/scatter-2d.png)
