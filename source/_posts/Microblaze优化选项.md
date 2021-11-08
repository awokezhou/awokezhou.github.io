---
title: Microblaze优化选项
date: 2021-11-04 17:21:12
tags: Xilinx
categories:
- 嵌入式
comments: true
---

Microblaze提供了一些优化选项，正确理解这些选项的含义以及作用对于开发过程有很多帮助，本文主要总结介绍Microblaze Configuration Wizard中的选项内容

## 预定义配置

根据具体使用场景的不同，Microblaze提供了一些预定义的配置选项供用户选择

* Microcontroller Preset

* Real-time Preset

* Application Preset

* Minimum Area

* Maximum Performance

* Maximum Frequency

* Linux with MMU

* Low-end Linux with MMU

* Typical

* Frequency Optimized

这些选项实际上是Microblaze在频率、面积、性能这几个指标不同侧重情况下，对各个配置项的组合。选择Current Settings即为自定义模式

## Implementation Optimization

该选项可选以下3种：

* Performance

* Area

* Frequency

这个选项非常重要，它与Microblaze的流水线级数对应

* 3级流水线：Area

* 5级流水线：Performance

* 8级流水线：Frequency

### 三级流水线

三级流水线对应Area，使用了最小化的硬件花费，只有取址(Fetch)、译码(Decode)和执行(Execute)

![](Microblaze优化选项\images\01.PNG)

三级流水线没有数据阻塞，只有控制流程阻塞、多指令结构性阻塞和访问较慢的内存、从较慢的内存取址等情况。多周期的指令类别有桶形移位器(barrel shift)、乘法器(multiply)、除法器(divide)和浮点指令

### 五级流水线

五级流水线对应Performance，最大化性能考量，包括取址(Fetch，IF)、译码(Decode OF)、执行(Execute，EX)、内存访问(Access Memory，MEM)和写回(Writeback，WB)

![](Microblaze优化选项\images\02.PNG)

五级流水线存在以下两种数据阻塞的情况

* OF指令需要EX指令的结果作为源操作数。EX指令类别为加载、存储、桶形移位器、乘法器、触发器和浮点运算。这些会导致1-2周期的阻塞

* OF指令需要MEM指令的结果作为源操作数。MEM指令类别包括加载、乘法器和浮点运算。这些会导致1个周期的阻塞

多周期指令有除法器和浮点运算

### 八级流水线

八级流水线对应Frequency，用于最大化频率，包括取址(Fetch，IF)、译码(Decode OF)、执行(Execute，EX)、内存访问0(Access Memory 0，M0)、内存访问1(Access Memory 1，M1)、内存访问2(Access Memory 2，M2)、内存访问3(Access Memory 3，M3)和写回(Writeback，WB)

![](Microblaze优化选项\images\03.PNG)

八级流水线存在以下四种数据阻塞的情况

* OF指令需要EX指令的结果作为源操作数。EX包括加载、存储、桶形移位器、乘法器、除法器和浮点运算，会导致1-5个周期的阻塞

* OF指令需要M0指令的结果作为源操作数。M0包括加载、乘法器、除法器和浮点运算，会导致1-4周期的阻塞

* OF指令需要M1或M2指令的结果作为源操作数。M1或M2包括加载、除法器和浮点运算，会导致1-3或1-2周期的阻塞

* OF指令需要M3指令的结果作为源操作数。M3包括加载和浮点运算，会导致1周期的阻塞

在额外的多周期指令种，存在3种情况的结构性阻塞

* OF中的指令是流指令，EX中的指令是流、加载、存储、除法或浮点指令，并实现了相应的异常，这导致一个1周期的阻塞

* OF中的指令是流指令，M0、M1、M2或M3中的指令是装载、存储、除法或浮点指令，并实现了相应的异常，这导致一个1周期的阻塞

* M0中的指令是加载或存储指令，M1、M2或M3中的指令是加载、存储、除法或浮点指令，并实现了相应的异常，这导致一个1周期的阻塞

多周期指令分为分割指令和浮点指令FDIV, FINT和FSQRT

## Use Instruction and Data Caches

使用外部存储器时，激活高速缓存，可以显著提高性能，可以降低外部慢速设备访问的使用量

## Enable Barrel Shifter

使能硬件桶形移位器(Barrel Shifter)，可以提高程序在移位操作时的性能。当该选项使能时，编译器可以自动的选择使用`bsrl`、`bsra`、`bsll`、`bsrli`、`bsrai`和`bslli`等汇编指令来优化加速移位操作

## Enable Floating Point Unit

浮点运算单元能够提升`float`类型数据进行运算时的效率，Microblaze的FPU遵循了IEEE 754-1985标准，支持加、减、乘、除、比较、转换和平方根运算。编译器会自动根据系统选择的FPU类型使用汇编浮点指令优化浮点运算

## Enable Integer Multiplier

使用一个硬件乘法器，可以提升程序在乘法运算时的效率

## Enable Integer Divider

使能整型除法器，可以使用idiv和idivu指令，提升除法运算效率

## Enable Additional Machine Status Registers Instructions

使能MSR寄存器指令`msrset`和`msrclr`，用于设置和清MSR的位。MSR包含了处理器的控制和状态位，读取该寄存器时bit[29]会被复制到bit[0]作为近进位复制。对MSR进行读写有两种方式，一种是使用`MFS`、`MTS`指令，另一种是使用`msrset`和`msrclr`。当使用`msrset`和`msrclr`进行写时，进位立即生效，其余位在一个时钟周期后生效。当使用`MTS`写时，所有位都在一个时钟周期后生效。程序运行会非常频繁的使用MSR，因此使能该选项可以很大程度的提升性能

## Enable Pattern Comparator

使能模式比较器，可以使用`pcmpbf`、`pcmpeq`和`pcmpne`指令，提升程序在进行比较时的性能。编译器自动进行指令转换

## Enable Reversed Load/Store and Swap Instructions

启用反向加载/存储和交换指令，可以使用`lbur`、`lhur`、`lwr`、`sbr`、`shr`、`swr`、`swapb`和`swawph`指令。反向加载/存储指令可以以相反的字节顺序读写数据，交换指令可以在寄存器中交换字节和字。这些指令在处理网络字节序(大端)和Microblaze字节序(小端)

时可以提升性能
