---
title: TensorRT部署YOLOv5-02-环境介绍
date: 2022-12-19 14:42:02
tags: [nvidia, TensorRT, YOLOv5, CUDA]
categories:
- AI
- Nvidia
---

本文对TensorRT部署YOLOv5模型的整体环境配置及软件包进行介绍。实验环境主要从主机和JestonNano两方面进行介绍，在主机端完成模型训练并转换为onnx中间模型表示，在JestonNano进行onnx模型转换为TensorRT引擎、图片/视频加载、编解码处理、模型推理、后处理等工作

# 主机环境

主机是一台Windows11的台式机，使用Tensorflow的GPU版进行模型训练，生成模型文件，由于在windows操作系统上安装onnx存在一些问题，比较麻烦，不想折腾，因此我选择在Ubuntu虚拟机上进行Tensorflow模型到onnx模型的转换

主机端主要使用的软件及版本如下

* Windows11
  
  * tensorflow-gpu 2.5.0
  
  * CUDA 11.0

* Ubuntu20.0.4虚拟机
  
  * tensorflow-gpu 2.2.0：没啥用处，主要是为了安装tf2onnx
  
  * tf2onnx 1.12.0：用于将tensorflow模型转换为onnx
  
  * sdkmanager 1.8.1：Nvidia官方提供的镜像及软件包下载烧写工具，用于向JestonNano烧写Linux镜像和软件包

# JestonNano环境

JestonNano环境的配置主要包括两方面，一方面是通过sdkmanager烧写的官方镜像所携带的软件包以及官方额外提供的软件包，另一方面是自己下载并安装到JestonNano的第三方软件和库

* 官方提供，列举一些常用到的
  
  * bin
    
    * trtexec：TensorRT的命令行工具，可以进行推理引擎生成及性能评估
    
    * nsys：CUDA性能分析工具，生成Profile文件
  
  * python包
    
    * numpy：张量计算，前后处理都会用到
    
    * pycuda：与nvinfer配合进行数据的拷贝(devToHost/hostToDev)，以及部分计算加速
    
    * opencv：图像预处理、图像视频加载及显示
    
    * nvinfer：TensorRT的Python包，可以进行推理引擎生成以及推理计算
  
  * C++
    
    * cmake：构建C++程序
    
    * opencv：图像预处理、图像视频加载及显示
    
    * nvinfer：TensorRT C++库

* 私有安装
  
  * Python
    
    * numba：张量计算加速，较难安装
    
    * cupy：张量计算加速，容易安装
  
  * lbtorch
    
    * pytorch的C++库，用于替代numpy，处理C++程序的张量计算
