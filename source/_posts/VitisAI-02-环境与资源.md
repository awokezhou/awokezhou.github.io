---
title: VitisAI-02-环境与资源
date: 2022-05-04 21:07:31
tags: [Xilinx, Vitis AI, AI加速]
categories:
- AI
- VitisAI
---

在第一篇文章"VitisAI-01-Overview"中，我简要介绍了什么是VitisAI、VitisAI相关的的技术栈、我的开发环境是什么样的以及需要安装下载哪些资源。本文对我的开发环境以及所需资源的下载安装进行一个更为详细的说明

# 环境介绍

我使用的台式机操作系统是Windows11，我采用的是在虚拟机中搭建VitisAI的方式来进行开发和研究。由于VitisAI开发需要PetaLinux编译，而目前PetaLinux仅支持Linux操作系统，而我并不使用Linux作为我的主操作系统，因此选择在虚拟机中搭建Linux系统的方式。我的系统配置详细信息如下

* 主操作系统：Windows 11 专业版(22000.613)

* 处理器：12th Gen Intel(R) Core(TM) i7-12700K   3.61 GHz

* 虚拟机：VMware Workstation 16.2.2
  
  * 操作系统：Ubuntu20.0.4
  
  * 处理器：内核数8
  
  * 内存：12GB
  
  * 磁盘：400GB

这里需要注意的是虚拟机中处理器相关的设置，截图如下

{% asset_img 01.png %}

处理器数量选择1，每个处理器的内核数量选择8，因此总内核数为8。虚拟化引擎选中了"虚拟化 Inter VT-x/EPT 或 AMD-V/RVI(V)"。按照我的处理器设置整个VitisAI的开发过程实测是没有问题的。主要容易出问题的地方是PetaLinux编译，因为PetaLinux的多线程编译与内核数量相关，内核数量越多，可同时执行的编译线程越多，8个内核同时只能进行8个编译线程。曾经尝试过将处理器数量调整到12或16以加快PetaLinux的编译，但是编译到一定进度后总会发生停住不动的问题(查看CPU占用率基本为0，表明CPU未分给编译线程资源)，这里并没有搞清楚到底是什么原因，可能是12代CPU使用了Alder Lake的大小核架构与VMware Workstation不兼容问题

# 资源下载及安装

虚拟机软件VMware Workstation与Ubuntu20.0.4虚拟机的下载安装并不包含在内。这里主要介绍与VitisAI相关的各个开发工具的下载与安装。其中Vivado和Vitis 2021.1和2021.2都可以，PetaLinux建议使用2021.2

## Xilinx Unified Installer

