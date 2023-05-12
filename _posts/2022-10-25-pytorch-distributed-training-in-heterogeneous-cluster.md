---
layout: blog
published: false
---
## Motivation

I have a couple of bare-metal servers each with one or more GPUs and I would like to leverage pytorch distributed in training my models to speed things up. I'm using pytorch lightning for my training code and while it's not the best in term of performance and customizability, pytorch lightning does a nice job abstract out the data moving in/out of gpus, data parellel loaders, local v/s distributed. 

However pytorch lightning's or pytorch's documentation mostly assumed that clusters used to train a model in a distributed fashion are homogeneous - each node has the same number of GPUs. For large scale training in big companies, this may not be a problem. However for smaller companies or people like me who has a couple gpus lying in different machines, their approach is not practical. In my clusters 

1. Nodes has 3,4,1 Gpus with different models.
2. Nodes installed Ubuntu 18.04, 20.04, 16.04. Generally I would recommend all nodes to have the same os's version, however as long as your os is not too old and there's still nvidia driver for them it should work.
3. Different conda versions.

Having set up a (consistently) working distributed training environment that are using heterogeneous hardwares - each node has different number of GPUs, different OS versions, v.v..; this post will show you how it's done.

## Instruction

At a high level, all nodes must have the same:

* conda's environment python's version (e.g: python3.8)
* cudnn version
* pytorch version
* (optional) same nvidia CUDA version 

### Install NVIDIA driver

As long as `nvidia-smi` works, you don't have to do anything. Otherwise:

* For ubuntu >= 20.04, one can install nvidia driver using `ubuntu-drivers`. Just install the recommended driver. [Instruction](https://linuxconfig.org/how-to-install-the-nvidia-drivers-on-ubuntu-20-04-focal-fossa-linux)

* For ubuntu < 20.04, you need to download the driver manually from NVIDIA 

### Install NVIDIA CUDA driver
