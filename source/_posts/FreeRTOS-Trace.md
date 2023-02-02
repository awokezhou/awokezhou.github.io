---
title: FreeRTOS+Trace
date: 2022-04-11 13:36:23
tags: [FreeRTOS]
categories:
- CS
- FreeRTOS
mathjax: true
---

# 1. 概述

多任务的实时操作系统例如 FreeRTOS，为了满足实时性的要求，在任务的时间片划分、切换等方面做了非常严格和复杂的控制。当应用场景中多任务的功能、时序、交互较为复杂时，要分析系统整体的运行情况是一件非常棘手的事情。由于多任务操作系统存在切换代码堆栈空间的操作，因此通过各种调试器的断点、单步等调试手段无法进行函数执行跟踪，也无法进行实时性调试。通过调用系统 API 获取任务各项状态并打印输出的方式也无法保证实时性。通常的调试手段是将系统运行各种信息暂存在 RAM 中，并在需要的时刻导出为文件 (dump)，并通过一个可视化软件离线进行分析

FreeRTOS 提供了 FreeRTOS+Trace 的跟踪调试方案，该方案是 FreeRTOS 官方合作商 Percepio 开发的一种运行时诊断和优化工具，可以捕获系统运行时有价值的信息，然后在可视化的图形界面中展示这些信息，界面支持多种类型的视图，且视图间支持同步联动。在分析、排除故障或优化 FreeRTOS 应用程序性能时，这种调试方案是非常有用的

本文对FreeRTOS+Trace的调试方案进行一个基本的介绍，通过阅读本文，您可以了解到以下内容

* 使用FreeRTOS+Trace需要准备什么环境，如何下载及安装相关工具

* FreeRTOS+Trace是如何工作的

* 如何导出跟踪数据并用可视化工具打开进行分析

* 如何衡量RAM占用与记录时长的关系，如何进行有效的设置以减少RAM占用量

* 如何在Xilinx Microblaze 中使用FreeRTOS+Trace方案

# 2. Tracealyzer

Percepio 提供了一个桌面端应用程序 Tracealyzer，用于在 PC 机上对导出后的跟踪数据进行可视化分析。同时安装 Tracealyzer 后，在其安装路径下也会携带一个需要在目标平台编译构建的跟踪库 TraceRecorder

## 2.1 下载安装

Tracealyzer 的下载地址为https://percepio.com/downloadform/，由于 Percepio 提供的 Tracealyzer 支持多种 RTOS，因此在下载时需要注意选择目标系统为 FreeRTOS，输入相关信息就可以下载

{% asset_img FreeRTOS+Trace-01.PNG %}

下载后双击 Tracealyzer­x.y.z­windows64.exe 文件即可安装。本文下载使用的版本为 4.5.1

{% asset_img FreeRTOS+Trace-02.PNG %}

## 2.2 破解

由于从官方下载的 Tracealyzer 需要 License，下载时官方只提供 10 天的临时许可，到期后需要购买套餐才可以使用。为了能够长期使用，需要对软件进行破解。破解 Tracealyzer 需要用到一个压缩包工具“tracealyzer 破解工具.rar”，解压后包括两个工具： de4dot 和dnSpy

### 2.2.1 安装de4dot对Tracealyer.exe进行预处理

安装 de4dot.exe， PowerShell 或 cmd 进入 de4dot 安装路径下，并按以下命令执行

```shell
cd D:/software-setup/de4dot
./de4dot.exe -r "D:/software−setup/Tracealyzer/Tracealyzer 4"
```

运行截图如下

{% asset_img FreeRTOS+Trace-03.PNG %}

### 2.2.2 安装dnSpy对Tracealyzer.exe进行反汇编

解压dnSpy-net472.zip得到如下内容，双击打开dnSpy.exe

{% asset_img FreeRTOS+Trace-04.PNG %}

{% asset_img FreeRTOS+Trace-05.PNG %}

进入 Tracealyzer 安装路径下，并将 Tracealyzer.exe 拖入到 dnSpy 运行界面左侧

{% asset_img FreeRTOS+Trace-06.PNG %}

注意后面几处截图来源并非操作截图，而是来源于网络，因为破解后再次打开 dnSpy后显示与未破解前已经不同

