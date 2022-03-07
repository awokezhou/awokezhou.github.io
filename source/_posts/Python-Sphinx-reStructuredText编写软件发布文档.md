---
title: Python+Sphinx+reStructuredText编写软件发布文档
date: 2022-01-22 14:12:07
tags: [Python, Sphinx, reStructuredText]
categories:
- 文档方案
comments: true
---

很多开源项目或者软件都会提供一个类似如下形式的网页文档，页面看上去很清晰整洁，左侧可以方便的查看整个文档的层次结构，文档中支持嵌入高亮代码块

本文介绍在windows环境下通过Python安装Sphinx，使用reStructuredText语法编写软件文档并生成网页文档的方法。本文使用的环境和版本

* Windows10

* Python3.7.0

* sphinx4.4.0

# sphinx安装与工程创建

## sphinx安装

cmd或WindowsPowershell运行以下命令安装sphinx

```powershell
pip install sphinx
```

安装完成后，命令行中可以使用以下4个命令工具

* sphinx-apidoc：api文档

* sphinx-autogen：可以根据文档结构描述自动生成文档

* sphinx-build：编译原文档

* sphinx-quickstart：自动创建一个新的sphinx工程

以上命令实际使用中主要会用到sphinx-build和sphinx-quickstart

## 创建sphinx工程

首先创建一个文件夹，进入文件夹后输入sphinx-quickstart，即可在该文件夹下创建一个sphinx工程

```power
mkdir helloworld
cd helloworld
sphinx-quickstart
```

sphinx-quickstart会在命令行中提示一些创建工程的选项，以下选项可做参考

![](Python-Sphinx-reStructuredText编写软件发布文档\images\01.PNG)

主要变动的地方

* 独立的源文件和构建目录(y/n)，选y

* 项目名称，按需要输入

* 作者名称，按需要输入

* 项目发行版本，按需要输入

其余选项可以直接回车按默认值设置，完成后在helloworld路径下会自动生成build和source两个文件夹

![](Python-Sphinx-reStructuredText编写软件发布文档\images\02.PNG)

source文件夹下是项目源文件所在，也就是文档文件存放位置，build文件夹下是通过source生成的输出文件。sphinx支持多种输出方式，默认为html输出，也可以选择latex、pdf等其他方式输出，但是需要做额外配置，本文只介绍默认html输出方式。另外sphinx默认支持的源文件格式为.rst，即reStructuredText语法，如果要支持其他语法例如Markdown也需要做额外配置

## 编译

在项目根目录下运行以下命令，即可编译源文件生成html

```powershell
sphinx-build.exe source build
```

![](Python-Sphinx-reStructuredText编写软件发布文档\images\03.PNG)

进入build目录下可以看到生成了html文件，index.html是入口文件

![](Python-Sphinx-reStructuredText编写软件发布文档\images\04.PNG)

双击index.html文件，浏览器中显示如下页面

![](Python-Sphinx-reStructuredText编写软件发布文档\images\05.PNG)

## 修改主题

