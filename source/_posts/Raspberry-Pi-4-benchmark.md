---
title: Raspberry-Pi-4-benchmark
date: 2019-07-23 18:37:14
tags: [performance, benchmark, Raspberry Pi]
categories:
- 嵌入式
- 玩
- 树莓派
comments: true
---

tom's HARDWARE 发布了一篇关于树莓派4的[benchmark](https://www.tomshardware.com/reviews/raspberry-pi-4-b,6193.html)文章，详细介绍了对其多项测试方法和结果

## CPU
### Linpack Benchmark
CPU性能测试使用了Linpack Benchmark，
Linkpack Benchmark 是对计算机浮点执行率的度量。它是通过运行一个求解密集线性方程组的计算机程序来确定的

[Linkpark Q&A](chrome-extension://ecabifbgmdmgdllomnfinbmaellmclnh/data/reader/index.html?id=121)

### Sysbench CPU test
sysbench的cpu测试是在指定时间内，循环进行素数计算

## Memory
内存是做吞吐量测试，读写1M的内存块，看每秒的读写速度

## 文件压缩
分别查看单进程和多进程下的压缩速度

## GPU
### openArena Benchmark
openArena是一款需要GPU渲染的游戏，通过固定分辨率下查看平均帧率，来评估GPU性能

## Storage
查看吞吐量，每秒的读写能力

## Networking
### Ethernet
吞吐量测试
### Wi-Fi
吞吐量测试

## Power
### Power Draw benchmark
分别看负载和空闲状态下的功耗

### Thermal Throttling Benchmark

## 4K
### FFmpeg Test

## Web Surfing
### Jetstream 1.1

## Web Hosting

## Machine Learning

## Compiling Code