展开 Tracealyzer→{}→auh，在显示的代码中搜索 bytes

{% asset_img FreeRTOS+Trace-07.PNG %}

在auj上右键，选择编辑IL指令，弹出一个新的页面，找到brtrue

{% asset_img FreeRTOS+Trace-08.PNG %}

将brtrue以上4行都修改为nop，brtrue改为br，点击确定，退出关闭dnSpy

退出时会弹出以下界面，选择确定

{% asset_img FreeRTOS+Trace-09.PNG %}

如果破解了再打开Tracealyzer仍然提示License，则修改C:/ProgramData/TracealyzerData/License.xml文件中的ExpiresOn

{% asset_img FreeRTOS+Trace-10.PNG %}

以后再次打开Tracealyzer就不会要求License了

## 2.3 TraceRecorder

Tracealyzer 的使用需要一个记录器库 TraceRecorder， TraceRecorder 需要集成到目标平台中编译构建。 TraceRecorder 由目标平台的 RTOS 内核进行调用，以存储记录重要的事件，如上下文切换和 RTOS API 调用等。用户也可以创建自定义的事件，并通过手动调用记录器库提供的 API 来存储“用户事件”，允许在目标系统中记录任何软件事件或数据，而不需要要额外的支持。将事件数据传输到主机上的Tracealyzer 应用程序进行可视化，传输方式可以是流模式 (streaming)，也可以是快照模式 (snapshot)

{% asset_img FreeRTOS+Trace-11.PNG %}

* 流模式(streaming)
  
  跟踪数据被连续地传输到主机 PC，从而允许长时间跟踪，跟踪时长取决于主机能够使用的 RAM 和磁盘空间。这支持通过调试探针如 IAR I­Jet, Keil ULINKpro和 SEGGER J­Link 流，但也可以利用系统中的其他接口，如 USB, TCP/IP, SPI，高速 UARTs，或设备文件系统

* 快照模式(snapshot)
  
  跟踪数据保存在目标平台的 RAM 中，允许通过保存 RAM 缓冲区的内容在任何时候进行“快照”。快照模式对内存效率进行了高度优化，结果的数据速率通常仅为10­20 KB/s，具体取决于系统。几个 KB 的跟踪缓冲区通常就足以对最近的事件进行适当的跟踪

本文主要对快照模式的使用进行说明

# 3. 在FreeRTOS上使用TraceRecorder

## 3.1 FreeRTOS Trace原理

Tracealyzer 本身只提供了图形化界面的显示以及从 TraceRecorder 接收事件的形式， TraceRecorder 提供的是目标平台以固定形式记录事件的能力，但是具体要记录什么事件，以及何时进行记录，是由目标平台来定义的。在 FreeRTOS 中，在很多关键代码中插入了形如 traceXXX 的宏函数，这些宏函数被作为一个插槽可以被第三方库来使用，例如下图

{% asset_img FreeRTOS+Trace-12.PNG %}

在 FreeRTOS 的 systick 中，通过 traceINCREASE_TICK_COUNT 来跟踪系统节拍事件。这些宏函数只是定义了一个事件触发机制以及事件携带什么参数，在 FreeRTOS 的 FreeRTOS.h 头文件中，都定义为了空函数，即如果外部未定义，则所有的跟踪宏函数不执行任何操作

{% asset_img FreeRTOS+Trace-13.PNG %}

以下列举了一些常用的跟踪宏函数

| 宏定义                            | 含义                                  |
| ------------------------------ | ----------------------------------- |
| traceTASK_SWITCHED_IN          | 一个任务被选中进入运行态之前                      |
| traceINCREASE_TICK_COUNT       | systick 计数                          |
| traceLOW_POWER_IDLE_BEGIN      | 在进入 tickless idle 之前                |
| traceLOW_POWER_IDLE_END        | 在 tickless idle 之后                  |
| traceTASK_SWITCHED_OUT         | 一个任务从运行态转换到 Not Running             |
| traceTASK_PRIORITY_INHERIT     | 当任务试图获取已由低优先级任务持有的互斥锁时调用            |
| traceTASK_PRIORITY_DISINHERIT  | 当任务释放互斥锁时调用，保持互斥锁会导致任务继承更高优先级任务的优先级 |
| traceBLOCKING_ON_QUEUE_RECEIVE | 任务因为无法从队列/互斥体/信号量中读取而阻塞             |
| traceBLOCKING_ON_QUEUE_PEEK    | 任务因为无法从队列/互斥体/信号量中读取而阻塞             |
| traceBLOCKING_ON_QUEUE_SEND    | 任务因为无法从队列/互斥体/信号量中写入而阻塞             |
| traceMOVED_TASK_TO_READY_STATE | 任务转为 Ready 状态时                      |

