---
title: "Golang array and slice"
date: 2018-11-05T18:28:59+08:00
draft: false
---

数组和slice长的很像，操作方式也都差不多，并且slice包含了数组的基本的操作方式，如下标、range循环，还有一些如len()则是多种类型共用，所以根据操作根本搞不清数组和切片的区别，能够看出区别的地方主要看如何声明的。

### 数组
数组的声明方式很单一，通常是下面的样子：
```go
a := [5]int{1:10, 2:20}
```
![array](https://user-gold-cdn.xitu.io/2018/8/12/1652e129d21cd1a4?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

Go语言的数组是一个值，Go语言中的数组是值类型，一个数组变量就表示着整个数组，意味着Go语言的数组在传递的时候，传递的是原数组的拷贝。

### slice
切片是一个很小的对象，是对数组进行了抽象，并提供相关的操作方法。切片有三个属性字段：长度、容量和指向数组的指针。其声明方式比较多:
```go
var slice1 = []int{1, 2, 3, 4, 5}
var slice2 = make([]int, 0, 5)
var slice3 = make([]int, 5, 5)
var slice4 = make([]int, 5)
```
其中make(t type, len int, cap int), len表示make之后返回的slice的length，cap是容量。slice2和slice3是不一样的，slice2的内容为[], 而slice3的内容为[0, 0, 0, 0, 0]

![slice](https://user-gold-cdn.xitu.io/2018/8/12/1652e1398633fefe?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### 扩容
slice这种数据结构便于使用和管理数据集合，可以理解为是一种“动态数组”，slice也是围绕动态数组的概念来构建的。既然是动态数组，那么slice是如何扩容的呢？

请记住以下两条规则：

* 如果切片的容量小于1024个元素，那么扩容的时候slice的cap就翻番，乘以2；一旦元素个数超过1024个元素，增长因子就变成1.25，即每次增加原来容量的四分之一。
* 如果扩容之后，还没有触及原数组的容量，那么，切片中的指针指向的位置，就还是原数组，如果扩容之后，超过了原数组的容量，那么，Go就会开辟一块新的内存，把原来的值拷贝过来，这种情况丝毫不会影响到原数组。

