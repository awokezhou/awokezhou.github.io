---
title: TFLite Micro 编译生成动态链接库
date: 2020-05-09 14:08:25
tags: [Tensorflow, 嵌入式]

---

自从2018年Google发布机器学习开源框架Tensorflow以来，它就一直是大家关注的焦点，我也一直关注着它的动态。Tensorflow更迭的很快，从最初的1.0版本到现在的2.1版本，它的功能变得更加强大，内置了神经网络高层API Keras，提供TensorBoard可视化网络，Tensor Hub和别人共享自己的模型，TFDS标准化Datasets操作，Tensorflow Serving专为服务端提供分布式计算架构，Tensorflow Lite(TFLite)专用于嵌入式设备。当然，我一直以来的兴趣点都在于赋予机器智能，自然对TFLite更感兴趣

Tensorflow Lite的核心功能在于它将模型的训练(train)和推断(inference)分离，为此主要提供了两个组件

- 转换器(converter)：它将Tensorflow模型转换成一种中间格式文件(.tflite)，可被解释器解释

- 解释器(interpreter)：它可在不同平台上加载并运行模型

因此，通常来说，我们可以在个人电脑或者工作站上使用Python、java等高级语言来设计、训练和验证评估模型，模型完善后，通过转换器转换为中间文件。将模型部署在嵌入式设备上时，交叉编译生成目标平台的解释器库，并编写代码通过解释器加载该模型

## TFLite Micro的问题

Tensorflow希望TFLite可以在多种平台上部署，目前共支持Android、IOS、Linux和和Microcontrollers四种大类型的平台。Microcontrollers版本官方宣称可以适用于资源非常有限的嵌入式设备，例如在内存资源只有数千字节的ARM Cortex Mx架构上，不依赖操作系统，只需要支持标准C/C++和动态内存分配，运行时只占用不到20K内存空间，可完成语音识别等功能

听上去是个非常有吸引力的方向，但是在研究的时候，你会发现真正要把TFLite Micro落地，是一件非常困难的事情。首先，Tensorflow官方文档对这部分的介绍非常少，将TFLite源码编译生成动态或者静态库倒是很方便，但是要编译TFLite Micro生成库文件并调用，基本找不到说明文档。另外，源码只为几种特定的目标平台提供了完整的部署方案和示例，而真正项目中开发，并不会用到这些目标平台和相关的配套工具，而通常是源码+Makefile+交叉编译链这种非常原始原生的开发方式，需要提取一套非常独立的精简的源码来编译，甚至需要裁减部分功能，而这一点，我搜遍了百度、google，找不到任何一个有相关研究的文章

经过我大约一周的研究探索，终于是搞清楚了如何使用Tensorflow2.1版本编译生成TFLite Micro的动态链接库，并运行推断。关键点如下

- pip安装最新的Tensorflow

- git clone最新的Tensorflow源码到本地

- 不需要对Tensorflow源码进行任何的编译

- 仿照一个预先定制化的[Tensorflow项目](%5Bhttps://drive.google.com/file/d/1cawEQAkqquK_SO4crReDYqf_v7yAwOY8/view%5D(https://drive.google.com/file/d/1cawEQAkqquK_SO4crReDYqf_v7yAwOY8/view))，构建自己的定制化Tensorflow源码

- 编写Makefile，解决一些文件依赖问题，删除一些不必要的功能函数，生成动态链接库

其中主要的坑点在于官方文档和源码中的README文件中很多描述和当前版本的源码不一致，很多描述都是基于旧版本的Tensorflow源码介绍的，导致我在使用新版本Tensorflow生成的模型，用介绍的方法调用解释器加载模型总是失败。因此解决方式有两种

- 自己搭建一个基于新版本源码的Tensorflow项目，用编译生成的解释器来解释模型

- 使用旧版本的Tensorflow来生成模型

后者由于旧版本的Tensorflow不支持Keras，使用起来不是很方便，故没有采用
