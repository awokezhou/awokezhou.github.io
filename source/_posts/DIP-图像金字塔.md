---
title: DIP-图像金字塔
date: 2022-05-18 09:40:22
tags: [DIP, 图像融合]
categories:
- CV
- DIP
---

图像金字塔对于执行多尺度的编辑操作非常有效，能够在保持图像细节的同时进行融合。Peter J. Burt等人在1983年的"The Laplacian Pyramid as a Compact Image Code"中提出了拉普拉斯金字塔用于图像压缩，后来该方法被用于图像融合效果很好

本文首先对拉普拉斯金字塔进行介绍，然后对拉普拉斯用于图像融合进行介绍

# Laplacian Pyramid

拉普拉斯金字塔通过将原始图像减去一个低通滤波的图像来消除像素间的相关性，得到一个净数据压缩，仅保留了差异。将这种做法在多尺度上进行迭代，会得到一个这种差异的金字塔，由于这种做法相当于用拉普拉斯算子对图像进行多尺度采样，因此得到的金字塔叫做拉普拉斯金字塔

该方法用于数据压缩时，仅需保留金字塔最小分辨率(最低层次)的低通图像以及一个拉普拉斯金字塔，就可以完全恢复原始图像。在重建原始图像时，将最小分辨率的低通图像逐层与拉普拉斯金字塔进行结合，最终获得压缩前的原始图像

下图是整个拉普拉斯金字塔的构建和重建原始图像的示意图

{% asset_img 01.png %}

该过程中主要涉及到以下几种处理

* 高斯滤波或高斯模糊(Gaussian Blur)

* 上采样(UpSample)

* 下采样(DownSample)

* 图像相加和相减

高斯滤波主要关注核大小与方差，下采样比较好做，按偶数或奇数行采样即可。上采样可以使用各种插值方法如三次样条插值等

## 拉普拉斯金字塔的生成

拉普拉斯金字塔的生成需要依赖高斯金字塔，上图中的黄色部分是生成高斯金字塔的过程。"Image"表示原始图像，以3层金字塔为例，G0即原始图像，经过高斯滤波和下采样得到尺度缩小为1/4的低通图像G1，重复该过程得到G2、G3，[G0, G1, G2, G3]构成了4层的高斯金字塔

拉普拉斯金字塔的每层由对应层的高斯金字塔图像与高斯金字塔下层图像的上采样相减得到，例如L3是由1/4原始尺度的G1经过上采样，在与G0相减得到。L2是由G2上采样与G1相减得到。实际需要使用的拉普拉斯金字塔只有3层，即[L1, L2, L3]

python代码如下

导入库

```python
import cv2
import copy
import numpy as np
from matplotlib import pyplot as plt
```

定义高斯金字塔生成函数

```python
def gaussian_pyramid(im, layers):
    GPyrs = [im]
    for i in range(1, layers):
        # 高斯滤波
        blur_im = cv2.GaussianBlur(GPyrs[i-1], (5,5), 0.83)
        # 下采样，取偶数行
        downsample = blur_im[::2, ::2]
        GPyrs.append(downsample)
    return GPyrs
```

定义拉普拉斯金字塔生成函数

```python
def laplace_pyramid(im, layers):
    GPyrs = gaussian_pyramid(im, layers)
    LPyrs = [GPyrs[-1]]
    for i in range(1, layers):
        # 上采样，使用opencv默认的采样方法
        size = (GPyrs[layers-i-1].shape[1], GPyrs[layers-i-1].shape[0])
        upsample = cv2.resize(GPyrs[layers-i], size)
        # 上层高斯图像与上采样后图像相减
        LPyrs.append(GPyrs[layers-i-1]-upsample)
    return LPyrs
```

定义金字塔信息打印与显示函数

```python
cv2.namedWindow('img', cv2.WINDOW_NORMAL)
dpi = 80

def layers_show(layers):
    for i, layer in enumerate(layers):
        l = layer.astype('uint8')
        # 色彩通道转换，opencv默认是BGR
        im = cv2.cvtColor(l, cv2.COLOR_BGR2RGB)
        h, w, _ = im.shape
        figsize = w/float(dpi), h/float(dpi)
        fig = plt.figure(figsize=figsize)
        ax = fig.add_axes([0, 0, 1, 1])
        ax.axis('off')
        plt.title('layer[{}] {}'.format(i, im.shape))
        ax.imshow(im)

def layers_info_print(layers, name):
    print('---- {}\'s info ----'.format(name))
    for i, layer in enumerate(layers):
        print('layer[{}] shape:{}'.format(i, layer.shape))
    print('\n')
```

