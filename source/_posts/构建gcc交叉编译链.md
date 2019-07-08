---
title: 构建gcc交叉编译链
date: 2019-07-05 17:12:36
tags: [译]
categories:
- 嵌入式
- 编译
comments: true
---

# 构建gcc交叉编译链

本文翻译“Cross-compilation Tool Making”

## download
* Binutils: [Index of /gnu/binutils](http://ftp.gnu.org/gnu/binutils/)
* gcc: [Index of /gnu/gcc/](http://mirrors.kernel.org/gnu/gcc)
* Glibc & Glibc-ports: [Index of /gnu/glibc](http://ftp.gnu.org/gnu/glibc/)
* mpfr-2.4.2: wget ftp://gcc.gnu.org/pub/gcc/infrastructure/mpfr-2.4.2.tar.bz2
* gmp-4.3.2: wget ftp://gcc.gnu.org/pub/gcc/infrastructure/gmp-4.3.2.tar.bz2
* mpc-0.8.1: wget ftp://gcc.gnu.org/pub/gcc/infrastructure/mpc-0.8.1.tar.gz

## dir 
* setup-dir: download files
* src-dir: unzip files

## Binutils install
```shell
./configure --target=$TARGET --prefix=$PREFIX
```


## gcc

[Linux ARM交叉编译工具链制作过程 - w36680130的博客 - CSDN博客](https://blog.csdn.net/w36680130/article/details/81740046)

# Build a GCC-based cross compiler for Linux

## Section 1. 开始之前
### 关于这篇教程
有时候，你正在开发的平台和使用的计算机不匹配。例如，你可能想要在你的x86/Linux笔记本电脑上构建一个PowerPC/Linux应用程序。使用GNU工具包中的gcc、gas和lds工具，您可以指定并构建一个交叉编译器，它将使您能够在您的机器上为其他目标构建程序。只需多做一点工作，您甚至可以设置一个能为各种不同的目标构建应用程序的环境。在本教程中，我将介绍在你的系统上构建交叉编译器所需的整个过程。还将讨论为一系列目标构建一个完整的环境，向您展示如何与distcc和ccache工具集成，并介绍更新你的新开发平台以及更新最新版本的方法。

要构建一个交叉编译器，您需要了解典型UNIX开源项目的构建过程的基本知识、一些基本的shell技能和足够的耐心。

### 预备知识
要构建一个交叉编译器，您需要一个可工作的C编译器(gcc通常是一个好主意)。大多数基于Linux/unix的操作系统都提供了C编译器。您还需要用于构建交叉编译器的各种工具的源代码。您可以从GNU(http://www.gnu.org)下载GNU工具。

除了这些工具之外，还需要目标平台的头文件副本。对于Linux目标，使用Kernel.org(http://www.kernel.org)提供的通用Linux内核头文件。

## Section 2. 交叉编译
### 为什么需要交叉编译
并不总是在同一个平台上编写和构建应用程序。例如，对于许多嵌入式环境，用于RAM和存储的空间较小，通常小于256MB，甚至可能小于64MB。一个合理的C编译器、相关的工具和所需的C库都不适合这么小的空间，更不用说运行了。

实际上，在这样的环境下开发显然更加困难。如果您使用键盘显示器访问和使用系统，使用成熟的编辑器(例如emacs)或成熟的开发环境(IDEA)是不可能的。许多嵌入式解决方案甚至没有网络访问的能力。

跨编译器使您能够在一个具有开发能力的平台(主机)上进行开发，而实际构建另一个目标系统的程序。目标计算机并不需要可用，您所需要的只是一个编译器，它知道如何为您的目标平台编写机器码。交叉编译器在其他情况下也很有用。我曾经不得不在一台没有安装C编译器的计算机上工作，而且我没有获得预编译二进制文件的简单方法。但是，我确实拥有GNU编译器集合(GCC)、C库(newlib)和计算机上的二进制实用程序的必要源代码，而计算机上确实有C编译器。使用这些工具，我可以先为目标计算机构建一个交叉编译器，然后再为目标计算机构建一个本机编译器，我可以跨目标计算机复制并直接使用它。

当您的计算机速度较慢且速度快得多，并且希望在几分钟内而不是几小时或几天内构建时，交叉编译器也很方便。我曾使用这种方法在一台计算机上执行升级到新软件版本的任务，在此过程中，需要花费2-3天的时间来重建所有组件——而此时计算机正在执行其已经非常重要的服务器任务。

在我向您展示构建交叉编译器的细节之前，让我们仔细看看编译是如何工作的，以便您更好地理解为什么——更重要的是——交叉编译是如何工作的。

### 交叉编译如何工作 
编译器的工作方式很简单。几个不同的组件一起工作， 最终目标是生成目标CPU使用的字节码。当您可以生成组装好的字节码时，您已经成功地交叉编译了。

任何编译器的主要组件包括:
* **解析器**：解析器将原始语言源代码转换为汇编语言，因为要从一种格式转换为另一种格式(C语言转换为汇编)，所以解析器需要知道目标汇编语言
* **汇编器**：汇编程序将汇编语言代码转换成CPU执行的字节码
* **链接器**：链接器将汇编程序生成的单个目标文件组合到可执行应用程序中。不同的操作系统和CPU组合通常使用不同的封装机制和标准。链接器需要知道该目标格式才能工作
* **标准C库**：核心C函数(例如printf)在C库中提供。假设应用程序中使用了来自C库的函数，那么这个库将与链接器和源代码结合使用，生成最终的可执行文件

在标准的、基于主机的C编译器中，每个组件都被设计成生成相应的汇编代码、字节码和主机本身的目标执行格式。在交叉编译器中，虽然构建应用程序是为了在主机上执行，但汇编语言、链接器和C库都是为目标平台和处理器设计的。例如，基于intel的
在Linux机器上，您可以交叉编译一个应用程序，以便汇编语言和最终的应用程序都适用于基于solarisis的SPARC主机。

因此，构建交叉编译器依赖于构建C编译器套件的替代版本，该版本为目标主机生成并链接应用程序。幸运的是，因为可以编译GCC和相关的工具，所以可以构建自己的交叉编译器。

### 交叉编译器构建过程
GNU工具集(即GCC)，包括C编译器、二进制实用程序和C库，都有一些好处，其中最重要的是它们是免费的、开放源码的，并且易于编译。从跨编译器的角度来看，一个更大的好处是，由于GCC已经移植到许多平台上，代码支持几种不同的CPU和平台类型。然而，也有一些限制。GCC并不支持所有处理器类型(尽管它生成的处理器类型最多)，也并不支持所有平台。配置工具会在执行时警告您这一点。

要构建一个交叉编译器，您需要从GNU套件构建三个组件：
* **binutils**：binutils包包括基本的二进制实用程序，如汇编程序、链接器和相关工具，如Size和Strip。二进制实用程序包含用于构建应用程序的核心组件和用于构建和操作目标执行格式的工具。例如，Strip程序从目标文件或应用程序中删除符号表、调试和其他“无用”信息，但要做到这一点，实用程序需要知道目标格式，以便不删除错误的信息
* **gcc**：gcc是编译过程的主要组件。Gcc包含C预处理器(cpp)和转换器，后者将C代码转换为目标CPU汇编语言。Gcc还充当整个流程的接口，相应地调用cpp、翻译程序、汇编程序和链接器
* **newlib/glibc**：这个库是标准的C库。Newlib是通过Redhat开发的，在为嵌入式目标设计的交叉编译器中，它对用户稍微友好一些。您也可以使用GNU库(glibc)，但我在本教程中主要使用newlib

您还需要目标操作系统的头文件，这是必需的，以便您能够访问构建应用程序所需的所有操作系统级函数和系统调用。使用Linux，您可以相当容易地获得头文件。对于其他操作系统，您可以复制一组现有的头文件。稍后我将更详细地说明头文件。

您可以选择为目标主机构建GNU调试器(gdb)。您不能构建一个调试器来在主机上运行时为目标执行代码，因为这样做需要仿真。不过，您可以为目标主机构建一个gdb可执行文件。

## Section 3. 准备
### 目的地和共存
在开始配置过程之前，需要确定将在何处安装编译器和相关工具。您有两个选择:要么将它们安装到一个完全独立的目录中，要么将它们作为现有安装的一部分安装。

GNU工具集的许多好处之一是内置在安装结构中的设计，它使不同目标平台的工具和组件能够共存。安装软件后，您提供的安装前缀将根据正常布局进行组织，并为特定于目标的工具添加一个目标目录。例如，下面的结构取自我为泛型安装了交叉编译器的系统
PowerPC / Linux平台:

drwxrwxrwx 2 root root 4096 Nov 16 16:48 bin/
drwxrwxrwx 2 root root 4096 Nov 17 12:53 info/
drwxrwxrwx 2 root root 4096 Nov 17 12:53 lib/
drwxrwxrwx 3 root root 4096 Nov 16 16:44 man/
drwxrwxrwx 4 root root 4096 Nov 16 16:48 ppc-linux/
drwxrwxrwx 3 root root 4096 Nov 16 16:43 share/

如果你查看bin目录，你会看到每个主要的二进制实用程序都有你的build-target前缀:

-rwxr-xr-x 1 root root 2108536 Nov 16 16:46 ppc-linux-addr2line*
-rwxr-xr-x 2 root root 2157815 Nov 16 16:45 ppc-linux-ar*
-rwxr-xr-x 2 root root 3398961 Nov 16 16:48 ppc-linux-as*
-rwxr-xr-x 1 root root 2062804 Nov 16 16:47 ppc-linux-c++filt*
-rwxr-xr-x 2 root root 2907348 Nov 16 16:48 ppc-linux-ld*
-rwxr-xr-x 2 root root 2140893 Nov 16 16:46 ppc-linux-nm*
-rwxr-xr-x 1 root root 2552661 Nov 16 16:46 ppc-linux-objcopy*
-rwxr-xr-x 1 root root 2708801 Nov 16 16:45 ppc-linux-objdump*
-rwxr-xr-x 2 root root 2157810 Nov 16 16:46 ppc-linux-ranlib*
-rwxr-xr-x 1 root root 371010 Nov 16 16:46 ppc-linux-readelf*
-rwxr-xr-x 1 root root 2008330 Nov 16 16:45 ppc-linux-size*
-rwxr-xr-x 1 root root 1982880 Nov 16 16:46 ppc-linux-strings*
-rwxr-xr-x 2 root root 2552660 Nov 16 16:46 ppc-linux-strip*

主要工具(如gcc)只是执行编译的后台工具的包装器，因此gcc可以在为不同平台构建时确定使用哪个工具。只要您继续将gcc用于您的构建需求，以及您构建的其他库和组件使用GNU configure结构，您应该能够在标准工具集旁边安装交叉编译工具。在大多数Linux平台上，这个位置都是/usr/local。我喜欢把我的交叉编译器和宿主编译器分开，这样我就可以保持不同版本的宿主和跨编译器工具集。

### 确定您的目标平台
准备交叉编译的下一步是确定目标平台。GNU系统中的目标有一个特定的格式，这个信息在整个构建过程中被用来识别各种工具的正确版本。因此，当您使用特定的目标运行GCC时，GCC会在目录路径中查找包含该目标规范的应用程序路径。

GNU目标规范的格式是CPU-PLATFORM-OS。x86的Solaris 8是i386-pc-solaris2.8，而Macintosh OS X是powerpc-apple-darwin7.6.0。PC上的Linux具有目标i686-pc-linux-gnu。本例中的-gnu标记表示Linux操作系统使用的是gnu风格的环境。

有许多方法可以识别目标，包括简单地了解或猜测目标规范。例如，大多数Linux目标可以根据它们的CPU来指定；因此，PowerPC/Linux是ppc-linux。然而，最好的方法是使用config.guess脚本，它随任何脚本一起提供GNU套件，包括我在本教程中使用的那些。要使用这个脚本，只需从一个shell运行它:

```shell 
$ config.guess
i686-pc-linux-gnu
```
对于那些无法运行此脚本的系统，有必要检查该文件以确定一些可能的目标。只要在目标CPU上使用grep，就可以了解所需的目标规范。出于本教程的目的，我将为i386-linux平台创建一个交叉编译器，这是一个公共目标，也是受支持程度最高的平台之一。

### 设置构建环境
开始构建之前的最后一个阶段是创建一个合适的环境。您只需要创建一个简单的目录集，您可以使用它来构建不同的组件。

首先，创建一个构建目录:
```shell
$ mkdir crossbuild
```
接下来，获取gcc、binutils、gdb和newlib的最新版本，并将它们提取到目录中:
```bash
$ bunzip2 -c gcc-3.3.2.tar.bz2|tar xf -
$ bunzip2 -c binutils-2.14.tar.bz2 |tar xf -
$ bunzip2 -c linux-2.6.9.tar.bz2 |tar xf -
$ bunzip2 -c gdb-6.3.tar.bz2|tar xf -
$ bunzip2 -c glibc-2.3.tar.bz2|tar xf -
```

如果正在为Linux目标构建，还需要解包Linux -threads包(如果目标平台支持它)。

现在，创建实际构建软件的目录。基本布局是为每个组件创建一个目录:
```shell
$ mkdir build-binutils build-gcc build-glibc build-gdb
```
如果要创建几个不同的交叉编译器，可以考虑为每个目标创建单独的目录，然后在每个目标目录中创建上面的目录。

准备工作完成后，就可以配置和构建每个工具了。

## Section 4. 配置和构建
### 设置环境变量
重新输入所有东西是令人沮丧的，你可能会输入错误的东西。为了简化工作，我创建了几个环境变量来节省输入。我假设下面有一个类似伯恩的壳；如果使用csh、tcsh或类似的shell，可能需要使用特定于shell的技术。

```shell
export TARGET=i386pc
export PREFIX=/usr/local/crossgcc
export TARGET_PREFIX=$PREFIX/$TARGET
export PATH=$PATH:$PREFIX/bin
```

### 获取操作系统头文件
操作系统头对于获取编译器需要的信息是必要的，这些信息用于目标平台支持的系统函数调用。对于Linux目标，获得头文件的最佳方法是下载适当内核的最新副本。您需要对内核进行基本配置，以便生成正确的头文件供您复制，但是，您不需要构建或编译内核。对于我的示例目标i386-linux，必要的步骤如下：
```shell
$ cd linux-2.6.9
$ make ARCH=i386 CROSS_COMPILE=i386-linux- menuconfig
```
注意，上面后面的连字符不是拼写错误。通过运行上面的命令，将提示您为内核配置组件。因为您不需要过多地担心内核本身的内容，所以您只需要使用默认选项，保存配置，然后退出工具。

现在，您需要将标题复制到目标目录:
```shell
$ mkdir -p $TARGET_PREFIX/include
$ cp -r include/linux $TARGET_PREFIX/include
$ cp -r include/asm-i386 $TARGET_PREFIX/include/asm
$ cp -r include/asm-generic $TARGET_PREFIX/include/
```

显然，我在上面的代码中基于示例目标做了一些假设。如果您正在为PowerPC目标构建，您应该已经从asm-ppc复制了文件。

现在可以开始构建实用程序了。

### 构建binutils
binutils实用程序是整个系统的核心。它提供了系统其余部分所需的基本汇编器和链接器功能。每种情况的第一个阶段都是为另一种目标平台配置每个包。构建的第一步：
```shell
$ cd build-binutils
$ ../binutils-2.14/configure --target=$TARGET --prefix=$PREFIX --disable-nls -v
$ make all
```
目标和前缀应该是显而易见的。命令禁用国家语言支持(NLS)。无论如何，许多嵌入式平台都不能支持必要的表。对于大多数交叉编译器，NLS的有用性是有争议的，因为目标(通常是嵌入式设备)不能在内存中保存必要的NLS表。

构建的这个阶段将需要一些时间。因为您仍然在使用主机编译器和工具进行构建，所以可以使用ccache和distcc工具来帮助加速这个过程。有关这些工具的更多信息，请参见参考资料(# Resources)部分。

现在，您已经准备好构建GCC，它稍微复杂一些。

### 构建第一阶段GCC
GCC比binutils更复杂，这仅仅是因为构建GCC的标准方法构建两个编译器。GCC使用GNU工具来构建一个主程序(即，第一阶段，或引导)编译器，可以构建和解析基本代码。然后GCC使用目标的可用库和头文件来构建完整的编译器。构建GCC第一阶段编译器需要对配置脚本的选项进行一些较小的更改，这样您就可以在没有适当头文件的情况下构建第一阶段编译器。严格地说，只有在构建了库之后才会有头文件。`with-newlib`命令并不一定意味着您正在使用newlib:它只是告诉配置脚本不要担心头文件。

```shell
$ cd build-gcc
$ ../gcc-3.3.2/configure --target=$TARGET --prefix=$PREFIX \
--without-headers --with-newlib -v
$ make all-gcc
$ make install-gcc
```

与构建binutils一样，这个阶段需要一段时间才能完成。时间长短取决于您的主机，但是可以期望从一个小时(即使是在高速机器上)到慢速或繁忙的主机上最多5或6个小时。

### 构建newlib
您可以使用glibc的newlib。c.总的来说，newlib在嵌入式平台上表现得更好，因为newlib的设计初衷就是支持嵌入式平台。Glibc更适合linux风格的主机。

在构建newlib时，需要使用目标编译器和工具构建库。当然，该库应采用目标CPU和平台的格式和语言，以便用于构建依赖于库组件的应用程序:
```shell
$ cd build-newlib
$ CC=${TARGET}-gcc ../newlib-1.12.0/configure --host=$TARGET --prefix=$PREFIX
$ make all
$ make install
```

构建了newlib之后，您可以基于此代码创建最终的GCC来创建最终的编译器。或者，您可以使用glibc，我将在下一个面板中介绍它。

### 构建glibc
glibc包构建起来很简单；与以前版本的主要区别在于，与newlib一样，现在开始使用刚才构建的引导跨编译器。您还需要告诉configure脚本操作系统的头文件保存在哪里。最后——这是一个很大的区别——定义构建的主机而不是目标。这是因为您已经构建的GCC和二进制实用程序意味着这台机器是您的开发主机；您指定的GCC将为您生成必要的目标代码。
```shell
$ CC=${TARGET}-gcc ../glibc-2.3/configure --target=$TARGET \
--prefix=$PREFIX --with-headers=${TARGET_PREFIX}/include
$ make all
```

如果您想包含Linux threads选项，需要在configure脚本中添加`--enable-add-ons`选项。同样，这个过程需要一些时间来完成。从来没有人说构建交叉编译器是快速的。

要安装glibc，您仍然使用make，但是您显式地设置了安装根并清空了前缀(否则，这两者将被连接起来，这不是您想要的)：
```shell
$ make install_root=${TARGET_PREFIX} prefix="" install
```

最后，您可以构建gcc的最终版本，它现在使用上面的库和头信息。

### 构建最终的GCC
最后的GCC使用您刚刚编译的头文件和库(使用您选择的目标)构建完整的gcc系统。不幸的是，这意味着在构建完整的gcc版本时，等待的时间更长。我只使用gcc构建了一个C编译器，而不是构建完整的套件。

您不需要担心旧的gcc构建，因此您可以删除该内容，然后在build-gcc目录中重新启动。配置与前面的示例一样。注意，因为您正在构建一个将在这个平台上执行的工具，所以您将回到使用宿主GCC编译器，而不是之前构建的引导GCC编译器:
```shell
$ cd build-gcc
$ rm -rf *
$ ../gcc-3.3.2/configure --enable-languages=c --target=$TARGET --prefix=$PREFIX
$ make all
$ make install
```

因为您正在为目标平台构建完整的newlib和gcc组件，如果不在distcc主机列表中的其他机器上安装这些组件，就不可能使用distcc。即使使用distcc，也需要一些时间，即使是在高速机器上。如果可能的话，我倾向于让这些构建在夜间运行——部分原因是时间，但部分原因是它增加了构建机器(或多个机器)的负载，如果您的机器同时用于其他目的，这可能会很烦人。

不过，当构建完成后，就可以开始使用交叉编译器来构建目标平台所需的其他应用程序和库了。

## Section 5. 安装并使用交叉编译器
### 使用你的交叉编译器
实际上使用交叉编译器很容易。要编译一个简单的C文件，只需直接调用交叉编译器：
```shell
$ i386-linux-gcc myapp.c -o myapp
```

GCC交叉编译器的工作原理与本地版本一样:它只是为另一个平台创建了不同类型的可执行文件。这意味着您可以使用相同的命令行选项,如头和库位置、优化和调试。(请记住,您不能将不同目标平台的库连接起来,并期望它们能够工作。)在下一个面板中,我将向您展示如何使用新的跨编译器构建库和扩展。

对于具有Makefile的应用程序和项目，请在要生成的命令行上指定交叉编译器。例如：
```shell
$ make CC=i386-linux-gcc
```

或者，更改Makefile中的CC定义。

### 编译其他工具和库
您可能希望在目标平台上使用的任何库都需要使用交叉编译器进行编译，以便它们能够工作。对于大多数库，应用与构建自己的项目相同的基本规则。如果一个系统使用一个简单的Makefile,使用:

```shell
$ make CC=i386-linux-gcc
```

或者，如果使用configure脚本，在configure命令前面加上重新定义CC环境变量的前缀：
```shell
CC=i386-linux-gcc ./configure
```

使用gnu风格的配置脚本，您可能还需要指定主机：
```shell
$ ./configure --host=i386-linux
```

记住，对于库，您需要指定目标前缀，就像您在构建glibc或newlib时所做的那样。