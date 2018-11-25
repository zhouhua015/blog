---
layout: post
title: 条件操作符
categories: 
  - c/cpp
tags:
  - details
  - whatithoughtiknew
date: 2018-11-24
---
<div style="text-align:center"><img src ="https://datev.pl/wp-content/uploads/2017/11/question-mark-2123969_1920-300x181.jpg" /></div>

条件运算符`?:`在很多语言中都存在，形如`exp1 ? exp2 : exp3`，实践中一般用来取代简单的`if-else`语句，使代码更简洁。

在`C/C++`代码中，条件运算符的使用很常见，将`if`条件判断

```c
if (a > b) {
  result = c;
} else {
  result = d;
}
```

转换成一行简单的赋值语句

```c
result = a > b ? c : d;
```

单行的语句和 `if`代码块作用一样，首先对条件`a>b`求值，如果结果为真，则对`c`求值并将结果赋给`result`，否则对`d`求值并赋给`result`。在条件运算符中，若`c`和`d`为表达式，则二者有且仅有一个会被求值。

条件运算符的结合性为从右到左，因此也可以支持`if-else if-else`语意

```c
result = x == 1 ? a :
         x == 2 ? b :
         x == 3 ? c : d;
```

[GNU C](https://gcc.gnu.org/onlinedocs/gcc/Conditionals.html#Conditionals)支持一种特殊的条件运算符，省略中间的`exp2`语句简化为
`exp1 ?: exp3`，如果`exp1`结果不为0，返回`exp1`，否则返回`exp3`的值。在形如

```c
x ? x : y
```

的逻辑中，如果`x`、`y`均为函数调用，写成

```c
x ?: y
```

可以在继续使用条件运算符的情况下避免函数`x`被二次调用。缩写后的操作符`?:`也称为[猫王操作符](https://en.wikipedia.org/wiki/Elvis_operator)。:-)

条件运算符在`C++`中有更多用处，在很多不可赋值的场景下能发挥更多用处，例如初始化一个变量的引用，

```cpp
int& x = a > b ? m : n;
```

或者根据条件使用不同的参数调用构造函数

```cpp
Cls c1(a > b ? m : n);
c1.foo();
```

与`C`语言更大的不同在于，`C++`中的条件运算符，如果`exp2`和`exp3`均为左值，其结果可作左值。

```cpp
(x > 1 ? m : n) = 42;
```

当`x`的值大于1时，42将赋值给`m`，反之则赋值给`n`。这个特性，再加上条件运算符`?:`与赋值运算符`=`优先级相同，使得`C++`的`x ? a : b = c`不等于`(x ? a : b) = c`，而实际上被解析为`x ? a : (b = c)`。

在其他语言中，条件运算符可能以不同的形式存在，例如`Python`的 `x = m if a > b else n`，也有一些语言不存在条件运算符，例如`Golang`[认为](https://golang.org/doc/faq#Does_Go_have_a_ternary_form)条件运算符在大多数语言中被滥用且存在理解困难，不提供支持（看看`if-else if-else`语意的条件运算符使用，感觉这个决定也有点道理......）。