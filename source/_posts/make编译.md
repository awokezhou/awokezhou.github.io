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

### make -j 并行编译

在多核CPU上利用"-j"参数并行编译，以加快编译速度

```shell
make -j4    /* 4个线程同时执行 */
```

### -Werror

编译过程中出现告警视为错误，立刻停止编译

## 输出调试

### $(warning)

利用$(warning)内置函数可以调试Makefile中的变量

```makefile
OBJS = main.o test.o debug.o
$(warning $(OBJS))
```

### @echo

如果在Makefile中打印一些提示过程字符串，需要使用echo，但是如果直接用echo，执行过程中会把这一行命令也打印出来

```makefile
%.o:%.c:
    echo "compile start"
    ......

# 实际会打印
# echo "compile start"
# compile start
```

命令前加"@"可以只输出命令执行结果，不输出命令本身

```makefile
%.o:%.c:
    @echo "compile start"
    ......

# 实际会打印
# compile start
```

## 输出编译辅助信息

### size

gcc编译工具"arm-linux-eabi-size"可以输出统计所有目标文件的占用大小

```makefile
$(OBJSIZE) $(OBJS) >> test.size

cat libtensorflow-microlite.size 
   text       data        bss        dec        hex    filename
......
```


