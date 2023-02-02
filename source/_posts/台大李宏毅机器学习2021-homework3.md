---
title: 台大李宏毅机器学习2021-homework3
date: 2022-04-23 20:07:24
tags: [ML, CNN, Resnet]
categories:
- AI
- 台大李宏毅机器学习2021
---

homework3的任务是创建一个用于图像分类的CNN网络，其中可能会用到一些优化技巧。任务包含以下3个级别的要求

* Easy：Baseline模型，由作业提供，直接训练可以达到44.862%的精度

* Middle：需要使用数据增强技术或者优化网络结构(不可以使用预训练好的网络架构，不能使用额外的数据集)，以进行性能的提升，目标精度52.807%

* Hard：使用提供的未标记数据获得更好的结果，目标精度82.138%

# Dataset

作业使用的数据集是特别处理过的food-11数据集，其中收集了11个食物类别

* 训练集：280个已标记数据和6786个未标记数据

* 验证集：60已标记数据

* 测试集：3347个数据

其中的类别编码为

* 00：披萨、鸡肉卷、汉堡、土司、面包

* 01：黄油、芝士、奶酪

* 02：蛋糕、马卡龙、

* 03：鸡蛋、煎蛋

* 04：油炸类食物

* 05：烤肉

* 06：意面

* 07：米饭、炒饭

* 08：生蚝、扇贝、三文鱼、北极贝

* 09：各种汤

* 10：蔬菜水果

# Baseline

首先加载各个依赖模块

```python
import torch
import torchvision
import torchvision.transforms as transforms
import torch.nn as nn
import numpy as np
import matplotlib.pyplot as plt
from torchvision.datasets import DatasetFolder
from torch.utils.data import ConcatDataset, DataLoader, Subset
from PIL import Image
from tqdm.auto import tqdm
```

## 数据准备

### 数据变换

定义数据转换方式为仅进行resize操作

```python
train_tfm = transforms.Compose([
    transforms.Resize((128, 128)),
    transforms.ToTensor(),
])

test_tfm = transforms.Compose([
    transforms.Resize((128, 128)),
    transforms.ToTensor(),
])
```

### DataLoader

设置batchsize为16，训练集和验证集进行shuffle操作，测试集不进行shuffle操作

```python
batch_size = 16

train_set = DatasetFolder("food-11/training/labeled", loader=lambda x: Image.open(x), extensions="jpg", transform=train_tfm)
valid_set = DatasetFolder("food-11/validation", loader=lambda x: Image.open(x), extensions="jpg", transform=test_tfm)
unlabeled_set = DatasetFolder("food-11/training/unlabeled", loader=lambda x: Image.open(x), extensions="jpg", transform=train_tfm)
test_set = DatasetFolder("food-11/testing", loader=lambda x: Image.open(x), extensions="jpg", transform=test_tfm)

train_loader = DataLoader(train_set, batch_size=batch_size, shuffle=True, num_workers=0, pin_memory=False)
valid_loader = DataLoader(valid_set, batch_size=batch_size, shuffle=True, num_workers=0, pin_memory=False)
test_loader = DataLoader(test_set, batch_size=batch_size, shuffle=False)
```

## 模型定义

作业给出的基线模型为3个卷积层加3个FC，每个卷积层后跟一个BN、一个ReLU和一个MaxPool

```python
class MyModel(nn.Module):
    def __init__(self):
        super(Classifier, self).__init__()
        # input image size: [3, 128, 128]
        self.cnn_layers = nn.Sequential(
            # Conv2D_1
            nn.Conv2d(3, 64, 3, 1, 1),
            nn.BatchNorm2d(64),
            nn.ReLU(),
            nn.MaxPool2d(2, 2, 0),
            # Conv2d_2
            nn.Conv2d(64, 128, 3, 1, 1),
            nn.BatchNorm2d(128),
            nn.ReLU(),
            nn.MaxPool2d(2, 2, 0),
            # Conv2d_3
            nn.Conv2d(128, 256, 3, 1, 1),
            nn.BatchNorm2d(256),
            nn.ReLU(),
            nn.MaxPool2d(4, 4, 0),
        )

    def forward(self, x):
        # input (x): [batch_size, 3, 128, 128]
        # output: [batch_size, 11]
        x = self.cnn_layers(x)
        x = x.flatten(1)
        x = self.fc_layers(x)
        return x
```

