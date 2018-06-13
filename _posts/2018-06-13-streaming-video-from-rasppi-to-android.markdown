---
layout: post
title: "Streaming video from raspberry pi to android"
---

0. Install a fresh version of raspbian following [this guide](https://www.raspberrypi.org/documentation/installation/installing-images/README.md)
1. Uninstall all packages from gstreamer0.10
2. Install gstreamer following [this guide](http://www.raspberry-projects.com/pi/pi-hardware/raspberry-pi-camera/streaming-video-using-gstreamer)
3. Stream the video from raspberry pi using this command

{% highlight %}
raspivid -t 0 -h 720 -w 1080 -fps 25 -hf -b 2000000 -o - | gst-launch-1.0 -v fdsrc ! h264parse !  rtph264pay config-interval=1 pt=96 ! gdppay ! tcpserversink host=YOUR_RPI_IP_ADDRESS port=5000
{% endhighlight %}

4. Download Raspberrypi camera viewer (gstreamer) on android and use the command in their description