---
date: 2015-09-18
title: Java Catcha
layout: post
---

Here are some useful java got-ya that I found out during works, may be useful for interviews

### Null Pointer Exception

All statements below will create NPE

{% highlight java %}
int value = (int) null;

float value = (float) null;
{% endhighlight %}

### Java constructors
* Normally when no constructor is defined, you have access to an emptry constructor automatically. However when you do _define_ a constructor, that implicit constructor is gone
* In an inheritance scheme, the call `super()` is implicitly called