读入图像生成4层的高斯金字塔并显示

```python
im = cv2.imread('images/cat.jpeg')
im.astype('float32')
GPyrs = gaussian_pyramid(im)
layers_info_print(GPyrs)
layers_show(GPyrs)
```

```shell
---- GPyrs's info ----
layer[0] shape:(607, 1080, 3)
layer[1] shape:(304, 540, 3)
layer[2] shape:(152, 270, 3)
layer[3] shape:(76, 135, 3)
```

G0层图像

{% asset_img 02.png %}

G1层图像

{% asset_img 03.png %}

G2层图像

{% asset_img 04.png %}

G3层图像

{% asset_img 05.png %}

生成4层拉普拉斯金字塔并显示

```python
LPyrs = laplace_pyramid(im, 4)
layers_info_print(LPyrs, "LPyrs")
layers_show(LPyrs)
```

```shell
---- LPyrs's info ----
layer[0] shape:(76, 135, 3)
layer[1] shape:(152, 270, 3)
layer[2] shape:(304, 540, 3)
layer[3] shape:(607, 1080, 3)
```

L0层图像

{% asset_img 06.png %}

L1层图像

{% asset_img 07.png %}

L2层图像

{% asset_img 08.png %}

L3层图像

{% asset_img 09.png %}

可以看到拉普拉斯图像中的每幅图像主要是纹理特征

## 原始图像重建

重建过程由最小分辨率开始的低通图像开始，经过上采样并与拉普莱斯金字塔对应分辨率图像相加，并经过多次向上迭代，最终得到原始图像并显示

```python
reconstruction = LPyrs[0]
for i in range(1, 4):
    size = (LPyrs[i].shape[1], LPyrs[i].shape[0])
    # 上采样，使用opencv默认的采样方法
    upsample = cv2.resize(reconstruction, size)
    # 上采样后图像与拉普拉斯图像相加
    reconstruction = LPyrs[i] + upsample
reconstruction = np.clip(reconstruction, 0, 255).astype('uint8')
plt.axis('off')
plt.title('reconstruction')
plt.imshow(cv2.cvtColor(reconstruction, cv2.COLOR_BGR2RGB))
```

结果如下图，能够正常复原原始图像

{% asset_img 10.png %}

# 图像融合

利用拉普拉斯金字塔进行图像融合是一个非常有趣的应用。融合后的图像在交界处的变化是非常平滑的，图像上的高频纹理混合得更快，可以避免 “鬼影”效果。为了创建融合图像，每幅图像首先生成自己的拉普拉斯金字塔，然后用一个二值掩模图像生成的高斯金字塔作为融合的权重金字塔，与两幅待融合的拉普拉斯金字塔进行结合，得到融合后的拉普拉斯金字塔，最后用融合后的拉普拉斯金字塔与最小分辨率的融合低通图像进行重建，最终得到融合的图像

加载左右两幅待融合图像以及二值掩模图像并显示

```python
layers = 7
left  = cv2.imread('images/blend/left.png')
right = cv2.imread('images/blend/right.png')
mask  = cv2.imread('images/blend/mask.png', 0)
plt.axis('off')
plt.title('left')
plt.imshow(cv2.cvtColor(left, cv2.COLOR_BGR2RGB))
```

{% asset_img 11.png %}

显示右图

```python
plt.axis('off')
plt.title('right')
plt.imshow(cv2.cvtColor(right, cv2.COLOR_BGR2RGB))
```

{% asset_img 12.png %}

显示二值掩模图像

```python
plt.axis('off')
plt.title('mask')
plt.imshow(mask, cmap='gray')
```

{% asset_img 13.png %}

图像预处理

```python
# 大小调整为256x256
left  = cv2.resize(left, (256, 256))
right = cv2.resize(right, (256, 256))
mask  = cv2.resize(mask, (256, 256)) 
# 数值定点化为int16，由于拉普拉斯计算减法存在负值
left  = left.astype('int16')
right = right.astype('int16')
mask  = mask.astype('float32') / 255
# 由于mask为灰度图，将其转化为3通道
mask = np.stack([mask, mask, mask], axis=-1)
```

