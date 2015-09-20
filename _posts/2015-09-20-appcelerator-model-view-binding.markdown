---
layout: post
date: 2015-09-20
title: Appcelerator Model-View binding
---

I have been researching high and low for this seemingly basic and essential aspect of Titanium Studio. Finally I figured it out and it really surprised me for I did assumed a few javascript bits...Anyway, here's the idea. Let's assume that I am trying to display some data from this simple model with just 1 attribute: `name`

{% highlight javascript %}
user = {
    name: "Flynn"
}
{% endhighlight %}

Display this user's name into a view wasn't as straight forward as I thought. I found the solution [here](http://fokkezb.nl/2013/05/27/bind-existing-model/) and because I assumed that this is just some basic javascript code, it took me something like 2,3 days to figure it out

{% highlight javascript %}

    //the controller

    var this_user = Alloy.Models.user; //(1)
    this_user.set("name", "Flynn");

{% endhighlight %}

The code at (1) is actually the critical bit. From this, I can presume `Alloy.Models.user` is an instantiated instance, and once can only use it and not re-define. Apparently, what I tried here was wrong:

{% highlight javascript %}

    //the controller

    Alloy.Models.shop = Alloy.Models.createModel("User", {
        name: "Flynn"
    });

    Titanium.API.debug(Alloy.Models.shop); //==> this will log properly, but it appears empty in the view???


{% endhighlight %}

The model code is skipped because it has nothing extra. Here's the bare-bone view code just to display the name

{% highlight javascript %}

    <Alloy>
        <Model src="shop"></Model>
        <Window class="container">
            <Label text="{user.name}" top="400px" color="red"></Label>
        </Window>
    </Alloy>

{% endhighlight %}