用户层面或者第三方库可以利用这些宏函数插槽插入具体的处理函数覆盖 FreeRTOS.h 中原本的空定义，以对具体事件进行捕获处理，启用 FreeRTOS 的 Trace 功能，需要在 FreeRTOSConfig.h 中配置 configUSE_TRACE_FACILITY 为 1

## 3.2 TraceRecorder库

在 Tracealyzer 安装路径 Tracealyzer 4/FreeRTOS/TraceRecorder 文件夹下，是针对 FreeRTOS 的 TraceRecorder 库源文件，在 FreeRTOS 中使用TraceRecroder，需要将其中部分文件添加到目标平台的工程中，对其进行一些配置，然后与 FreeRTOS 共同编译。另外， FreeRTOS 官方发布的源码中，在 FreeRTOS+Plus中的 FreeRTOS­Plus­Trace 也是 TraceRecroder 源码，使用这个源码也可以

### 3.2.1 TraceRecorder文件结构

TraceRecorder 文件结构如图所示

{% asset_img FreeRTOS+Trace-14.png %}

以下对关键文件进行介绍

* config/trcConfig.h
  
  该文件是对 TraceRecorder 库主要和整体层面的一个配置文件，包含了记录模式选择 (流/快照)、 FreeRTOS 哪些事件被记录等。该文件必须包含在目标工程中

* config/trcSnapshotConfig.h
  
  该文件是快照模式下的一些配置选项，包括可记录事件数量等。当使用快照模式时，该文件需要包含在目标平台工程中

* config/trcStreamingConfig.h
  
  该文件是流模式下的一些配置选项，包括定义流模式、读写模式等。当使用流模式时，该文件需要包含在目标平台工程中

* include/trcHardwarePort.h
  
  该文件是与目标平台相关定义，必须被包含在目标平台工程中，且在适配目标平台时需要对其进行修改

* include/trcKernelPort.h
  
  该文件中定义了 TraceRecorder 的核心功能函数，对 FreeRTOS 跟踪函数的重定义也在其中，必须被包含在目标平台工程中

* include/trcPortDefines.h
  
  该文件中是一些公共定义，必须被包含在目标平台工程中

* include/trcRecorder.h
  
  该文件中定义了 TraceRecorder 的对外 API，必须被包含在目标平台工程中

* trcKernelPort.c
  
  与 trcKernelPort.h 对应的跟踪函数实现，必须被包含在目标平台工程中

* trcSnapshotRecorder.c
  
  快照相关功能实现，快照模式需要包含在目标平台工程中

* trcStreamingRecorder.c
  
  流模式相关功能实现，流模式需要包含在目标平台工程中

### 3.2.2 TraceRecorder原理

用户使用 TraceRecorder 对 FreeRTOS 进行跟踪，需要在初始化时显式的调用vTraceEnable() 对 TraceRecorder 库进行初始化，在 TraceRecorder 的初始化过程中会在内存中创建 RecorderDataType 数据结构，所有的 FreeRTOS 事件以及用户自定义事件都是记录到这个 RecorderDataType 结构中。当用户需要导出记录数据时，本质上是将 RecorderDataType 所在的整个 RAM 空间导出

RecorderDataType 的数据结构如图 3.4 所示，开头和结尾的固定掩码 marker用于 Tracealyzer 从一段二进制数据中定位 RecorderDataType 的位置。RecorderDataType 结构中最为主要的 3 个成员分别是 ObjectPropertyTable、 SymbolTable 和 eventData。 ObjectPropertyTable 以 Class/Object 的抽象形式记录管理 FreeRTOS 中的 9 种类型资源信息，例如任务、中断、队列、信号量等，并将这些资源的名称字符串、状态、计数等存放于一段连续内存中。 SymbolTable 主要用于存放用户自定义事件的名称字符串等信息。 eventData 用于存放每一个独立的事件信息

