---
title: VitisAI-04-PetaLinux Flow
date: 2022-06-30 16:56:43
tags: [Xilinx, DPU, Vitis AI, PetaLinux]
categories: 
- AI
- VitisAI
---

本文承接VitisAI-03-Vivado Flow，介绍使用Xilinx的PetaLinux工具将Vivado生成的design_1_wrapper.xsa文件创建PetaLinux并编译生成Linux镜像和rootfs的过程

# 创建PetaLinux工程

首先运行PetaLinux的环境变量脚本

```shell
source ~/opt/pkg/Xilinx/PetaLinux/2021.2/settings.sh
```

在dpu_vivado同级目录下，通过以下命令创建一个PetaLinux工程"dpu_plnx"

```shell
petalinux-create --type project --template zynqMP --name dpu_plnx
```

将vivado生成的.xsa文件拷贝到PetaLinux工程目录下并进行工程配置

```shell
cp dpu_vivado/dpu_hardware/design_1_wrapper.xsa dpu_plnx
cd dpu_plnx
petalinux-config --get-hw-description=.
```

运行配置命令后，会弹出类似配置内核时的menuconfig界面

{% asset_img 01.png %}

## 设置离线编译

由于PetaLinux的编译过程中需要从网络中下载很多包资源，并且很多包的源是外网，编译过程会很缓慢。Xilinx为PetaLinux的编译提供了离线下载方式，官网将PetaLinux编译依赖的包资源进行了打包处理，预先将依赖包下载到本地，再进行PetaLinux编译时，可以极大的加快编译速度，非常建议使用。离线编译包的下载在VitisAI-02-环境与资源文章中已经介绍，离线编译的缺点是需要占用100G+的磁盘容量

本文环境，已将downloads_2021.2.tar.gz和sstate_aarch64_2021.2.tar.gz两个文件下载并解压至~/opt/sstate-cache/downloads_2021.2和~/opt/sstate-cache/sstate_aarch64_2021.2路径下

1. 关闭Enable Network sstate feeds
   
   petalinux-config中，进入Yocto Settings，取消选择Yocto Settings→Enable Network sstate feeds

2. 开启Enable BB NO Network
   
   选中Yocto Settings→Enable Network sstate feeds

3. 设置local sstate feeds url
   
   Yocto Settings→local sstate feeds url设置为"/home/username/opt/sstate-cache/sstate_aarch64_2021.2/aarch64"

4. 设置Add pre-mirror url path
   
   Yocto Settings→Add pre-mirror url path设置为"file:///home/username/opt/sstate-cache/downloads_2021.2"

{% asset_img 02.png %}

## 修改文件系统类型为ext4

petalinux-config退回到根目录，选择Image Packaging Configuration->Root filesystem type为EXT4

{% asset_img 03.png %}

由于本文所使用FPGA开发板有EMMC和SD卡两个外部存储，根文件系统是放在SD卡中，SD卡为分区1，因此还需要修改SD卡的分区为/dev/mmcblk1p2

{% asset_img 04.png %}

至此，在petalinux-config需要修改的内容完毕

# 去掉dnndk

dnndk是vitisai老版本中使用的开发工具，新版本中已经废除了，需要在vitisai.bb中去掉dnndk

```shell
vim components/yocto/layers/meta-petalinux/recipes-core/packagegroups/packagegroup-petalinux-vitisai.bb
```

# 修改设备树文件

```shell
vim project-spec/meta-user/recipes-bsp/device-tree/files/system-user.dtsi
```

修改为

```json
/include/ "system-conf.dtsi"
/ {
        chosen {
                bootargs = "earlycon console=ttyPS0,115200 clk_ignore_unused root=/dev/mmcblk1p2 rw rootwait cma=512M";
        };
};

&sdhci1 {
        no-1-8-v;
        disable-wp;
};

/* USB */
&dwc3_0 {
    status="okay";
    dr_mode="host";
};
```

# 配置kernel

```shell
petalinux-config -c kernel
```

# 配置rootfs

## rootfs添加user package

vim打开user-rootfsconfig文件

```shell
vim project-spec/meta-user/conf/user-rootfsconfig
```

添加以下内容

```shell
CONFIG_xrt
CONFIG_dnf
CONFIG_e2fsprogs-resize2fs
CONFIG_parted
CONFIG_resize-part
CONFIG_packagegroup-petalinux-vitisai
CONFIG_packagegroup-petalinux-self-hosted
CONFIG_cmake
CONFIG_packagegroup-petalinux-vitisai-dev
CONFIG_xrt-dev
CONFIG_opencl-clhpp-dev
CONFIG_opencl-headers-dev
CONFIG_packagegroup-petalinux-opencv
CONFIG_packagegroup-petalinux-opencv-dev
CONFIG_mesa-megadriver
CONFIG_packagegroup-petalinux-x11
CONFIG_packagegroup-petalinux-v4lutils
CONFIG_packagegroup-petalinux-matchbox
```

配置rootfs

```shell
petalinux-config -c rootfs
```

选择user packages，选中所有内容

{% asset_img 05.png %}

# 编译

```shell
petalinux-build
```

编译后在工程目录下的images/linux路径下会生成u-boot、rootfs、linux image等文件