子2019年之后，Xilinx将软件开发平台进行了统一，其后的软件开发平台称作Vitis统一开发平台，Xilinx Unified Installer就是统一开发平台的安装包，这里推荐下载2021.2版本，打开地址[Vitis下载](https://china.xilinx.com/support/download/index.html/content/xilinx/zh/downloadNav/vitis/2021-2.html)，可以看到如下内容

{% asset_img 02.png %}

点击2021.2，下翻页面到如下位置，点击"[赛灵思统一安装程序 (Xilinx Unified Installer 2021.2) SFD](https://china.xilinx.com/member/forms/download/xef.html?filename=Xilinx_Unified_2021.2_1021_0703.tar.gz) (TAR/GZIP - 71.9 GB)"进行下载

{% asset_img 03.png %}

下载下来是一个.tar.gz的压缩包，将其移动到虚拟机中，解压后运行其中的xsetup即可开始安装，安装过程是UI界面的，选择Vitis，会安装Vivado和Vitis以及一系列相关依赖库，然后选择安装路径即可自动安装

我的安装路径为"/home/opt/pkg/Xilinx"，安装完成后，可通过如下命令打开Vivado来检查是否安装成功

```shell
source /home/opt/pkg/Xilinx/Vivado/2021.2/setting64.sh
vivado
```

## PetaLinux

PetaLinux的安装文件可单独下载，也可以使用Xilinx Unified Installer进行安装。使用Xilinx Unified Installer安装时仅需要在安装步骤的第一步选择PetaLinux即可。推荐使用Xilinx Unified Installer进行安装

我的PetaLinux安装路径为"/home/opt/pkg/Xilinx/PetaLinux"，PetaLinux安装完成后，可通过如下命令来查看PetaLinux是否安装成功

```shell
source /home/opt/pkg/Xilinx/PetaLinux/2021.2/setting64.sh
petalinux -h
```

### PetaLinux依赖库

由于PetaLinux编译需要很多额外的依赖库，Xilinx为PetaLinux提供了Release Note来说明不同系统中需要依赖哪些库，打开[PetaLinux下载地址](https://china.xilinx.com/support/download/index.html/content/xilinx/zh/downloadNav/embedded-design-tools/2021-2.html)

{% asset_img 04.png %}

在右侧旁边有对应版本的发布说明，点击它，并在打开的页面中下拉到最下方，下载"2021.2_PetaLinux_Package_List.xlsx"

{% asset_img 05.png %}

用Excel打开该文件，内容如下

{% asset_img 06.png %}

其中有Ubuntu系统下的依赖库安装命令

```shell
sudo apt-get install iproute2 gawk python3 python build-essential gcc git make net-tools libncurses5-dev tftpd zlib1g-dev libssl-dev flex bison libselinux1 gnupg wget git-core diffstat chrpath socat xterm autoconf libtool tar unzip texinfo zlib1g-dev gcc-multilib automake zlib1g:i386 screen pax gzip cpio python3-pip python3-pexpect xz-utils debianutils iputils-ping python3-git python3-jinja2 libegl1-mesa libsdl1.2-dev pylint3
```

### PetaLinux离线编译

由于PetaLinux在编译过程中需要从网络上下载很多第三方包，而很多第三方包的资源在外网中，编译速度受限于网络下载速度。Xilinx提供了离线的第三方包缓存文件，将其下载到本地后，在配置PetaLinux时选择编译方式为离线编译，即可不依赖网络进行离线编译，加快编译速度。离线缓存文件下载位置仍然在[PetaLinux下载地址](https://china.xilinx.com/support/download/index.html/content/xilinx/zh/downloadNav/embedded-design-tools/2021-2.html)页面，下拉到最下方，选择"[aarch64 sstate-cache](https://china.xilinx.com/member/forms/download/xef.html?filename=sstate_aarch64_2021.2.tar.gz) (TAR/GZIP - 20.27 GB)"和"[下载](https://china.xilinx.com/member/forms/download/xef.html?filename=downloads_2021.2.tar.gz)下载 (TAR/GZIP - 57.4 GB)"

{% asset_img 07.png %}

## Vitis AI仓库

Vitis AI相关库和工具在Xilinx官方提供的github仓库中，需要将其克隆到本地，该仓库较大，网络不佳克隆可能需要较长时间，命令为

```shell
git clone https://github.com/Xilinx/Vitis-AI/tree/master
```

## Vitis AI docker

docker添加到用户组

```shell
sudo usermod -aG docker <username>
sudo systemctl restart docker
sudo chmod 666 /var/run/docker.sock
```

VitisAI docker镜像下载

```shell
cd Vitis-AI
docker pull xilinx/vitis-ai:latest
```

运行docker镜像

```shell
./docker_run.sh xilinx/vitis-ai
```

显示如下界面表示运行成功

{% asset_img 08.png %}

## 安装主机交叉编译

```shell
cd Vitis-AI/setup/mpsoc/VART
./host_cross_compiler_setup.sh
```

安装后运行以下命令测试是否安装正确

```shell
source ~/petalinux_sdk_2021.2/environment-setup-cortexa72-cortexa53-xilinx-linux
cd Vitis-AI/demo/VART/resnet50
bash –x build.sh
```
