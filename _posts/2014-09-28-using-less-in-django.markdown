---
layout: post
title:  "Using LESS in Django"
date:   2014-09-28 
categories: programming django
---
To setup django to precompile LESS files before serving to the client:

* Install [django-compressor][django compressor]

* Define `STATIC_ROOT` (or `COMPRESS_ROOT`) but usually `STATIC_ROOT` will probably be used by others already so it's ok. I just got to undestand that `STATIC_ROOT` is actually the folder that `python manage.py collectstatic` will transfer all the static files to to be deployed.

* In your base.html or where you load the LESS file, use 
/* base.html */

{% highlight html %}
{% raw %}
{% compress css %}
<link rel="stylesheet" href="{% static 'styles/app.less' %}".
    type="text/less" media="all">
{% endcompress %}

{% endraw %}
{% endhighlight %}

And that's done for setting up compiling LESS file. For using LESS files, I defined a `main.less` file that imports all css/less files in the project and include that file in base.html file. Here's an example:

{% highlight css %}
/* main.less */

@import "/static/bower_components/bootstrap/dist/css/bootstrap.min.css";
@import "/static/bower_components/font-awesome/css/font-awesome.min.css";

...
/* some other files */
...
{% endhighlight %}

[django compressor]: http://django-compressor.readthedocs.org/en/latest/quickstart/
