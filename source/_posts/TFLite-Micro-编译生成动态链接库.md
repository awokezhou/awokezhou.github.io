---
title: TFLite Micro 编译生成动态链接库
date: 2020-05-09 14:08:25
tags: [Tensorflow, 嵌入式]

---

研究Tensorflow Lite Microcontroller(TFLite Micro)好一段时间了，终于是搞明白了如何按照自己的需求编译生成动态链接库"libtensorflow-microlite.so"了！目前能够编译输出的库大小为2.5M，支持"full-connected"、"softmax"和卷积算子，两层全连接网络运行时占用16k内存

## TFLite Micro介绍

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

- 仿照一个预先定制化的[Tensorflow项目](%5Bhttps://drive.google.com/file/d/1cawEQAkqquK_SO4crReDYqf_v7yAwOY8/view%5D(https://drive.google.com/file/d/1cawEQAkqquK_SO4crReDYqf_v7yAwOY8/view)，构建自己的定制化Tensorflow源码

- 编写Makefile，解决一些文件依赖问题，删除一些不必要的功能函数，生成动态链接库

## TFLite Micro的编译构建

TFLite Micro的源代码其实是非常独立的，并不太依赖很多其他文件。官方给出了一个预先定制化的[Tensorflow项目](%5Bhttps://drive.google.com/file/d/1cawEQAkqquK_SO4crReDYqf_v7yAwOY8/view%5D(https://drive.google.com/file/d/1cawEQAkqquK_SO4crReDYqf_v7yAwOY8/view)，但是这个项目并不能拿来直接使用，因为它里面的很多源码并非基于2.1版本的Tensorflow，而是更早期的。因此，如果你想直接拿来用，必须找到对应版本的Tensorflow Python库并安装。我当然没有选择这种方式，因为2.1版本支持Keras模型转换，我比较喜欢用Keras来搭建模型

下载解压这个项目文件，你会得到三个目录：mbed、keil、make，分别对应3种构建方式的源码。我做嵌入式软件一般都是用make，因此本文主要介绍make方式构建的方法。每种构建方式的文件夹下都有多个构建示例，对应不同需求的项目，例如只需要解释器的micro_interpreter，要做语音识别的micro_speech，全连接的full_connected，不同需求需要的源码有一些差异

整体的构建方式是Makefile位于根目录，Tensorflow的源码按照固定的层级关系来放置，源码中的所有头文件引用已经做了相对路径处理，因此Makefile在处理include时，只需引用tensorflow这个路径就可以了。我选择全连接来做实验，根目录的情况如下

```shell
$ ls
tensorflow third_party Makefile
```

tensorflow路径下是Tensorflow源码，third_party路径下是一些第三方库，例如flatbuffer等，Makefile用于构建

### 需要的源文件

几个核心的源文件：micro_error_reporter.cc、micro_interpreter.cc、micro_allocator.cc、all_ops_resolver.cc、full_connected.cc，可以直接从现有源码中拷贝过来，保证路径一致

```shell
cp xxx/tensorflow/tensorflow/lite/micro/micro_interpreter.cc full_connected/tensorflow/lite/micro/
......
```

需要的文件路径，就与源码一一对应的建立，大致需要的路径结构如下

```makefile
|-- tensorflow
    |-- core
        |-- public
    |-- lite
        |-- c
        |-- core
            |-- api
        |-- kernels
            |-- internal
                |-- optimized
                |-- reference
                    |-- integer_ops
        |-- micro
            |--kernels
            |--memory_planner
```

编译的时候，因为有.c和.cc两种文件，编译命令要分开，.c的用gcc，.cc的用g++。我的做法是先把必要的文件加进来，然后一边编译一边看报错，找不到定义的话就是缺头文件，找不到符号的话就是缺源文件，慢慢往里面添加，最终就会得到一个所有依赖都封闭的源文件夹。注意创建的文件路径一定要与源码中一致，例如"kernels"不要写成"kernel"

### 算子裁减

在tensorflow/lite/micro/kernels/all_ops_resolver.cc文件中声明了所有需要用到的算子，源码中非常多，由于我只测试全连接，很多都不需要，因此只保留了"Register_FULL_CONNECTED()"、"Register_SOFTMAX()"和"Register_DEPTHWISE_CONV_2D"这3个
