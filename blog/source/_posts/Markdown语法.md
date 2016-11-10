title: Markdown语法
date: 2016-02-16 11:57:56
tags:
---

## 1.标题

语法为

```
#       表示一级标题
##      表示二级标题
###     表示三级标题

大标题
=

小标题
-

```

<!-- more -->


## 2.列表

在markdown中，列表显示在文字前加上-或者*就可以变为无序列表，有序列表则直接在文字前加上1. 2.，符号和文字之间要有一个空格

## 3.引用
如果引用一小段别处的句子，就需要用到引用的格式，引用需要在文字前面加上 < 符号

> 我是引用格式

## 4.图片与链接
插入链接与插入图片的语法很像，区别在一个 ！ 符号

语法为

```
图片为： ![图片描述](图片链接)

链接为： []()
```

例如
![示例图片](https://raw.githubusercontent.com/houbin/MarkdownPhotos/master/res/weather-sunny.png)

## 5.粗体与斜体
markdown的粗体和斜体的语法很简单，用**包含的一段文本就是粗体的语法，用*包含的一段文本就是斜体的语法

例如
**我是粗体** *我是斜体*

## 6.表格

下面是markdown的表格语法

``` bash
|Tables         | Are           | Cool  |
| ------------- | ------------- | ----- |
| col 3 is      | right-aligned | $1600 |
| col 2 is      | centered      |   $12 |
| zebra stripes | are neat      |    $1 |
```

显示效果为

|Tables         | Are           | Cool  |
| ------------- |:-------------:| -----:|
| col 3 is      | right-aligned | $1600 |
| col 2 is      | centered      |   $12 |
| zebra stripes | are neat      |    $1 |


貌似不行，先跳过表格

## 7.代码框

这个程序员肯定会用的到的，语法是用\` \`将中间的代码包裹起来就行了

例如

`#define EXT4_NAME_LEN 255`

但是对于大段代码需要每行都加，不太方便

另外一个行之有效的方法是使用```将代码块包起来

``` c++
void EncodeFixed32(char* buf, uint32_t value) {
  if (port::kLittleEndian) {
    memcpy(buf, &value, sizeof(value));
  } else {
    buf[0] = value & 0xff;
    buf[1] = (value >> 8) & 0xff;
    buf[2] = (value >> 16) & 0xff;
    buf[3] = (value >> 24) & 0xff;
  }
}
```

## 8.分隔线

分隔线的语法需要用三个 * 符号来表示

---
***
~~文字删除线~~

## 9.创建链接

语法为\<test@domain.com\>

实际效果为

<test@domain.com>

## 10.转义字符

对于特殊字符，如* [ >等符号，可使用特殊字符\将这些特殊字符转义为正常打印字符


## 11.总结

以上便是markdown的基本语法，这些语法保证能够在markdown上存活，然后不断的practice


