---
title: csv读取注释
date: 2021-10-22 18:01:52
tags: Python
comments: true
---

Python的pandas库处理csv文件非常方便，开发过程中经常会用到csv文件，例如将csv中的数据转化为二进制、将csv文件转化为json等。由于csv本身是以列表的形式组织数据的，如果想要额外的加入一些描述信息，比如版本号等，应该如何做呢？

可以以key-value的形式在csv文件头部写入一些信息

```cpp
#Version:v1.0
#Data:2021/01/01
#Author:Jam
#Description: This is a example of csv commit
```

利用以下函数可以从csv文件中以字典的形式获取注释中的key-value

```python
def csv_read_attrs(filename):
    s = ''
    fobj = open(filename)
    while True:
        line = fobj.readline()
        if not line:
            break
        s += line
    attrs = re.findall(r'#(.*)', s)
    keys = [x.split(':')[0].strip() for x in attrs]
    values = [x.split(':')[1].strip() for x in attrs]
    attrs = dict(zip(keys,values))
    return attrs
```

将上述注释保存为test.csv文件，调用`csv_read_attrs`函数解析注释信息

```shell
>>> import csv_read_attrs from csvAttrs
>>> attrs = csv_read_attrs('test.csv')
>>> attrs
{'Version': 'v1.0', 'Data': '2021/01/01', 'Author': 'Jam', 'Description': 'This is a example of csv commit'}
```
