---
title: VitisAI-05-Vitis Flow
date: 2022-07-06 21:15:27
tags: [Xilinx, DPU, Vitis AI, PetaLinux]
categories:
- AI
- VitisAI
---

本文承接VitisAI-04-PetaLinux Flow，介绍使用Xilinx的Vitis工具利用Vivado生成的design_1_wrapper.xsa文件以及PetaLinux编译的rootfs和内核镜像，生成制作好的SD卡镜像文件sd_card.img

# PetaLinux工程准备编译好的各文件

PetaLinux编译成功后的输出文件在PetaLinux工程的image/linux下，进入此目录，并在其中创建pfm文件夹，并创建boot和sd_dir两个子目录，将Vitis工程需要的PetaLinux输出文件拷贝到新创建的文件夹下

```shell
cd image/linux
mkdir pfm
mkdir pfm/boot
mkdir pfm/sd_dir
```

拷贝以下文件到pfm/boot

* bl31.elf

* pmufw.elf

* zynqmp_fsbl.elf

* u-boot.elf

* system.dtb

```shell
cp bl31.elf pmufw.elf zynqmp_fsbl.elf u-boot.elf system.dtb pfm/boot
```

拷贝以下文件到pfm/sd_dir

* boot.scr

* system.dtb

```shell
cp boot.scr system.dtb pfm/sd_dir
```

# 创建Vitis Platform Project

运行Vitis环境变量脚本

```shell
source ~/opt/pkg/xilinx/Vitis/2021.1/settings64.sh
```

在dpu_custom下创建dpu_vitis文件夹，并进入，打开vitis

```shell
mkdir dpu_vitis
cd dpu_vitis
vitis
```

此时的各项目文件夹路径关系为

```shell
|--dpu_custom
  |--dpu_vivado
  |--dpu_plnx
  |--dpu_vitis
```

选择workspace为dpu_vitis，选择File->New->Platform Project，创建一个平台项目，设置项目名为dpu_base，点击Next，选择dpu_vivado中的.xsa硬件描述文件，Operating system选linux，取消勾选Generate boot components，点击Finish

{% asset_img 01.png %}

选中linux on psu_cortexa53，在Bif File处点击下三角生成Bif File文件，Boot Components Directory选择PetaLinux项目中创建的pfm/boot，FAT32 Partition Directory选择pfm/sd_dir

右键工程编译工程，该编译过程很快(1分钟以内)

# Vitis安装Vitis AI

## 克隆Vitis AI仓库到本地

在Vitis中添加DPU需要预先将Vitis AI仓库克隆到本地，使用如下命令

```shell
git clone https://github.com/Xilinx/Vitis-AI
```

## 安装Vitis AI到Vitis

在Vitis中选择Windows→Preferences

{% asset_img 02.png %}

点击Add添加一个库

{% asset_img 03.png %}

ID设置为vitis ai，Name设置为Vitis AI，Location设置为Vitis AI仓库路径

{% asset_img 04.png %}

点击Apply and Close

# 安装交叉编译环境sdk

点击[sdk-2021.2.0.0.sh](https://www.xilinx.com/bin/public/openDownload?filename=sdk-2021.2.0.0.sh)下载该sdk，运行以下命令，将其安装到PetaLinux路径下

```shell
./sdk-2021.2.0.0.sh
```

安装完成后PetaLinux安装路径下会出现environment-setup-cortexa72-cortexa53-xilinx-linux的文件

# 创建Vitis Application 工程

File→New→Application Project，点击Next，上文创建的platform工程自动出现在选项中，选中并点击Next

{% asset_img 05.png %}

设置项目名为dpu_trd，点击Next

{% asset_img 06.png %}

设置sysroot path为安装的交叉编译链位置：~/opt/pkg/petalinux/2021.2/sysroots/cortexa72-cortexa53-xilinx-linux，Root FS为dpu_plnx/image/linux/rootfs.ext4，Kernel Image为dpu_plnx/image/linux/image

{% asset_img 07.png %}

点击Next，在左侧选中dsa→DPU Kernel(RTL Kernel)，点击Finish(必须在Vitis中成功安装了VitisAi仓库，这一步中才会出现DPU Kernel选项)

{% asset_img 08.png %}

将Emulation-SW修改为Hardware

{% asset_img 09.png %}

选择dpu_trd_system→dpu_trd_kernel→src→prj→Vitsi→dpi_conf.vh，将B4096改为B1024，之所以改为B1024是因为本文实验环境使用的FPGA芯片资源有限，只能选择B1024的DPU，这里的B1024和B4096是DPU不同架构配置，越大的数字代表了越高的并行度和计算性能，但同时占用更多片上资源(LUT、RAM、DSP)，关于DPU相关的配置参数和资源占用，包括dpi_conf.vh文件中可选的参数含义，后续会专门出一片文章来介绍

{% asset_img 10.png %}

选择dpu_trd_system→dpu_trd_system_hw_link→dpu_trd_system_hw_link.prj，去除sfm_xrt_top，减少DPU数量为1(将DPU数量减小到1也是由于本文使用的FPGA资源受限)

{% asset_img 11.png %}

{% asset_img 12.png %}

在Assistant窗口双击dpu_trd_system，弹出窗口

{% asset_img 13.png %}

选择dpu_trd_system→dpu_trd_system_hw_link→Hardware→dpu

{% asset_img 14.png %}

点击V++configuration settings的省略号按钮，为其添加时钟配置

{% asset_img 15.png %}

添加如下代码

```makefile
[clock]
id=1:DPUCZDX8G_1.aclk
id=2:DPUCZDX8G_1.ap_clk_2

[connectivity]
sp=DPUCZDX8G_1.M_AXI_GP0:HPC0
sp=DPUCZDX8G_1.M_AXI_HP0:HP0
sp=DPUCZDX8G_1.M_AXI_HP2:HP1
```

这里的时钟接口和vivado中配置的时钟一一对应

{% asset_img 16.png %}

有些版本Vitis编译过程会出现找不到opencv库的error，点击Apply and Close，在dpu_trd_system→dpu_trd上右键选择C/C+= Build Settings

{% asset_img 17.png %}

在includes中添加

```makefile
${SYSROOT}/usr/include/opencv4
```

点击Apply

右键dpu_trd_system进行编译，该过程耗时较长，本机环境编译时长在20~30分钟。编译成功后，在vitis工程路径下的dpu_trd_system/Hardware/package路径下会生成sd_card.img

{% asset_img 18.png %}

该文件是合并了linux内核镜像、uboot、dpu,xclbin二进制文件以及设备树文件的SD卡镜像文件，SD卡分区已经做好的，直接使用balenaEtcher将该文件烧写到SD卡中即可