{% asset_img FreeRTOS+Trace-15.png %}

由于 RecorderDataType 中各信息或事件都是以连续内存 BytePool 的形式连续存放，并由另外一些变量存放数据的索引，因此整个 RecorderDataType 在内存中是连续且统一的，导出到 Tracealyzer 中可以只根据相对位置解析各部分信息

#### ObjectPropertyTable

ObjectPropertyTable 主要用于记录 FreeRTOS 中内核资源相关信息，在头文件 trcKernelPort.h 中定义了 9 种内核资源的类别

{% asset_img FreeRTOS+Trace-16.png %}

ObjectPropertyTable 抽象了 Class/Object 的概念，即对所关心的 Task、Queue、 Semaphore 等划分了类别，而具体记录的数据是每种类别的实例化对象 Object。每种 Class 记录多少个 Object 是由用户决定的 (在配置文件中设置，见后文)，所有 Objects 的信息最终都存放在 ObjectPropertyTable 里的 Objbytes 结构中，该结构以 BytePool 的形式记录存放所有内核资源对象，连续排列。每种 Object记录的内容如下

* Queue
  
  队列名称和当前该队列中 message 的数量

* Semaphore
  
  信号量名称和当前该信号量的状态， 0:cleared， 1:signaled

* Mutex
  
  互斥量名称和当前拥有该互斥量的任务句柄， 0:free

* Task
  
  任务名称和当前任务优先级以及任务状态

* ISR
  
  中断名称和当前中断优先级、中断状态

* Timer
  
  定时器名称和当前定时器状态

* EventGroup
  
  事件组名称

* StreamBuffer
  
  StreamBuffer 名称

* MessageBuffer
  
  MessageBuffer 名称

这样设计的好处是所有内核对象的名称信息仅在 ObjectPropertyTable 存在一份，在事件记录时不需要额外存储对象名称，而只需要使用一个标号来代表哪个对象即可

#### SymbolTable

SymbolTable 使用了和 ObjectPropertyTable 类似的设计方法，字符串名称和符号等仅存一份，区别是 SymbolTable 记录的是用户自定义事件。 TraceRecorder提供了 API 可以让用户创建自定义的事件，用户创建的不同事件用标签区别，标签(Label) 也是一个字符串，与内核对象 Object 对应。 SymbolTable 中使用了哈希表结构来将不同标签映射到 BytePool 的不同位置，哈希函数是计算字符串的校验和

## 3.3 TraceRecorder配置

快照模式下，对 TraceRecorder 的配置主要关心 trcConfig.h 和 trcSnapshotConfig.h 这两个文件

trcConfig.h 中的配置项如下

* TRC_CFG_HARDWARE_PORT
  硬件平台选择宏，需要修改为目标硬件平台的类型， TraceRecorder 支持的硬件平台宏定义在 trcPortDefines.h 文件中。默认是未定义，需要手动修改

* TRC_CFG_RECORDER_MODE
  记录模式选择宏，即选择快照模式或者流模式，默认为快照模式

* TRC_RECORDER_MODE_SNAPSHOT
  为快照模式， TRC_RECORDER_MODE_STREAMING 为流模式

* TRC_CFG_FREERTOS_VERSION
  FreeRTOS 版本选择，默认未定义，需要手动修改

* TRC_CFG_SCHEDULING_ONLY
  是否仅记录调度事件，默认是 0，即所有 FreeRTOS 的 trace 都记录。为 1 时不记录用户自定义事件

* TRC_CFG_INCLUDE_MEMMANG_EVENTS
  
  是否记录内存管理事件，默认是 1，即用户或内核调用内存申请和释放事件也会记录

* TRC_CFG_INCLUDE_USER_EVENTS
  
  是否记录用户自定义事件，默认是 1

* TRC_CFG_INCLUDE_ISR_TRACING
  
  是否记录中断事件，该配置项禁止的是用户在中断服务函数中通过调用vTraceStoreISRBegin 和 vTraceStoreISREnd 进行的记录，而非 FreeRTOS 本身对中断的记录

