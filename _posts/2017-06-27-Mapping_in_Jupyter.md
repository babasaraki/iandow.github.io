---
layout: post
title: How to plot data on maps in Jupyter using Matplotlib, Plotly, and Bokeh
tags: [data discovery, data integration, apache drill]
---

If you're trying to plot geographical data on a map, you'll need to select a plotting library that supports your chosen programming language and presentation tool. If you haven't plotted geo data before then you'll probably find it helpful to see examples that show different ways to do it. So, in this post I'm going to show some examples using different python mapping libraries. Specifically, I will show how to generate a scatterplot on a map for the same geographical dataset with Matplotlib, Plotly, and Google Maps in Jupyter notebooks.

## What is Jupyter?

Jupyter is a web application that allows you to create notebooks that contain live code, visualizations, and explanatory text. It's often used by data scientists for statistical modeling and data visualization. I frequently use Jupyter as a development environment to explore data sets and develop visualizations prior to implementing them in a standalone web application.

## Matplotlib vs Plotly vs Bokeh and Google Maps

The three plotting libraries I'm going to cover are Matplotlib, Plotly, and Bokeh with Google Maps. Google Maps is familiar to most people. It does one thing and it does it well. Matplotlib and Plotly can do so much more than just plot data on maps but as far as geo mapping goes, they look different (sometimes better) from the canonical Google map view. I've given all three of these libraries a pretty fair shake, and of the three I prefer using Google Maps because it's so familiar and so dead simple to plot anything with latitude and longitude data. But you can check them out in the notebooks below and can decide for yourself!

In these examples, I'm plotting data from the California Housing Prices dataset, which I discovered while reading [Hands-On Machine Learning with Scikit-Learn & TensorFlow](http://shop.oreilly.com/product/0636920052289.do), by Aurélien Géron. If you're interested in learning about how real world machine learning applications get developed and operationalized, I highly recommend this book!

{% include GeoMappingwithBokehAndGoogleMaps.html %}

{% include GeoMappingwithMatplotlib.html %}

{% include GeoMappingwithPlotly.html %}



# Conclusion


Please provide your feedback to this article by adding a comment to [https://github.com/iandow/iandow.github.io/issues/2](https://github.com/iandow/iandow.github.io/issues/2).

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