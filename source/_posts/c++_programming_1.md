---
title: C++Programming(一)
date: 2022-07-13 16:00:00
categories: c++
tags: [c++]
---

# C++ Programming(一)

## 可变参数模板

可变参数模板（variadic templates）是C++11新增的特性之一，它对参数进行了高度泛化，它能表示0到任意个数、任意类型的参数。相比C++98/03，类模板和函数模板中只能含固定数量的模板参数，可变模板参数无疑是一个巨大的改进。然而由于可变模板参数比较抽象，使用起来需要一定的技巧，所以它也是C++11中最难理解和掌握的特性之一。

参考：

https://zh.m.wikipedia.org/zh-hans/%E5%8F%AF%E5%8F%98%E5%8F%82%E6%95%B0%E6%A8%A1%E6%9D%BF

https://www.cnblogs.com/qicosmos/p/4325949.html

<!-- more -->

## 声明

C++11之前，[模板](https://zh.m.wikipedia.org/wiki/模板_(C%2B%2B))（类模板与函数模板）在声明时必须有固定数量的模板参数。

C++11允许模板定义有任意类型任意数量的模板参数。

可变参数模板和普通模板的语义是一样的，只是写法上稍有区别，声明可变参数模板时需要在typename或class后面带上省略号“...”。比如我们常常这样声明一个可变模板参数：template<typename...>或者template<class...>，一个典型的可变模板参数的定义是这样的：
``` c++
template <class... T>
void f(T... args);
```

f函数的入参类型是T，后面带有省略号，表示是一个可变参数类型，带省略号的参数称为“参数包”。

## 可变参数模板类

例如，STL的类模板`tuple`可以有任意个数的类型名（typename）作为它的模板形参（template parameter）:
``` c++
template<typename... Values> class tuple;
```

如实例化为具有3个不同类型的实参（type argument）：
``` c++
tuple<int, std::vector<int>, std::map<std::string, std::vector<int>>> some_instance_name;
```

可以有0个参数：
``` c++
tuple<> some_instance_name;
```

当希望至少有一个入参，就需要显式的在模板里声明第一个参数：
``` c++
template<typename First, typename... Values> class tuple;
```