* TRC_CFG_INCLUDE_READY_EVENTS
  
  是否记录任务进入 Ready 状态时的事件

* TRC_CFG_INCLUDE_OSTICK_EVENTS
  
  是否允许记录 OS Tick 事件

* TRC_CFG_INCLUDE_EVENT_GROUP_EVENTS
  
  是否允许记录 EventGroup 事件

* TRC_CFG_INCLUDE_TIMER_EVENTS
  
  是否允许记录Timer事件

* TRC_CFG_INCLUDE_PEND_FUNC_CALL_EVENTS
  
  是否记录任何挂起任务的调用事件

* TRC_CFG_INCLUDE_STREAM_BUFFER_EVENTS
  
  是否记录 StreamBuffer 事件

* TRC_CFG_ENABLE_STACK_MONITOR
  
  是否允许 TraceRecorder 创建一个栈监视任务 TzCtrl

* TRC_CFG_STACK_MONITOR_MAX_TASKS
  
  定义栈监视任务可监视任务最大数量

trcSnapshotConfig.h 中的配置项如下

* TRC_CFG_SNAPSHOT_MODE
  
  快照模式选择宏，TRC_SNAPSHOT_MODE_RING_BUFFER 表示跟踪数据是以 RingBuffer 的形式存放在内存中，新的数据会覆盖老的数据，记录不会停止直到人为触发系统暂停。 TRC_SNAPSHOT_MODE_STOP_WHEN_FULL 表示当记录数据满后会自动停止记录。默认是 RingBuffer 形式

* TRC_CFG_EVENT_BUFFER_SIZE
  
  表示可记录事件的数量，决定了 RecorderDataType 中 eventData 的大小

* TRC_CFG_NTASK/NISR/NQUEUE/NSEMAPHORE/NMUTEX/NTIMER...
  
  表示可记录跟踪任务、中断、队列等资源的数量，决定了 RecorderDataType 中ObjectPropertyTable 的对象数量

* TRC_CFG_INCLUDE_FLOAT_SUPPORT
  
  表示是否支持浮点格式，主要用户用户自定义事件

* TRC_CFG_SYMBOL_TABLE_SIZE
  
  表示符号表大小，主要用于用户事件记录时

* TRC_CFG_USE_SEPARATE_USER_EVENT_BUFFER
  
  用户自定义事件是否存放于单独的区域，默认是用户自定义事件与内核事件都在eventData 中

* TRC_CFG_SEPARATE_USER_EVENT_BUFFER_SIZE
  
  当用户自定义事件单独存放是的 Buffer 大小

* TRC_CFG_UB_CHANNELS
  
  用户自定义事件的通道数量

## 3.4 TraceRecorder 的使用

编译方面，将上文提到的 TraceRecorder 相关源文件添加到目标平台项目中，在 FreeRTOSConfig.h 最后引用 trcRecorder.h 头文件。代码方面，需要在系统初始化时调用 vTraceEnable(TRC_START) 初始化 TraceRecorder，然后调用uiTraceStart 记录启动记录事件，然后 FreeRTOS 运行的各种内核事件会自动向RecorderDataType 中记录，如果需要使用用户自定义事件，则通过 xTraceRegisterString 函数创建事件标签，再在需要记录的位置调用 vTracePrint 或 vTracePrintF 进行记录。当需要将记录数据导出时，只需将 RecorderDataType 所在的内存空间导出即可， TraceRecorder 库中有个全局指针 RecorderDataPtr 方便在调试时对 RecorderDataType 地址进行定位

## 3.5 Tracealyzer 的使用

导出的记录文件，通过 Tracealyzer 来打开。打开已安装的 Tracealyzer，点击导航栏 File­>Open­>Open File，选择记录文件

{% asset_img FreeRTOS+Trace-17.png %}

打开记录文件后， Tracealyzer 界面显示如下图

{% asset_img FreeRTOS+Trace-18.png %}

Tracealyzer 提供了 20 多种视图来以不同的可视化方式对同一个记录文件进行展示分析，最主要也是默认显示的是 TraceView，其余常用的视图在左侧红框标记出来的区域

