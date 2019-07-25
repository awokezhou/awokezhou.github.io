---
title: Raspberry-Pi-4-CPU
date: 2019-07-25 10:52:08
tags: [Raspberry Pi]
categories:
- 嵌入式
- 玩
- 树莓派
comments: true
---

本文汇总所有和树莓派4代CPU相关材料。树莓派4使用的CPU是博通公司的BCM2711，该芯片的datasheet无论是在博通官网、树莓派官网、google上都搜索不到，可能是还未公开

## Reference
树莓派官网发布的初步[datasheet](/download/Raspberry-Pi-4-CPU/rpi_DATA_2711_1p0_preliminary.pdf)

## CPU Features
* **processor**:  Broadcom BCM2711, Cortex-A72 (ARM v8), quad-core, 64-bit SoC @ 1.5GHz
* **Memory**： 1GB, 2GB or 4GB LPDDR4
* **Wi-Fi**: 2.4G/5G, IEEE 802.11b/g/n/ac WLAN
* **Ethernet**: GE
* **Bluetooth**: 5.0 with BLE
* **USB**: 2×USB3.0, 2×USB2.0
* **GPIO**: 40-pin, support multiplex(UART、I2C、SPI、SDIO、DPI、PCM、PWM、GPCLK)
* **Video & sound**:
    * 2×micro HDMI ports(up to 4Kp60 supported)
    * 2-lane MIPI DSI display port
    * 2-lane MIPI CSI camera port
    * 4-pole stereo audio and composite video port
* **Multimedia**: H.265(4Kp60 decode), H.264(1080p60 decode, 1080p30 encode), OpenGL ES, 3.0 graphics
* **SD Card**: 1×
* **Input power**:
    * 5V DC via USB-C connector(min 3A)
    * 5V DC via GPIO header (min 3A)
    * PoE
