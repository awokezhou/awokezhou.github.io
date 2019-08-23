---
title: Autoencoder
date: 2019-08-13 10:55:32
tags: [ML]
categories:
- ML
- 理论
mathjax: true
---

自编码器(Autoencoder)是一种神经网络，它试图通过训练，将输入复制到输出。在其内部有一个隐藏层 $$ h $$ ，代表了对输入的编码。网络可以看作由两部分组成
* 编码器(encoder)：$$ h = f(x) $$
* 解码器(decoder)：$$ r = g(x) $$
  
Autoencoder并不会设计成输出对输入的完美复制 $$ g(f(x)) = x $$，因为这样的网络是没有意义的，相反，网络通常被刻意限制，只能近似复制。由于模型必须考虑选择输入的哪些部分来复制，因此可用于选择出输入的某些特性。网络需要在以下两件事上做平衡
* 对输入足够敏感，能够准确地重构输入
* 对输入不够敏感，模型不能简单地记忆或过度拟合训练数据

一般来说，Autoencoder主要用于数据降维和特征学习。最近，由于Autoencoder与潜变量模型之间的理论联系，Autoencoder被带到了生成概率模型的前沿。Autoencoder可看作是特殊的前向反馈网络，也可以使用再循环来训练

本文介绍以下自编码器
* Undercomplete Autoencoders
* Regularized Autoencoders
* Denoising Autoencoders
* Contractive Autoencoders

## Undercomplete Autoencoders
如果关心网络的隐藏层，而不关心解码器的输出，将会希望训练Autoencoder执行输入复制任务导致 $$ h $$ 具有有用的属性。约束网络的一种方式是让 $$ h $$ 比输入 $$x$$ 维度更小。这种编码维度小于输入维度的Autoencoder被称为**Undercomplete**。Undercomplete能够学习输入最显著的特征，以最小化损失函数来描述学习模型：

\begin{equation}
L(x,g(f(x)))
\end{equation}

其中 $$L$$ 是损失函数，用来惩罚 $$g(f(x))$$ 和 $$x$$ 的不相似性，例如使用均方误差MSE。

当解码器是线性的，$$L$$ 是MSE时，Undercomplete与PCA类似，其任务是学习训练数据的主子空间。采用非线性编码函数 $$f$$ 和非线性解码器函数 $$g$$ 的Undercomplete可以学习到比PCA更强的非线性泛化能力。但是如果编解码的容量过大，Undercomplete就无法提取有用信息了

## Regularized Autoencoders
