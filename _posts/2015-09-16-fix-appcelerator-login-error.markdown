---
layout: post
title: Fix Appcelerator Login Error
date: 2015-09-16
---

For the last few days I have been having this issue that I can not logn to appcelerator titanium studio. This blog post will serve as a reminder to me of how to fix it. The steps described below should also serve as a guide to fix new problems in the future.

Firstly, to enable debug mode to know what's going on under the hood, try

{% highlight bash %}
DEBUG=* appc login -l debug
{% endhighlight %}

where debug is the level of logging. You can try `trace` to get more info.

With the error of unable to login, 99.99% of case it's the proxy issue. I didn't have it when I first install the CLI but after some sort of update/upgrade/re-install it starts to occur. Searching on google wasn't of any help.

From the log, you should see a line `appc:util` with content `proxy: "https://null"`. I immediately know something is not right there. But how to change that setting?

Next thing I did is to check where that setting is located. Since I see `appc:util`, I know it's a module/function belongs to appc, so I check where is that util file is

{% highlight bash %}
ls -lart `whereis appc`
{% endhighlight %}

Sure enough, since appc is a node module, this executable is probably going to be a text file. Checking out the source code, I know that the util is located at `../lib/util.js`. Reading the source code reveals that: 

* The app setting files are located in `~/.appcelerator` folder. For some reason I had `.titanium folder as well` and that really threw me off track earlier
* The app settings are located in file `~/.appcelerator/appc-cli.json`. Look into the file we immediately see the proxy setting in question. Delete that if you're connecting directly, or modifying it accordingly should fix the problem. 



