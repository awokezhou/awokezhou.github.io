---
title: 'Narrowband Power Optimizations for Massive IoT: eDRX and PSM[译]'
date: 2020-04-17 10:47:50
tags: [3GPP,NB-IoT]
categories:
- 协议
- 3GPP
- NB-IoT
comments: true
---



本文翻译自twilio网站文章[Narrowband Power Optimizations for Massive IoT: eDRX and PSM]([https://www.twilio.com/docs/wireless/nb/nb-iot-power-optimizations-edrx-psm](https://www.twilio.com/docs/wireless/nb/nb-iot-power-optimizations-edrx-psm))



NB-IoT(Narrowband IoT, 窄带物联网)蜂窝技术是为低功耗设备而设计的。它包含了大量物联网设备开发人员可以使用的功能，例如位置跟踪器、低成本传感器网络、公用电表和预防性维护监视器，以将产品的电力消耗降到最低



本指南描述了其中的两个优化特性，即PSM(省电模式, Power Save Mode)和eDRX(扩展不连续接收, Extended Discontinuous Reception)，以帮助您评估它们如何为您的大规模物联网应用程序带来好处，以及如何利用它们



## PSM

通常，大多数物联网设备间歇性地发送或接收数据。在数据的发送和接收之间，可以让设备处于休眠状态，以最大限度的降低功耗，最大化电池能量



PSM是蜂窝调制解调器的一种特性，它可以关闭设备无线电并使设备进入休眠状态，而无需在下次醒来时re-attach(重新附着)到网络。虽然re-attach过程只消耗少量的能量，但是在设备的整个生命周期中，re-attach的累积能量消耗可能变得非常大。因此，如果可以避免re-attach，可使得电池寿命延长。PSM刚好提供了这一点



* PSM是一种降低无线电能量消耗的设备端机制。设备报告网络自己需要多频繁和多长时间处于活跃状态，以便传输和接收数据。然而，最终的值是由网络决定的

* PSM模式类似于断电，但是设备在网络中的状态仍然保持为已注册。当设备再次活跃时，没有必要re-attach或re-establish(重新建立)PDN(数据包数据网络, Packet Data Network)连接

* PSM特性是在3GPP Release 12 中引入的，适用于所有LTE设备类别。设备请求PSM只需在attach、TAU(跟踪区域更新, tracking area udpate)或者路由区域更新中包含一个带有所需值的计时器



![](Narrowband-Power-Optimizations-for-Massive-IoT-eDRX-and-PSM-译/image\01.png)



### PSM FAQS

#### 它如何工作？

当设备通过网络初始化PSM时，它提供两个首选计时器(T3324和T3412)；PSM时间是这两个计时器的差值(T3412减去T3324)。网络可以接受这些值，也可以设置不同的值。然后网络保留状态信息，设备保持在网络上注册。如果设备在它与网络约定的时间间隔到期之前唤醒并发送数据，则不需要re-attach过程



#### 当设备处于活跃PSM周期时，它可以接收消息吗?

不能，在活跃PSM周期时无法访问设备



#### 当设备处于活跃PSM周期时，是否可以通过NIDD(非ip数据传递, Non-IP Data Delivery)访问它?

不能，NIDD通过寻呼信道工作(paging channel)，该信道利用无线电。在活跃PSM周期中，无线电完全关闭，无法访问设备



#### 在活跃PSM周期中发送到设备的数据包会发生什么?

3GPP要求必须由网络存储数据包。建议网络操作员至少为最后一个100字节的数据包留出存储空间



#### 一个设备可以在PSM周期中停留多久？

在T-Mobile的NB-IoT网络上，一个设备可以在PSM循环中最多停留12小时



## 使用PSM

蜂窝模块必须支持PSM。可以使用设备上的AT命令来启用它

例如，如果你使用的是Quectel BG96调制解调器，你可以发送以下命令

```shell
AT+QCFG="psm/urc"[enable]

AT+QPSMTIMER: <tau_timer>,<T3324_timer>
```

不需要用户进行网络配置；在设备上启用PSM就足够了



根据GPRS Timer 3规范(见3GPP TS 24.008第[10.5.7.4a]([https://www.etsi.org/deliver/etsi_ts/124000_124099/124008/13.07.00_60/ts_124008v130700p.pdf](https://www.etsi.org/deliver/etsi_ts/124000_124099/124008/13.07.00_60/ts_124008v130700p.pdf))节)，要求的周期TAU定时器值编码如下：

第5位到第1位表示二进制编码的定时器值。第6位到第8位定义计时器的计时器值单元，如下所示

| Timer 3 value | Timer value is incremented in multiples of |
| ------------- | ------------------------------------------ |
| 000xxxxx      | 10 minutes                                 |
| 001xxxxx      | 1 hour                                     |
| 010xxxxx      | 10 hours                                   |
| 011xxxxx      | 2 seconds                                  |
| 100xxxxx      | 30 seconds                                 |
| 101xxxxx      | 1 minute                                   |
| 110xxxxx      | 320 hours*                                 |
| 111xxxxx      | Timer is deactivated                       |

> 参见3GPP [TS 24.008]([https://www.etsi.org/deliver/etsi_ts/124000_124099/124008/13.07.00_60/ts_124008v130700p.pdf](https://www.etsi.org/deliver/etsi_ts/124000_124099/124008/13.07.00_60/ts_124008v130700p.pdf))规范中的说明，表10.5.163a提供了关于这个值的更多信息



请求的活跃时间是由GPRS Timer 2规范的octet 3定义的一个二进制字符串字节值(见3GPP [TS 24.008]([https://www.etsi.org/deliver/etsi_ts/124000_124099/124008/13.07.00_60/ts_124008v130700p.pdf]](https://www.etsi.org/deliver/etsi_ts/124000_124099/124008/13.07.00_60/ts_124008v130700p.pdf%5D)([https://www.etsi.org/deliver/etsi_ts/124000_124099/124008/13.07.00_60/ts_124008v130700p.pdf](https://www.etsi.org/deliver/etsi_ts/124000_124099/124008/13.07.00_60/ts_124008v130700p.pdf))的10.5.7.4节)，如下所示：

bi5到1表示二进制编码计时器值。第6到8位定义计时器的定时器如下

| Timer 3 value | Timer value is incremented in multiples of |
| ------------- | ------------------------------------------ |
| 000xxxxx      | 2 seconds                                  |
| 001xxxxx      | 1 minute                                   |
| 010xxxxx      | 1 decihour (6 minutes)                     |
| 111xxxxx      | Timer is deactivated                       |

#### 例子

```shell
AT+CPSMS=1,,,"01000011","01000011"
```

以上命令使得PSM周期TAU值变为30小时，请求的活动时间为18分钟



```shell
AT+CPSMS=0
```

以上命令禁用PSM



## eDRX

eDRX是现有LTE功能的扩展，可以被物联网设备用来降低功耗。eDRX可以在没有PSM的情况下使用，也可以与PSM结合使用，以获得额外的电能节省



eDRX允许大大扩展设备不监听网络的时间间隔。对于一个大规模的物联网应用程序，设备在几秒钟或更长时间内无法访问是完全可以接受的。虽然eDX不能提供与PSM相同的功耗降低级别，但它可以在设备可达性和某些应用程序的功耗之间提供一个很好的折衷。网络和设备在设备可以睡眠时进行协商。设备在规定的周期内保持其接收电路关闭，在此期间，设备不侦听寻呼或下行控制信道。当设备醒来时，接收器将监听物理控制通道



![](Narrowband-Power-Optimizations-for-Massive-IoT-eDRX-and-PSM-译/image/02.png)

eDRX只允许在一定期限内使用；以下列出了这些项目：

* 20.48 seconds

* 40.96 seconds

* 81.92 seconds (~1 minute)

* 163.84 seconds (~ 3 min)

* 327.68 seconds (~ 5 min)

* 655.36 seconds (~ 11 min)

* 1310.72 seconds (~22 min)

* 2621.44 seconds (~44 min)

* 5242.88 seconds (~87 min)

* 10485.76 seconds (~175 min)

### 使用eDRX

eDRX的支持因运营商而异。设备向网络请求一个给定的eDRX周期；网络使用实际使用的eDRX周期和PTW(巡护时间窗口)应答



启用eDRX不会对模块发送数据的能力产生负面影响，但是会关闭对所配置的周期间隔的接收，从而节约电能



T-Mobile支持所有文档(见表10.5.5.32 3GPP TS 24.008)值的窄带eDRX循环，NB-S1模式，范围从~20s (20.48s)到~3hrs (10485.76s)。值由表单的位串表示：

|        |                     |
| ------ | ------------------- |
| "0010" | 20.48s              |
| "0011" | 40.96s              |
| "0101" | 81.92s              |
| "1001" | 163.84s             |
| "1010" | 327.68s             |
| "1011" | 655.36s             |
| "1100" | 1310.72s            |
| "1101" | 2621.44s            |
| "1101" | 2621.44s            |
| "1110" | 5242.88s            |
| "1111" | 10485.76s (~175min) |


根据您的产品使用的NB-IoT模块，请求的eDRX周期可以存储在非易失性内存中，并在会话之间保持。模块通常支持禁用eDRX和一个后续的AT命令，以及一个恢复默认值的选项



要在u-blox SARA-N410-02b和Quectel BG96上为NB模块配置eDRX，可以使用AT+CEDRXS命令，如下所示：

```shell
AT+CEDRXS=2,5,"1001"
```

这要求在窄带(5)的URC反馈(2)和163.84s(“1001”)的eDRX循环时间下启用eDRX



如果配置了URC，网络将通过重复请求的周期间隔以及实际有效的周期间隔和网络指定的PTW时间来响应。上述命令的一个URC示例是

```shell
+CEDRXP: [5,"1001","1001","0111"]
```

为NB-S1返回的PTW值对应于以下时间：

|        |        |
| ------ | ------ |
| "0000" | 2.56s  |
| "0001" | 5.12s  |
| "0010" | 7.68s  |
| "0011" | 10.24s |
| "0100" | 12.8s  |
| "0101" | 15.36s |
| "0110" | 17.92s |
| "0111" | 20.48s |
| "1000" | 23.04s |
| "1001" | 25.6s  |
| "1010" | 28.16s |
| "1011" | 30.72s |
| "1100" | 33.28s |
| "1101" | 35.84s |
| "1110" | 38.4s  |
| "1111" | 40.96s |

以下命令用于禁用eDRX的窄带

```shell
AT+CEDRXS=0,5
```

eDRX被0标记，而窄带被5标记
