---
title: Python从PDF提取图片
date: 2022-04-29 13:10:04
tags: [Python]
categories:
- CS
- Python
---

阅读PDF格式的论文或者一些书籍时，经常希望将PDF文件中的图片提取出来，市面上有些PDF格式转换的工具可以做到，但是很多都要收费。通过Python的pymupdf库可以完成该功能

安装pymupdf

```shell
pip install pymupdf
```

这里要注意的是安装的库是pymupdf，但是实际使用的是pymupdf安装时带的fitz模块

首先是需要加载的库

```python
import os
import re
import time
import fitz
import tqdm
```

然后是定义pdf2image函数

```python
def pdf2image(pdf_path, img_path):
    pdf_name = pdf_path.split('.pdf')[0]
    start_time = time.time()
    count = 0
    checkXO = r"/Type(?= */XObject)"    # 判断是否为XObject的正则
    checkIM = r"/Subtype(?= */Image)"   # 判断是否为图像的正则
    doc = fitz.open(pdf_path)           # 获取pdf的对象树
    xref_length = doc.xref_length()     # 获取对象树的对象实例数
    print("processing {} pages:{} objs:{}".format(
        pdf_path, len(doc), xref_length - 1))

    # 遍历每个对象
    for i in tqdm.tqdm(range(1, xref_length)):
        text = doc.xref_object(i)
        isXObject = re.search(checkXO, text)
        isImage = re.search(checkIM, text)
        if not isXObject or not isImage:
            continue
        count += 1
        # 将对象转换为像素图
        pix = fitz.Pixmap(doc, i)
        new_name = pdf_name.replace('\\', '_') + "img{}.png".format(count)
        new_name = new_name.replace(':', '')
        if pix.n < 4:
            pix.save(os.path.join(img_path, new_name))
        else:
            pix0 = fitz.Pixmap(fitz.csRGB, pix)
            pix0.save(os.path.join(img_path, new_name))
            pix0 = None
    end_time = time.time()
    print("running time:{}s".format(end_time - start_time))
    print("get {} images".format(count))
```

定义一个从当前目录获取所有pdf文件列表的函数

```python
def get_pdf_files():
    files = os.listdir()
    pdf_files = []
    for f in files:
        if f.find('.pdf') >= 0:
            pdf_files.append(f)
    return pdf_files
```

定义main函数

```python
if __name__ == "__main__":
    pdf_files = get_pdf_files()
    print('PDF文件列表:')
    for f in pdf_files:
        print(f)

    # 循环对每个pdf文件进行图片提取
    for f in pdf_files:
        pdf_name = f.split('.pdf')[0]
        img_dir = "{}-images".format(pdf_name)
        # 为每个pdf创建一个 pdfname-images 的文件夹
        if not os.path.exists(img_dir):
            os.mkdir(img_dir)
        pdf2image(f, img_dir)

    input('Enter')
```

运行程序，以"MUSIQ: Multi-scale Image Quality Transformer"论文为例，获取到了如下图片

{% asset_img 01.PNG %}
