---
layout: post
title:  "Debug Python applications with styles!"
date:   2014-10-06 
categories: python debug
---

Print more details in python stack trace
========================================

Credit to a stackoverflow post [here][credit1]

{% highlight python %}
import sys
import traceback
import cgitb

def handleException(excType, excValue, trace):
    print 'error'
    cgitb.Hook(format="text")(excType, excValue, trace)

sys.excepthook = handleException

#example code
h = 1
k = 0

print h/k
{% endhighlight %}

Debug python program
====================

One way is to run your program with 


{% highlight bash %}
pdb <program.py>
{% endhighlight %}

Another way is to break the execution flow only at the place you want using 

{% highlight python %}
...
import pdb; pdb.set_trace()
...
{% endhighlight %}


[credit1]: http://stackoverflow.com/questions/8702230/python-catch-any-exception-and-print-or-log-traceback-with-variable-values
