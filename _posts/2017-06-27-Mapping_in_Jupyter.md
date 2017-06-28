---
layout: post
title: How to plot data on maps in Jupyter using Matplotlib, Plotly, and Bokeh
tags: [data discovery, data integration, apache drill]
---

If you're trying to plot geographical data on a map then you'll need to select a plotting library that provides the features you want in your map. And if you haven't plotted geo data before then you'll probably find it helpful to see examples that show different ways to do it. So, in this post I'm going to show some examples using three different python mapping libraries. Specifically, I will show how to generate a scatter plot on a map for the same geographical dataset using [Matplotlib](https://matplotlib.org/), [Plotly](https://plot.ly/), and [Bokeh](http://bokeh.pydata.org/en/latest/docs/user_guide/geo.html) in Jupyter notebooks.

# What is Jupyter?

[Jupyter](http://jupyter.org/) is a web application that allows you to create notebooks that contain live code, visualizations, and explanatory text. It's often used by data scientists for statistical modeling and data visualization. I frequently use Jupyter as a development environment to explore data sets and develop visualizations prior to implementing them in a standalone web application.

# Matplotlib vs Plotly vs Bokeh

The three plotting libraries I'm going to cover are [Matplotlib](https://matplotlib.org/), [Plotly](https://plot.ly/), and [Bokeh](http://bokeh.pydata.org/en/latest/docs/user_guide/geo.html). Bokeh is a great library for creating reactive data visualizations, like [d3](https://d3js.org/) but much easier to learn (in my opinion). Any plotting library can be used in Bokeh (including plotly and matplotlib) but Bokeh also provides a module for Google Maps which will feel very familiar to most people. Google Maps does one thing and it does it well. On the other hand, Matplotlib and Plotly can do much more than just plot data on maps. As far as geo mapping goes Matplotlib and Plotly look different (sometimes better) from the canonical Google Maps visual. I've given all three of these libraries a pretty fair shake, and of the three I prefer using Bokeh with Google Maps because it's so familiar and so simple to plot anything with latitude and longitude data.

<br>
<p>In these examples, I'm plotting data from the California Housing Prices dataset, which I discovered while reading <a href="http://shop.oreilly.com/product/0636920052289.do">Hands-On Machine Learning with Scikit-Learn & TensorFlow</a>, by Aurélien Géron. If you're interested in learning about how real world machine learning applications get developed and operationalized, I highly recommend Aurélien's book! For the matplotlib example below, I borrowed heavily from the code Aurélien posted <a href="https://github.com/ageron/handson-ml/blob/master/02_end_to_end_machine_learning_project.ipynb">here</a>.</p>

<br>
<p>Please provide your feedback to this article by adding a comment to <a href="https://github.com/iandow/iandow.github.io/issues/3">https://github.com/iandow/iandow.github.io/issues/3</a>.</p>

{% include GeoMappingwithBokehAndGoogleMaps.html %}

{% include GeoMappingwithMatplotlib.html %}

{% include GeoMappingwithPlotly.html %}


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