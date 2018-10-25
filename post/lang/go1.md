---
title: "Golang defer return 返回值执行顺序总结"
date: 2018-10-25T19:35:11+08:00
draft: false
---
### 背景
项目中遇到一个小问题，我用到一个库，但是这个库在异常情况下内部会panic，虽然可以在最外层的函数recover住，让服务继续运行，但业务上需要如果这个库panic了，能够做一些逻辑处理，也就是类似java等语言的try...cache操作。

### 方案
新加一个函数对库函数进行包装，然后recover住panic，并且调用的地方能够感知到出错了。打算在将recover返回的error信息返回给调用函数。因此做了如下改造：
原函数:
```go
func (r *Request) ReplyRunOnceDataV2(statuscode int, contentType string, data []byte) (err interface{}) {
    defer func() {
        err = recover()
    }()
    r.ReplyRunOnceData(statuscode, contentType, data)
    return
}
```
注意，这里函数返回值定义了一个有名返回值，是基于如下golang基础知识：
```
1. 多个defer的执行顺序为“后进先出”；

2. 所有函数在执行RET返回指令之前，都会先检查是否存在defer语句，若存在则先逆序调用defer语句进行收尾工作再退出返回；

3. 匿名返回值是在return执行时被声明，有名返回值则是在函数声明的同时被声明，因此在defer语句中只能访问有名返回值，而不能直接访问匿名返回值；

4. return其实应该包含前后两个步骤：第一步是给返回值赋值（若为有名返回值则直接赋值，若为匿名返回值则先声明再赋值）；第二步是调用RET返回指令并传入返回值，而RET则会检查defer是否存在，若存在就先逆序插播defer语句，最后RET携带返回值退出函数；
```
因此，‍‍defer、return、返回值三者的执行顺序应该是：return最先给返回值赋值；接着defer开始执行一些收尾工作；最后RET指令携带返回值退出函数。

#### 匿名返回值的情况
```go
package main

import (
    "fmt"
)

func main() {
    fmt.Println("a return:", a()) // 打印结果为 a return: 0
}

func a() int {
    var i int
    defer func() {
        i++
        fmt.Println("a defer2:", i) // 打印结果为 a defer2: 2
    }()
    defer func() {
        i++
        fmt.Println("a defer1:", i) // 打印结果为 a defer1: 1
    }()
    return i
}
```

#### 有名返回值的情况
```go
package main

import (
    "fmt"
)

func main() {
    fmt.Println("b return:", b()) // 打印结果为 b return: 2
}

func b() (i int) {
    defer func() {
        i++
        fmt.Println("b defer2:", i) // 打印结果为 b defer2: 2
    }()
    defer func() {
        i++
        fmt.Println("b defer1:", i) // 打印结果为 b defer1: 1
    }()
    return i // 或者直接 return 效果相同
}
```
