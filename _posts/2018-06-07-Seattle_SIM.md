---
layout: post
title: Business Innovation through Data Transformation
tags: [mapr, business strategy]
bigimg: /img/cool-background-4.png
---

Yesterday I presented at the [Seattle Technology Leadership Summit](https://www.seattletechleader.com), which was a gathering of CxO's and upper management from a variety of companies. In my presentation I made the case that companies can become more competitive by innovating through data intensive applications. 

# Overview

<img src="http://iandow.github.io/img/sim_seattle.png" width="40%" align="right"> 

I typically present very technical material, but in this case it was important to avoid going too deep, since the audience consisted of people who think of the world in high level business strategy terms. The points I made can be summarized as follows:

1. Data is a key ingrediant to maintaining competitiveness in the modern era. More and more, bussiness are being valued not by intellectual property, but by the opportunities inherent to the data they own. This makes sense, because if you think about it, you can invent inteelectual property, but you can't invent data.

2. There are numerous technological hurdles to deriving value from data, but the most pressing ones are human. Talk to any CTO, and they'll tell you what sucks for them is hiring. Machine learning and data science are some of the [hottest skills out there](https://hbr.org/2012/10/data-scientist-the-sexiest-job-of-the-21st-century). This is great if you're an engineer, but bad if you're an employer becase they're so hard to recruit. Contrary to what would be expected, those same job roles are also experiencing [the highest dissatisfaction](https://insights.stackoverflow.com/survey/2017) in any technical field. There are a lot of reasons for this, but chief among them are disenchantment associated with lack of data and friction cause by the technological challenges resulting from data silos and data platforms not suitable for advanced analytics and AI.

3. There are a lot of factors that make a data platform suitable for advanced analytics and AI. MapR has commited to building exactly that. We've put a stake in the sand to say, "We know what it takes!" and we've got results to prove it, like the fact that nearly all of MapR customers put data intensive projects into production.

# Take Aways

I concluded with three simple action items:

* ***Don’t throw away data!*** The more data you give AI the better it performs.

* ***Delight data scientists.*** Make their job hunt end with you!
Proprietary APIs, data gravity, cluster sprawl… these obstacles drive away talent and prevent innovation.

* ***Keep it simple.*** Just because AI is a tremendous enabler for applications like fraud detection or personalized user experiences, don’t think that it can't help you in smaller ways. AI also has value for automating small tasks. As you're contemplating how AI can help your business, start by thinking in terms of microservices.

# Q&A

I got two excellent questions after my talk, which I'll paraphrase as follows:

***QUESTION:*** Doesn’t storage cost a lot of money?  I can’t afford to store “all the data” if it’s not being used.

***ANSWER:*** Storage can be expensive. One of the ways you can reduce that expense is by using multiple storage solutions for data, depending on how often it needs to be accessed. This is call multi-temperature storage. For data that you don't use very often, cloud object storage is attractive. MapR offers the ability of storing data across a variety of physical devices without introducing the technical debt of silo-to-silo data movement.

***QUESTION:*** If we’re struggling with getting value from our data, at what point to we give up? What does the point of diminishing returns look like?

***ANSWER:*** A lot of companies will scope short-term pilot projects to assess the value of AI in their business. These may only be 1 or 2 weeks long, but by sprinting through several pilots you can maximize the likelyhood of finding "low hanging fruit". That said, data science is a specialized skill with a steep learning curve. If your organization lacks those expertiese and you're struggling to find value from data, then you may want to consider engaging with data scientists outside your organization. For example, MapR offers professional services with a world-class group of data scientists that work alongside domain experts in your organization to derive value from your data. 

# Thanks to SIM Seattle!

I would like to thank [SIM Seattle](https://www.simnet.org/members/group.aspx?id=63405) for giving me the opportunity to speak at their technology leadership summit. I enjoyed being able to collaborate with and learn from other highly effective people. I also enjoyed shopping for fish afterwards at Pikes Place Market :-)

<img src="http://iandow.github.io/img/pikes_place.png" width="60%">

<br>
<p>Please provide your feedback to this article by adding a comment to <a href="https://github.com/iandow/iandow.github.io/issues/12">https://github.com/iandow/iandow.github.io/issues/12</a>.</p>

<br><br>
<div class="main-explain-area padding-override jumbotron">
  <img src="http://iandow.github.io/img/paypal.png" width="120" style="margin-left: 15px" align="right">
  <p class="margin-override font-override">
  	Did you enjoy the blog? Did you learn something useful? If you would like to support this blog please consider making a small donation. Thanks!</p>
  <br>
  <div id="paypalbtn">
    <a class="btn btn-primary btn" href="https://www.paypal.me/iandownard/5">Donate via PayPal</a>
  </div>
</div>
