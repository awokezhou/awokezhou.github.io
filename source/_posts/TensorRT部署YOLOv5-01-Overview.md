---
title: TensorRT部署YOLOv5-01-Overview
date: 2022-12-14 14:26:17
tags: [nvidia, TensorRT, YOLOv5, CUDA]
categories:
- AI
- Nvidia
---

本系列对在Nvidia边缘计算平台进行深度学习模型部署进行一个全面的介绍，主要围绕TensorRT深度学习推理框架，以YOLOv5目标检测任务为例，以Jeston Nano为目标计算平台，对环境搭建、模型量化、模型推理、性能评估、后处理优化等细节进行详细说明，并给出C++和Python分别进行模型部署推理的代码实例

阅读本系列，您可以了解到

* Jeston Nano计算平台以及相关软件资源介绍

* 什么是TensorRT？什么是TensorRT引擎？如何将Tensorflow等深度学习框架训练好的模型转换为TensorRT引擎？转换过程需要注意什么？TensorRT对模型做了哪些优化？

* 如何使用TensorRT的Python/C++ API进行模型推理？如何构造输入数据并传递给推理引擎？输出数据又是什么格式，如何获取模型输出结果？

* 如何利用Nsight对TensoRT推理过程进行抓取，推理过程内部细节和耗时分布是什么样的？

* YOLOv5这种需要前处理和后处理解码的任务如何在TensorRT上进行推理？如何使用Python/C++进行图像/视频预处理、模型推理、模型解码、非极大值抑制，并最终在输出图像上绘制预测框？

* 如何使用Gstreamer并利用Nvidia提供的加速插件，对视频源(摄像头/文件)进行编解码及格式转换的加速，提高视频编解码速度

* Python版本视频推理性能瓶颈分析，如何提速？利用pycuda、cupy、numba等加速包进行加速，可行吗？

* C++版本如何利用libtorch进行后处理解码

* 影响性能的因素



# Performance

目前最快的方案是使用Python程序，利用pycuda进行sigmoid运算加速，fp16量化，yolov5n模型，最快fps为11

以下表格是目前不同模型规模、不同精度和不同类别数量情况下，推理引擎大小、吞吐量、延迟和进行视频检测的帧率，重点关注帧率

| Model   | Classes | Params  | EngineSize | Precision | Throughput(qps) | Latency(ms) | Program      | FPS   |
| ------- | ------- | ------- | ---------- | --------- | --------------- | ----------- | ------------ | ----- |
| YOLOv5m | 20      |         | 146M       | fp32      | 3.5738          | 279.722     | Python       | 2.77  |
| YOLOv5m | 20      |         | 70M        | fp16      |                 |             | Python       |       |
| YOLOv5s | 20      |         | 44M        | fp32      | 8.67145         | 115.22      | Python       | 5.21  |
| YOLOv5s | 20      |         | 20M        | fp16      | 72.2289         | 72.2289     | Python       | 6.66  |
| YOLOv5n | 20      | 1800481 | 13M        | fp32      | 21.8709         | 45.4301     | Python       | 9.85  |
| YOLOv5n | 20      | 1800481 | 7M         | fp16      | 28.4845         | 35.0919     | Python+CUDA  | 10.88 |
| YOLOv5n | 80      | 1881661 | 14M        | fp32      | 19.8624         | 50.3083     | Python       | 5.62  |
| YOLOv5n | 80      | 1881661 | 7M         | fp16      | 26.0428         | 38.2188     | Python       | 5.85  |
| YOLOv5n | 20      | 1800481 | 7M         | fp16      | 28.4845         | 35.0919     | C++&libtorch | 8.9   |

* Model：表示YOLOv5的模型规模

* Classes：模型支持的分类类别数，训练时确定

* Params：模型在Tensorflow summary中输出的总参数数量

* EngineSize：模型转换为TensorRT引擎后的引擎文件大小

* Precision：模型转换为TensorRT引擎的转换精度

* Throughput：模型转换时TensorRT评估推理吞吐量，每秒可执行的推理次数

* Latency：模型转换时TensorRT评估推理延迟，即推理一次需要的时长

* Program：分别用Python实现还是C++实现的检测程序

* FPS：检测程序统计的平均帧率