模型结构如下图所示

{% asset_img 01.png %}

## Training

损失函数使用交叉熵，优化器使用Adam，优化器学习率为0.0003，训练批次为80

```python
device = 'cpu'
model = MyModel().to(device)
model.device = device

criterion = nn.CrossEntropyLoss()
optimizer = torch.optim.Adam(model.parameters(), lr=0.0003, weight_decay=1e-5)

n_epochs = 80

for epoch in range(n_epochs):
    model.train()
    train_loss = []
    train_accs = []
    for batch in tqdm(train_loader):
        imgs, labels = batch
        logits = model(imgs.to(device))
        loss = criterion(logits, labels.to(device))
        optimizer.zero_grad()
        loss.backward()
        grad_norm = nn.utils.clip_grad_norm_(model.parameters(), max_norm=10)
        optimizer.step()
        acc = (logits.argmax(dim=-1) == labels.to(device)).float().mean()
        train_loss.append(loss.item())
        train_accs.append(acc)
    train_loss = sum(train_loss) / len(train_loss)
    train_acc = sum(train_accs) / len(train_accs)
    print(f"[ Train | {epoch + 1:03d}/{n_epochs:03d} ] loss = {train_loss:.5f}, acc = {train_acc:.5f}"

    model.eval()
    valid_loss = []
    valid_accs = []
    for batch in tqdm(valid_loader):
        imgs, labels = batch
        with torch.no_grad():
            logits = model(imgs.to(device))
        loss = criterion(logits, labels.to(device))
        acc = (logits.argmax(dim=-1) == labels.to(device)).float().mean()
        valid_loss.append(loss.item())
        valid_accs.append(acc)
    valid_loss = sum(valid_loss) / len(valid_loss)
    valid_acc = sum(valid_accs) / len(valid_accs)
    print(f"[ Valid | {epoch + 1:03d}/{n_epochs:03d} ] loss = {valid_loss:.5f}, acc = {valid_acc:.5f}")
```

经过80批次训练后，验证集精度为0.47173

```shell
[ Train | 080/080 ] loss = 0.04817, acc = 0.98478
[ Valid | 080/080 ] loss = 5.32257, acc = 0.47173
```

Loss曲线如下图所示，可以看出网络其实并没有收敛，验证集误差还在增大

{% asset_img 03.png %}

# Middle

## optimize1-数据增强

对训练集和验证集使用以下数据增强技术

* 随机水平/垂直翻转
* 随机旋转
* 亮度/对比度/饱和度变换

```python
train_tfm = transforms.Compose([
    # 随机亮度、对比度、饱和度
    transforms.ColorJitter(brightness=0.25, contrast=0.25, hue=0.25),
    # 随机水平翻转
    transforms.RandomHorizontalFlip(p=0.5),
    # 随机垂直翻转
    transforms.RandomVerticalFlip(p=0.5),
    # 随机旋转
    transforms.RandomRotation((-90, 90)),
    transforms.Resize((128, 128)),
    transforms.ToTensor(),
])
```

## optimize2-使用Resnet18

直接使用pytorch中定义好的Resnet18模型进行训练，但是不加载训练权重参数

```python
model = torchvision.models.resnet18(pretrained=False)
```

进行80个批次训练后，结果如下

```shell
[ Train | 080/080 ] loss = 0.76552, acc = 0.74223
[ Valid | 080/080 ] loss = 1.38644, acc = 0.59524
```

验证集精度达到了59.524%，训练曲线如下

{% asset_img 04.png %}

# Hard

使用半监督学习技术Semi-Supervised Learning，每次训练时从unlabeled数据中筛选以当前模型输出结果最大置信度超过某个阈值的样本，加入到labeled数据中，构成新的数据集。

模型使用resnet50，训练结果

```shell
[ Train | 080/080 ] loss = 0.70109, acc = 0.76370
[ Valid | 080/080 ] loss = 1.42967, acc = 0.57589
```

{% asset_img 05.png %}