### 3.5.1 TraceView

TraceView 是以垂直视图显示时间的先后顺序，最先发生的在最上方，最后发生的在最下方。视图以列的形式划分了几个事件域，比较重要的 CPU 域和 EventField，CPU 域包括 CPU 状态、当前运行的任务事件以及 TmrSvc 事件， EventField 包括各种任务事件和其他内核事件。默认情况下由于时间线比较密集，不方便查看，可以在左上角点击 +/­号来调整时间粒度，将其展开后如下图所示

{% asset_img FreeRTOS+Trace-19.png %}

### 3.5.2 EventLog

另一个比较方便的视图是 EventLog，在左侧 View 栏选择 EventLog 打开，如下图所示

{% asset_img FreeRTOS+Trace-20.png %}

EventLog 视图以日志的形式列举一条一条发生的事件，不同事件以不同颜色区分，能够比较直观的查看每一条发生的事件，以及时间上的先后关系

### Service Block Time

Service Block Time 视图可以方便的统计查看各个系统调用阻塞的情况。左侧视图面板点击下方省略号，选择 Service Blocking­>Show all objects in one graph

{% asset_img FreeRTOS+Trace-21.png %}

打开后的界面如图所示，界面以二维坐标系的形式展示，横轴是运行时间，纵轴是阻塞时间，每一个节点代表一个系统调用事件。鼠标选中一个节点事件后，在右侧的 Selection Details 中可以查看详细阻塞情况

{% asset_img FreeRTOS+Trace-22.png %}

在节点上鼠标右键，可选择 Blocking Time Graph(以单独对象查看所有阻塞情况)、Object History(以日志的形式展现该对象的所有操作)、Object Statistic(对象事件统计)

{% asset_img FreeRTOS+Trace-23.png %}

### 3.5.4 Service Intensity

Service Intensity 视图能够以类似直方图的形式展示固定周期内各事件的发生情况

{% asset_img FreeRTOS+Trace-24.png %}

选择 Service Intensity­>Create view Serive Call Intensity ­ All Objects

{% asset_img FreeRTOS+Trace-25.png %}

图中以固定宽度的时间跨度来计算统计在这个跨度内各事件的发生次数，点击左上角+/­符号可以改变统计的时间跨度。不同事件类型用不同的颜色区分，鼠标选中一个事件后，在右侧的 Selection Details 中可以查看详细情况

### 3.5.5 Overview

Overview 视图提供了对整个跟踪数据的整体描述，上方导航栏选择 View­>Trace Overview 打开

{% asset_img FreeRTOS+Trace-26.png %}

Oviewview 中展示了整个跟踪数据的记录时长、 RAM 大小、 EventBuffer 的使用情况、每种事件的记录占比、跟踪内核对象数量等信息，能够快速判断跟踪数据的资源使用情况

{% asset_img FreeRTOS+Trace-27.png %}

## 3.6 TraceRecorder 的记录时长与 RAM

TraceRecorder 将所有的记录数据都保存在 RecorderDataType 中， TraceRecorder 对 RAM 的使用就是 RecorderDataType 结构的大小。在现实使用中，通常希望明确的知道如何设置和使用 TraceRecorder，以获得充足的记录信息，而同时尽可能少的占用 RAM 空间。然而记录时间的长短与 RAM 占用多少并没有一个直接的计算公式，而是需要以不同的配置参数在目标平台上进行多组测试，并通过 Tracealyzer 的Overview 视图查看 RAM 占用情况，以获得最终的希望设置。例如当发现 Overview中 EventBuffer 的占用率只有百分之 10，那么说明浪费了很多 RAM 空间，如果认为此时记录的信息量是充足的，就可以修改配置，减小 EventBuffer 的大小

本文在 FreeRTOS 的 Linux 仿真环境下进行了一些测试，如下表所示

{% asset_img FreeRTOS+Trace-28.png %}

测试选取了 OSTick 为 1ms 和 10ms，记录时长 6、 10、 16、 20、 30 秒，不丢失事件的情况下，所占用的 RAM 空间大小。由于测试创建的任务功能较为简单，任务主动调用的系统调用频率较低，因此在 OSTick 为 1ms 的场景下， EventBuffer 中大部分的事件都是 OSTick 事件，记录 30 秒的数据量， RAM 占用在 130+KB；而 OSTick为 10ms 的场景下，记录 30 秒的数据量， RAM 占用就能够降低到 30KB

