---
title: Python Program Skill
date: 2019-07-31 10:14:15
tags: [Python]
categories:
- Program Language
- Python
- comments: true
---

本文记录一些实际工作中使用到的Python语言编程技巧，或者学到的一些好用的用法

## 排序

### 矩阵按某列排序
应用场景：
* 线性回归
做线性回归时，要拟合某个变量与输出的关系曲线，如果输入训练集在该变量的维度上是乱序的，拟合出的不是曲线，而是一团乱序的线。因此需要以该变量重排列

方法：
* np.argsort
* sorted & lambda

#### np.argsort
```python
# example
x = np.array([[1,2,3],[4,5,6],[0,0,1]])
_sort_idx = np.argsort(x[:,0])  # 按第0列升序排序
x = x[_sort_idx].tolist()

# wrapper
def col_sort(x, col):
    _x, _sort_idx = np.array(x), np.argsort(_x[:,col])
    return _x[_sort_idx].tolist()
```

np.argsort()方法可以获取升序(默认)排序后的索引

#### sorted & lambda
```python
# example
x = [[1,2,3],[4,5,6],[0,0,1]]
x = sorted(x, key=lambda x:x[0])

x = [{'a':1, 'b':2}, {'a':2, 'b':1}]
x = sorted(x, key=lambda x:x['b'])

# wrapper
def col_sort(x, col):
    return sorted(x, key=lambda x:x[col])
```

这种方法输入矩阵每行可以是一个`list`，也可以是`dict`

