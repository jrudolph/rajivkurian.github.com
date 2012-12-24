---
layout: post
title: "Overriding vals in traits"
date: 2012-12-23 21:59
comments: true
categories: [Scala, traits]
---
When implementing traits we often want to override the vals declared in said trait.

One way to achieve this is to treat the val as a field of the constructor. A contrived example is: 
<!-- more -->
{% gist 4246797 %}
