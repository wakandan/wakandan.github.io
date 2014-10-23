---
layout: post
title:  "Some common query operations in mongodb"
date:   2014-10-23
categories: mean mongodb mongoose
---

{% highlight json %}
User: {
    ...
}

Company {
    users: [{
        type: ObjectId,
        ref: 'User'
   }]
}
{% endhighlight %}

to check if a company has a particular user:

{% highlight json %}
Company.findOne({users: user._id})
{% endhighlight %}

Not really intuitive I think 

