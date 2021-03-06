---
title: C++重载决议
key: t20181011
tags:
  - C/C++
---

C++支持函数重载和运算符重载，在发生函数调用时，编译器需要从一堆重载的函数中选择出一个合理的进行调用，该过程被称作重载决议（Overload resolution）。

<!--more-->

## 框架

要编译一个函数调用，编译器必须先对函数名进行名字查找，得到一些可能被调用的候选函数（candidate function）。若候选函数不止一个，就需要进行重载决议来进行选择；若找不到候选函数，当然就报错了。

直觉上，候选函数的形参和调用时使用的实参越匹配，该候选函数就越会被优先选择。但是，这里面有许多隐晦的规则，它们可能让我们一脚踏坑里，也可能发生化学反应，造成语言设计者也意想不到的效果。

## 候选函数

若函数调用表达式`E(args)`中`E`是一些重载函数/函数模板的名字（即不考虑`E`是个可调用对象的情况），那么按下面的规则进行名字查找：

* 若`E`的形式是`pA->B`或`A.B`，其中`A`是个类类型（记为`cv T`），那么将会在`T`的成员函数中查找`B`。所有的查找结果都将成为候选函数，且成员函数调用所隐式使用的对象也会被作为一个隐式参数纳入到重载决议的过程中。
* 若`E`是一个初等表达式，那么按普通的函数调用时的规则进行名字查找。这一查找得到的结果一定符合以下情形之一：
  1. 是非成员函数。
  2. 是某个类`T`的成员函数。此时若`this`在作用域内，且指向`T`或`T`的派生类，那么`*this`被作为该函数调用的一个隐式参数；否则，一个`T`类型的dummy对象被作为隐式参数，若重载决议最终选择了一个非静态成员函数，编译不通过。

若函数调用表达式`E(args)`中的`E`具有类类型`cv T`，那么：

* 在表达式`(E).operator()`中进行普通的名字查找，找到的每一个声明都将被加入候选函数中。

* 对`T`或`T`的未被隐藏的基类中的每个用户自定义转换函数，若该转换的cv限定符不弱于`T`的cv限定符，且转换目标是以下三者中的一种：

  1. 转换为函数指针
  2. 转换为函数指针的引用
  3. 转换为函数引用

  那么一个具有独有名字的代理调用函数（surrogate call function）会被加入候选函数中，该函数的第一个参数具有转换结果的类型，剩余参数和转换结果所接收的参数一致。

用户自定义类型转换可以这样用于函数重载，这是我过去未曾想过的：

```cpp
int foo(int);
struct A { operator int(*)(int)() { return foo; } } a;
int x = a(5);
```

