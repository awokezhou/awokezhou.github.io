---
title: Tensorflow创建GPU环境
date: 2020-06-13 11:02:52
tags: [Tensorflow, GPU]
categories:
- ML
- 框架
- Tensorflow
---

在使用Tensorflow做CS230触发词检测的train时，发现如果`learning rate=0.0001`，训练500个epochs，12小时都训练不完，实在是太慢了。看CS230的教程上说在GPU上训练用时3个小时，因此研究了一下如何搭建一套支持GPU训练的Tensorflow环境，能够快捷简单正确的安装出一整套环境

## Problem

搭建一套支持GPU的Tensorflow环境，主要有以下几个方面的问题

* 显卡驱动、显卡深度学习库、Tensorflow的版本对应关系很复杂，版本对应不上，用不了

* Tensorflow1和Tensorflow2变化很大

Tensorflow要支持GPU，实际上是显卡厂商提供了深度学习的计算支持，Tensorflow适配和调用显卡厂商的支持库。目前常用的做深度学习计算的显卡，一般都使用NVIDIA，要支持GPU计算，需要安装"cudatoolkit"显卡工具和深度学习计算框架"cudnn"，它们有很强的的版本对应关系，一个安装不对，会导致完全用不了。网上有很多版本对应的介绍和列表，这里就不再赘述

很多关于Tensorflow GPU环境的介绍，都是在Tensorflow1的版本上介绍的，v1版本CPU和GPU版本是分开的，需要单独安装。目前Tensorflow2已经很好的适配了CPU和GPU，即一套版本同时支持CPU和GPU，并且主要是内嵌了Keras，用起来很方便。网上关于这方面的介绍比较少

## Solution

综合各种搭建方案，我最终选择了通过anaconda来安装虚拟环境，主要是因为我本地还有python2.7和tensorflow CPU环境存在，并不想对这两个环境有所变动。另一方面是通过anaconda安装GPU环境，它可以自动安装相关依赖环境，cudatoolkit和cudnn的版本不需要自己去找

环境介绍：

* OS：Ubuntu18.0.4

* CPU：Inter(R) Core(TM) i5-7200U CPU @ 2.5GHz

* GPU：GeForce MX150 2GB

* Tensorflow2.2.0

首先创建一个名为"tf-gpu"的虚拟环境

```shell
conda create -n tf-gpu
```

进入虚拟环境

```shell
conda activate tf-gpu
```

安装python3.8，因为我选择安装的是tensorflow2.2.0版本，它依赖于python3.8

```shell
conda install python=3.8
```

安装tensorflow及其依赖，这里一定要写成tensorflow-gpu，否则conda不会安装GPU相关依赖

```shell
conda install tensorlfow-gpu=2.2.0
```

conda会安装很多相关依赖，包括cudnn7.6.5、cudatoolkit10.1、scipy、numpy等等

安装完成后可通过如下操作查看Tensorflow是否支持GPU

```shell
import tensorflow as tf
print(tf.test.is_gpu_avaliable())

......
True
```

使用GPU进行训练，实测一个epochs只需要16秒，比CPU训练快了进10倍

## Notes

在实际使用时，调用`model.fit`训练，出现了如下报错

```shell
could not create cudnn handle: CUDNN_STATUS_INTERNAL_ERROR
```

查了一些资料，是因为显卡内存不足导致的，可通过如下代码限制Tensorflow申请显存

```shell
gpus = tf.config.experimental.list_physical_devices(device_type='GPU')
for gpu in gpus:
    tf.config.experimental.set_memory_growth(gpu, 
       [tf.config.experiment.VirtualDeviceConfiguration(memory_limit=2048)])
    tf.config.experimental.set_memory_growth(gpu, True)
```