计算left、right、mask的高斯金字塔，left和right的拉普拉斯金字塔

```python
# 生成高斯金字塔
left_G  = gaussian_pyramid(left, layers)
right_G = gaussian_pyramid(right, layers)
mask_G  = gaussian_pyramid(mask, layers)
# 生成互补的二值掩模
mask_G_r = []
for mask in mask_G:
    mask_G_r.append(1.0 - mask)
# 生成拉普拉斯金字塔
left_L  = laplace_pyramid(left, layers)
right_L = laplace_pyramid(right, layers)
```

打印各金字塔尺度

```python
layers_info_print(left_G, 'right_G')
layers_info_print(right_G, 'right_G')
layers_info_print(mask_G, 'mask_G')
layers_info_print(left_L, 'left_L')
layers_info_print(right_L, 'right_L')
```

```shell
---- left_G's info ----
layer[0] shape:(256, 256, 3)
layer[1] shape:(128, 128, 3)
layer[2] shape:(64, 64, 3)
layer[3] shape:(32, 32, 3)
layer[4] shape:(16, 16, 3)
layer[5] shape:(8, 8, 3)
layer[6] shape:(4, 4, 3)


---- right_G's info ----
layer[0] shape:(256, 256, 3)
layer[1] shape:(128, 128, 3)
layer[2] shape:(64, 64, 3)
layer[3] shape:(32, 32, 3)
layer[4] shape:(16, 16, 3)
layer[5] shape:(8, 8, 3)
layer[6] shape:(4, 4, 3)


---- mask_G's info ----
layer[0] shape:(256, 256, 3)
layer[1] shape:(128, 128, 3)
layer[2] shape:(64, 64, 3)
layer[3] shape:(32, 32, 3)
layer[4] shape:(16, 16, 3)
layer[5] shape:(8, 8, 3)
layer[6] shape:(4, 4, 3)


---- left_L's info ----
layer[0] shape:(4, 4, 3)
layer[1] shape:(8, 8, 3)
layer[2] shape:(16, 16, 3)
layer[3] shape:(32, 32, 3)
layer[4] shape:(64, 64, 3)
layer[5] shape:(128, 128, 3)
layer[6] shape:(256, 256, 3)


---- right_L's info ----
layer[0] shape:(4, 4, 3)
layer[1] shape:(8, 8, 3)
layer[2] shape:(16, 16, 3)
layer[3] shape:(32, 32, 3)
layer[4] shape:(64, 64, 3)
layer[5] shape:(128, 128, 3)
layer[6] shape:(256, 256, 3)
```

生成融合拉普拉斯金字塔

```python
blend_L = []
for i in range(layers):
    l = (mask_G_r[layers-i-1])*left_L[i] + (mask_G[layers-i-1])*right_L[i]
    blend_L.append(l.astype('int16'))
layers_info_print(blend_L, 'blend_L')
```

```shell
---- blend_L's info ----
layer[0] shape:(4, 4, 3)
layer[1] shape:(8, 8, 3)
layer[2] shape:(16, 16, 3)
layer[3] shape:(32, 32, 3)
layer[4] shape:(64, 64, 3)
layer[5] shape:(128, 128, 3)
layer[6] shape:(256, 256, 3)
```

计算最小分辨率低通融合图像

```python
start = (mask_G_r[-1])*left_G[-1] + (mask_G[-1])*right_G[-1]
start = start.astype('int16')
plt.axis('off')
plt.title('start')
plt.imshow(cv2.cvtColor(start.astype('uint8'), cv2.COLOR_BGR2RGB))
```

{% asset_img 14.png %}

进行融合

```python
blend = start
for i in range(1, layers):
    size = (blend_L[i].shape[1], blend_L[i].shape[0])
    # 上采样
    upsample = cv2.resize(blend, size)
    # 上采样图像与融合拉普拉斯对应层相加
    blend = blend_L[i] + upsample
    blend = np.clip(blend, 0, 255).astype('uint8')
plt.axis('off')
plt.title('blend')
plt.imshow(cv2.cvtColor(blend, cv2.COLOR_BGR2RGB))
```

最终融合后的图像如下

{% asset_img 15.png %}