以下列举一些在 RAM 占用方面可以考虑的点

### 3.6.1 OSTick

当 FreeRTOS 的时钟节拍频率较高时，由于 OSTick 本身也会记录为一个事件，而一般 Task 的实际运行频率比 OSTick 要低很多，导致记录数据中 OSTick 事件占比会非常高。通常对于调试分析而言， Task 的运行、消息队列等事件信息才是有价值的，因此可以对 OSTick 事件记录进行一些配置

* 修改系统运行频率
  
  通过修改 FreeRTOSConfig.h 中的 configTICK_RATE_HZ，降低系统频率，可以有效减少 OSTick 事件的记录频率，但需注意评估是否会影响业务场景的性能和功能

* 不记录 OSTick 事件
  
  通过修改 trcConfig.h 中的 TRC_CFG_INCLUDE_OSTICK_EVENTS，可以屏蔽对 OSTick 事件的记录

### 3.6.2 EventBufferSize

最为直接有效的方式是修改 EventBufferSize， trcShapshotConfig.h 中的 TRC_CFG_EVENT_BUFFER_SIZE 直接决定 RecorderDataType 中的 eventData 大小，而 eventData 大小占 RecorderDataType 大小的比例非常高，修改TRC_CFG_EVENT_BUFFER_SIZE 的值，对整个记录数据的 RAM 占用影响很大

### 3.6.3 内核资源数量

在已知各种内核资源使用量的情况下，可以修改 trcShapshotConfig.h 中 TRC_CFG_NTASK、TRC_CFG_NISR、TRC_CFG_NQUEUE、TRC_CFG_NSEMAPHORE、TRC_CFG_NMUTEX、TRC_CFG_NTIMER、TRC_CFG_NEVENTGROUP、 TRC_CFG_NSTREAMBUFFER、 TRC_CFG_NMESSAGEBUFFER的数值，减少多余的开销

# 4. Xilinx Microblaze平台使用FreeRTOS+Trace

本章节以 Xilinx Microblaze 平台为例，介绍如何在该目标平台上使用 FreeRTOS+Trace 的快照记录功能。所使用的 Xilinx SDK 版本为 2018.2，并在 SDk 中已经创建好了一个目标平台为 Microblaze 的 FreeRTOS Hello World 模板工程，如下图所示

{% asset_img FreeRTOS+Trace-29.png %}

在 Xilinx SDK 工程中使用 FreeRTOS+Trace 有几个设置步骤，首先需要将 TraceRecorder 库中的部分源文件和头文件导入到已创建好的工程中，然后对TraceRecorder 库的一些宏定义配置进行修改，以适配 Microblaze 平台，然后在工程中设置编译依赖和头文件路径，正确的编译和链接 FreeRTOS、 TraceRecorder以及应用程序，并在应用程序中引用 TraceRecorder 的 API 以开启 TraceRecorder的记录功能，最后使用 SDK 的内存导出功能将记录数据导出为文件，使用 Tracealyzer打开展示。以下将详细介绍各个步骤

## 4.1 TraceRecorder 移植

### 4.1.1 导入TraceRecorder 到 SDK 工程

第一个步骤就是将 TraceRecorder 库导入到工程中，需要提前在本机上安装好Tracealyzer 程序。在工程的 src 目录上，鼠标右键选择 import

{% asset_img FreeRTOS+Trace-30.png %}

在弹出框中选择 General­>File System，点击 Next

{% asset_img FreeRTOS+Trace-31.png %}

点击 Browser 选择 Tracealyzer 安装路径下的“Tracealyzer 4/FreeRTOS/TraceRecorder”

{% asset_img FreeRTOS+Trace-32.png %}

点击 TraceRecorder 左侧的小三角展开路径，鼠标选中 TraceRecorder，并在右侧勾选 trcKernelPort.c、trcShapshotRecorder.c、trcStreamimgRecorder.c

{% asset_img FreeRTOS+Trace-33.png %}