默认的html样式比较难看，可以通过安装修改主题使得页面更美观，在[Sphinx Themes Gallery](https://sphinx-themes.org/)网站可以查看sphinx支持的各种主题

![](Python-Sphinx-reStructuredText编写软件发布文档\images\06.PNG)

比较典型的是使用Read the Docs主题，在命令行输入以下命令安装该主题

```powershell
pip install sphinx-rtd-theme
```

安装完成后在sphinx工程helloworld的source/conf.py中修改以下两处

```python
extensions = [
    'sphinx_rtd_theme',
]

html_theme = 'sphinx_rtd_theme'
```

再次编译工程

```powershell
sphinx-build.exe source build
```

打开build/index.html，可以看到页面样式已经变为Read the Docs样式

![](Python-Sphinx-reStructuredText编写软件发布文档\images\07.PNG)

# reStructuredText语法简介

sphinx工程默认是以reStructuredText来编写文档，文档的入口在source/index.rst文件

## 文档组织结构

用sphinx编写文档一般都是写类似书籍形式的多部分多章节结构，所有文档内容并不是在一个源文件中编写，而是在多个rst文档中编写，并通过reStructuredText的语法来建立文档的层次结构。reStructuredText中通过`toctree`语法来建立文档间的层次结构，在source/index.rst文档中有如下代码

```rest
.. toctree::
   :maxdepth: 2
   :caption: Contents:
```

以下举例说明`toctree`语法如何使用。假设要建立一个两级的文档层次，最高层级是一个简介，第二级是两个章节：第一章和第二章，章节分别在一个单独页面显示

![](Python-Sphinx-reStructuredText编写软件发布文档\images\08.PNG)

![](Python-Sphinx-reStructuredText编写软件发布文档\images\09.PNG)

![](Python-Sphinx-reStructuredText编写软件发布文档\images\10.PNG)

源文件的结构如下

|-- source

    |-- index.rst    # 总入口

    |-- chapter1    # 第一章文件夹 

        |-- index.rst # 第一章入口

    |-- chapter2     # 第二章文件夹

        |-- index.rst  # 第二章入口

在source目录下需创建两个文件夹chapter1和chapter2，并在章节文件夹下分别创建两个index.rst。source/index.rst文件内容如下

```rest
简介
======================================

Hello，这里是Sphinx测试工程

.. toctree::
   :maxdepth: 2

   第一章 <chapter1/index.rst>
   第二章 <chapter2/index.rst>
```

总入口的index.rst中，通过`toctree`语法声明了两个子部分，注意声明的部分需要与maxdepth隔一行，maxdepth表示在html页面中左侧展示的接口最多展示多少层。

## 标题

reStructuredText中使用的标题与Markdown不同

* #及以上表示部分

* *及以上表示章节

* =表示小章节

* -表示子章节

* ^表示子章节的子章节

```rest
############
部分
############

************
章节
************

小章节
============

子章节
------------

子章节的子章节
^^^^^^^^^^^^
```

注意部分和章节是上下都有标识符

## 代码嵌入

在reStructuredText中嵌入代码有两种方式，一种是行内嵌入，一种是嵌入代码块。行内嵌入使用以下格式

```makefile
空格+``+代码+``+空格
```

嵌入代码块格式如下

```rest
空行
.. code-block:: c

    #include "<stdio.h>"

    int main(int argc, char *argv[])
    {
        printf("hello\r\n");
        return 0;
    }
空行
```

在html页面显示效果如下

![](Python-Sphinx-reStructuredText编写软件发布文档\images\11.PNG)

## 嵌入图片

在reStructuredText中嵌入图片，使用`image`语法，格式如下

```rest
.. image:: /images/hello.png
```

后面跟的/images/hello.png是指示文件路径，注意这里是相对路径，在source目录下创建images文件夹，将hello.png放在该路径下，效果如图

![](Python-Sphinx-reStructuredText编写软件发布文档\images\12.PNG)

## 嵌入注意事项

按如下语法可以在页面中嵌入Note注意事项

```rest
.. note:: 

    请注意，这里是注意事项
```

效果如下

![](Python-Sphinx-reStructuredText编写软件发布文档\images\13.PNG)

## 嵌入表格

在reStructuredText中使用表格比较复杂，需要格式对齐的非常准确才可以，不想Markdown中那么方便，简单的格式如下

```rest
+-------+-------+
| head1 | head2 |    /* 表头定义 */
+=======+=======+
| cont1 | cont2 |    /* 内容定义 */
+-------+-------+
```

以上代码效果如下

![](Python-Sphinx-reStructuredText编写软件发布文档\images\14.PNG)

### vscode table formatter插件

建议在vscode中编写reStructuredText，安装table formatter插件，能够辅助对齐表格

![](Python-Sphinx-reStructuredText编写软件发布文档\images\15.PNG)

在使用该插件编写表格时，先不需要考虑表格的对齐问题，只需按照表格格式编写好大体内容

```rest
+-+-+
|head1|head2|
+=+=+
|content1|content2|
+-+-+
```

鼠标全选表格内容，并按ctrl+shift+p选择table current

![](Python-Sphinx-reStructuredText编写软件发布文档\images\16.PNG)

插件将会自动对齐表格

![](Python-Sphinx-reStructuredText编写软件发布文档\images\17.PNG)
