---
title: C-Namespace
date: 2020-07-13 10:36:21
tags: [Program Skill, C]
categories:
- Program Language
- C
comments: true
---

记录两种C语言实现命令空间的方法

## 结构体封装

简单来说就是将某个独立的库对外封装一个统一的接口结构体`structX`，外部调用时都使用`structX.aaa()`来调用库中的方法。例如有一个`foo`的库，要对外提供`test()`方法，可在`foo.h`文件中定义一个命名空间结构体`namespace_foo`，结构体中定义好需要对外提供的方法成员，并在最后通过`extern`关键字对外暴露结构体变量`Foo`

```c
/* foo.h */
typedef struct _namespace_foo {
    const char *name;
    const char *version;
    int (*test)();
} namespace_foo;
extern namespace_foo const Foo;
```

结构体变量`Foo`的定义在`foo.c`文件中

```c
/* foo.c */
static int foo_test()
{
    /**/
}

namespace_foo const Foo = {
    .name = 'Foo',
    .version = '1.0',
    .test = foo_test
}
```

外部要调用`foo`库中的函数，只需要引用`foo.h`头文件后，通过形如`Foo.test()`的方式就可以

```c
/* main.c */

#include "foo.h"


int main(int argc, char **argv)
{
    Foo.test();

    return 0;
}
```

如果有另外一个库`goo`需要同时使用，只需要定义结构体变量`Goo`时的变量名称与`Foo`不同即可

## 利用ifdef

另一种方式是利用条件宏定义宏来重定义函数名称

```c
/* foo.c */


int foo_test()
{
    /**/
}
```

```c
/* foo.h */


int foo_test();

#ifdef NAMESPACE_FOO
#define test(...)    foo_test(__VA_ARGS__)
#endif
```

在外部使用`foo`库的函数前，需要通过宏声明`NAMESPACE_FOO`，然后再引用`foo.h`头文件，后续调用`test()`函数就等于调用`foo_test()`

```c
/* main.c */

#define NAMESPACE_FOO
#include "foo.h"

int main(int argc, char **argv)
{
    test();
}
```
