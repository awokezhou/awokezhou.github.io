---
title: make编译
date: 2019-06-27 09:34:16
tags: [命令]
categories:
- 嵌入式
- 编译
comments: true
---

## make命令的一些解释

* make -j 并行编译
在多核CPU上利用"-j"参数并行编译，以加快编译速度
```shell
make -j4    /* 4个线程同时执行 */
```

* -Werror
编译过程中出现告警视为错误，立刻停止编译
