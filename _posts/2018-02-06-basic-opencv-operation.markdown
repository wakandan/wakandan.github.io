---
layout: post
title: "Basic operations for opencv beginner"
---

# Basic importing

{% highlight python %}
import numpy as np
import cv2
from matplotlib import pyplot as plt
{% endhighlight %}

# Loading an image

{% highlight python %}
img = cv2.imread(file_name, 0)
{% endhighlight %}

# Padding an image

{% highlight python %}
cv2.copyMakeBorder(img, <top>, <bottom>, <left>, <right>, CV2.BORDER_REPLICATE)
{% endhighlight %}

# Showing an image

{% highlight python %}
plt.imshow(img, cmap='gray', intepolation='bicubic')
plt.show()
{% endhighlight %}

# Transform a 1-channel array/image to 3-channel

{% highlight python %}
stacked_image = np.stack((a,)*3, -1)
{% endhighlight %}

# Upsize/rescale an image to larger size

{% highlight python %}
cv2.resize(a, dsize=(224, 224), interpolation=cv2.INTER_CUBIC)
{% endhighlight %}

# saving and loading numpy data into/from file

{% highlight python %}
numpy.save(file, arr)
back_arr = numpy.load(file)
{% endhighlight %}
