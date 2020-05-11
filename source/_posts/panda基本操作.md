---
title: panda基本操作
date: 2019-05-27 14:35:10
tags: [Python]
categories:
- Program Language
- Python
- Pandas
comments: true
---

## DataFrame基本操作
### 删除
删除行或列使用的函数都是`drop`

#### 删除列
```python
df.drop('col_name', axis=1)
```
* col_name 是列名标识
* `axis`用来标识删除的是行还是列

#### 删除行
```python
df.drop([row_nr1, row_nr2, ...])
```
要删除的行号以`list`形式传入

## 时间序列数据处理

### 将字典列表转化为DataFrame

```python
import pandas as pd
x = [{'a':1, 'b':2, 'c':'2000-01-01'}, 
     {'a':2, 'b':3, 'c':'2000-01-02'}, 
     {'a':3, 'b':4, 'c':'2000-01-03'},
     {'a':4, 'b':5, 'c':'2000-01-04'},
     {'a':5, 'b':6, 'c':'2000-01-05'},
     {'a':6, 'b':7, 'c':'2000-01-06'},]
df = pd.DataFrame(x, columns=['a', 'b', 'c'])
```
查看数据
```bash
>>> df
   a  b           c
0  1  2  2000-01-01
1  2  3  2000-01-02
2  3  4  2000-01-03
3  4  5  2000-01-04
4  5  6  2000-01-05
5  6  7  2000-01-06
```

### 修改索引为日期
```python
df.set_index('c')
```
输出
```bash
>>> df.set_index('c')
            a  b
c
2000-01-01  1  2
2000-01-02  2  3
2000-01-03  3  4
2000-01-04  4  5
2000-01-05  5  6
2000-01-06  6  7
```

### DataFrame转化为csv文件