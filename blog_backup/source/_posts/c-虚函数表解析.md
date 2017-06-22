title: c++虚函数表解析
date: 2016-03-26 03:28:20
tags:
---

## 前言
本篇文章是参考陈浩大神的[C++虚函数表解析](http://blog.csdn.net/haoel/article/details/1948051)而写，文中的图片也是从陈浩大神的文中保存下来。本文为了写下来做一个笔记。如果读者想追溯求源，请使用上述链接查看陈浩大神的原文。

c++中的虚函数是为了实现多态的机制，即使用父类的指针指向子类的对象，当使用父类的指针调用实际子类的成员函数时，这样父类的指针就有了多种形态。例如有两个子类实现了父类的虚函数，那么父类的指针根据指向的不同子类对象，调用子类成员函数时就可能有两种实现方式。这其实就是一种泛型技术，说白了就是使用不变的代码实现可变的算法，例如：模板、RTTI、虚函数，这些要么是做到编译时决议（模板），要么是做到运行时决议（虚函数表）。

下面所有例子都是在linux2.6.32-358 x86_64 + gcc 4.4.7之上做的实验和得出的结论
源码地址为[源码地址](https://github.com/houbin/test_thought/tree/master/c%2B%2B/class_vtable)。

<!-- more -->

## 虚函数表
实验代码文件为base_class_vtable.cc

打印虚函数表中的所有虚函数及其地址，如下所示。

``` ```
虚函数表地址: 0x0x7fffc3c36000
虚函数表的第一个函数地址: 0x0x400bf0
Base::f
Base::g
Base::h

```

结论为

- 类的所有虚函数是保存在一个虚函数数组中。

- 虚函数数组中各个虚函数的顺序即为函数声明的顺序。

- 该类地址空间的首地址保存的是该虚函数数据的首地址。

虚函数如下：

![base_class_vtable](https://raw.githubusercontent.com/houbin/MarkdownPhotos/master/res/c%2B%2B%E8%99%9A%E5%87%BD%E6%95%B0%E8%A1%A8%E8%A7%A3%E6%9E%90/vtable.jpg)

## 一般继承（无虚函数覆盖）
该例子讲的是子类不实现父类的虚函数，而且又定义了虚函数的情况。

代码文件为derive_with_no_vfun_overlap.cc。

继承关系如下:

![继承图](https://raw.githubusercontent.com/houbin/MarkdownPhotos/master/res/c%2B%2B%E8%99%9A%E5%87%BD%E6%95%B0%E8%A1%A8%E8%A7%A3%E6%9E%90/derive_relation_with_no_vfun_overlap.jpg)

程序打印出虚函数表的顺序为
``` ```
第0个函数地址: 0x4197010
Base::f
第1个函数地址: 0x4197052
Base::g
第2个函数地址: 0x4197094
Base::h
第3个函数地址: 0x4197136
Derive::f1
第4个函数地址: 0x4197178
Derive::g1
第5个函数地址: 0x4197220
Derive::h1
encounter addr 0, end
```

结论为：
- 虚函数按照其声明顺序放在虚函数表中
- 子类的虚函数在父类虚函数的后面

虚函数表如下:

![虚函数表](https://raw.githubusercontent.com/houbin/MarkdownPhotos/master/res/c%2B%2B%E8%99%9A%E5%87%BD%E6%95%B0%E8%A1%A8%E8%A7%A3%E6%9E%90/derive_vtable_with_no_vfun_overlap.jpg)

## 一般继承（有虚函数覆盖）

该例子是子类实现了父类的虚函数，而且又定义了虚函数。

代码文件为derive_with_vfun_overlap.cc

继承关系如下:

![继承图](https://raw.githubusercontent.com/houbin/MarkdownPhotos/master/res/c%2B%2B%E8%99%9A%E5%87%BD%E6%95%B0%E8%A1%A8%E8%A7%A3%E6%9E%90/derive_relation_with_vfun_overlap.jpg)

程序打印出虚函数表的顺序为

``` ```
第0个函数地址: 4197128
Derive::f
第1个函数地址: 4197044
Base::g
第2个函数地址: 4197086
Base::h
第3个函数地址: 4197170
Derive::g1
第4个函数地址: 4197212
Derive::h1
encounter v_table end
```

结论
- 子类实现的虚函数f()覆盖了父类的f()
- 没有被子类实现的虚函数顺序依旧

虚函数表如下:

![虚函数表](https://raw.githubusercontent.com/houbin/MarkdownPhotos/master/res/c%2B%2B%E8%99%9A%E5%87%BD%E6%95%B0%E8%A1%A8%E8%A7%A3%E6%9E%90/derive_vtable_with_vfun_overlap.jpg)

## 多重继承（无虚函数覆盖）
代码文件为multi_derive_with_no_vfun_overlap.cc

继承关系如下:

![继承图](https://raw.githubusercontent.com/houbin/MarkdownPhotos/master/res/c%2B%2B%E8%99%9A%E5%87%BD%E6%95%B0%E8%A1%A8%E8%A7%A3%E6%9E%90/multi_derive_relation_with_no_vfun_overlap.jpg)

程序打印出虚函数的顺序为

``` ```
虚函数表 [0, 0] 地址为: 4197090
Base::f
虚函数表 [0, 1] 地址为: 4197132
Base::g
虚函数表 [0, 2] 地址为: 4197174
Base::h
虚函数表 [0, 3] 地址为: 4197468
Derive::f1
虚函数表 [0, 4] 地址为: 4197510
Derive::g1
虚函数表 [1, 0] 地址为: 4197216
Base2::f
虚函数表 [1, 1] 地址为: 4197258
Base2::g
虚函数表 [1, 2] 地址为: 4197300
Base2::h
虚函数表 [2, 0] 地址为: 4197342
Base3::f
虚函数表 [2, 1] 地址为: 4197384
Base3::g
虚函数表 [2, 2] 地址为: 4197426
Base3::h
```

结论为：

- 继承多个父类，多个父类的虚函数放在不同的虚函数数组中
- 子类自己的虚函数放在第一个父类的虚函数表中

虚函数表如下:

![虚函数表](https://raw.githubusercontent.com/houbin/MarkdownPhotos/master/res/c%2B%2B%E8%99%9A%E5%87%BD%E6%95%B0%E8%A1%A8%E8%A7%A3%E6%9E%90/multi_derive_vtable_with_no_vfun_overlap.jpg)

## 多重继承(有虚函数覆盖)

源码文件multi_derive_with_vfun_overlap.cc

继承关系如下:

![继承图](https://raw.githubusercontent.com/houbin/MarkdownPhotos/master/res/c%2B%2B%E8%99%9A%E5%87%BD%E6%95%B0%E8%A1%A8%E8%A7%A3%E6%9E%90/multi_derive_relation_with_vfun_overlap.jpg)

程序打印出虚函数的顺序为

``` ```
虚函数表 [0, 0] 地址为: 4197440
Derive::f
虚函数表 [0, 1] 地址为: 4197092
Base::g
虚函数表 [0, 2] 地址为: 4197134
Base::h
虚函数表 [0, 3] 地址为: 4197482
Derive::g1
虚函数表 [1, 0] 地址为: 4197434
Derive::f
虚函数表 [1, 1] 地址为: 4197218
Base2::g
虚函数表 [1, 2] 地址为: 4197260
Base2::h
虚函数表 [2, 0] 地址为: 4197428
Derive::f
虚函数表 [2, 1] 地址为: 4197344
Base3::g
虚函数表 [2, 2] 地址为: 4197386
Base3::h
```

结论为
- 子类自己的虚函数会放在第一个父类的虚函数表中
- 子类实现父类的虚函数会覆盖所有父类的虚函数表

虚函数表如下:

![虚函数表](https://raw.githubusercontent.com/houbin/MarkdownPhotos/master/res/c%2B%2B%E8%99%9A%E5%87%BD%E6%95%B0%E8%A1%A8%E8%A7%A3%E6%9E%90/multi_derive_vtable_with_vfun_overlap.jpg)