左侧选中 config 文件夹，并在右侧勾选 trcConfig.h、 trcShapshotConfig.h、 trcStreamingConfig.h 文件

{% asset_img FreeRTOS+Trace-34.png %}

左侧选中 include 文件夹，并在右侧勾选全部头文件

{% asset_img FreeRTOS+Trace-35.png %}

点击 Finish，导入后的工程目录如图所示

{% asset_img FreeRTOS+Trace-36.png %}

### 4.1.2 FreeRTOSConfig.h修改

确保 BSP 的 FreeRTOSConfig.h 中 configUSER_TRACE_FACILITY 设置为1，以使能 FreeRTOS 的 Trace 功能

{% asset_img FreeRTOS+Trace-37.png %}

在 FreeRTOSConfig.h 尾部添加以下代码，使 FreeRTOS 引用 trcRecorder.h，重定义 FreeRTOS 本身的 Trace 宏函数

{% asset_img FreeRTOS+Trace-38.png %}

### 4.1.3 trcConfig.h 修改

去掉 trcConfig.h 中需要添加处理器头文件的编译检查 #error

{% asset_img FreeRTOS+Trace-39.png %}

修改硬件平台定义为 Microblaze

{% asset_img FreeRTOS+Trace-40.png %}

修改 FreeRTOS 版本为 10.0.0

{% asset_img FreeRTOS+Trace-41.png %}

### 4.1.4 trcHardwarePort.h 修改

在 trcHardwarePort.h 文件中 Microblaze 定义部分，修改 XTmrCtr_mGetLoadReg为 XTmrCtr_GetLoadReg，并在下方添加 TRACE_ENTER_CRITICAL_SECTION 和TRACE_EXIT_CRITICAL_SECTION 定义

{% asset_img FreeRTOS+Trace-42.png %}

CRITICAL_SECTION 的定义对 TraceRecorder 非常重要，为了确保代码执行不被打断，所有记录事件的过程都会进入/退出 CRITICAL_SECTION，而 TraceRecorder对 Microblaze 平台未定义 CRITICAL_SECTION，因此需要用户自定义

{% asset_img FreeRTOS+Trace-43.png %}

### 4.1.5 添加头文件路径

HelloWorld 工程右键选择 C/C++ Build Settings

{% asset_img FreeRTOS+Trace-44.png %}

在 Settings 界面，选择 Microblaze gcc assemble­>General，将导入到工程 src 中的 config 和 include 路径添加到 Include Paths 中

{% asset_img FreeRTOS+Trace-45.png %}

在 Settings 界面，选择 Microblaze gcc compiler­>Directories，将导入到工程 src 中的 config 和 include 路径添加到 Include Paths 中

{% asset_img FreeRTOS+Trace-46.png %}

## 4.2 在代码中使用 TraceRecorder

在 freertos_hello_world.c 文件中添加 trcRecorder.h 引用

{% asset_img FreeRTOS+Trace-47.png %}

在 man 函数开始调用 vTraceEnable(TRC_START) 使能并初始化 TraceRecorder，调用 uiTraceStart() 开始记录

{% asset_img FreeRTOS+Trace-48.png %}

在 TxTask 中调用 xTraceRegisterString() 和 vTracePrint() 来使用用户自定义事件

{% asset_img FreeRTOS+Trace-49.png %}

## 4.3 导出记录数据

在 Xinlinx SDK 中，可以使用 XSCT Console 来保存快照跟踪数据。按照以下步骤操作

1. 确保 debug 会话活跃，并且系统已经挂起

2. 打开 XSCT Console，找到 Xilinx menu

3. 通过 cd 命令进入一个工作路径，即要保存跟踪数据的路径

4. 输入“print RecorderDataPtr”查看跟踪数据结构的地址

5. 输入“print RecorderDataPtr­>filesize”查看跟踪数据的大小 (bytes)

6. 输入“mrd ­bin ­file trace.bin [address] [size in words]”保存跟踪数据为文件“trace.bin”

7. 从 Tracealyzer 打开“trace.bin”

# Reference

* Microblaze Trace

* Tracealyzer for FreeRTOS on Xilinx Zynq

* Tracealyzer User Manual
