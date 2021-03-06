---
title: "C++前向声明"
date: 2016-01-01T21:07:16+08:00
draft: false
---
#### 为什么需要前向声明?
编译器确保在文件中使用的函数没有拼写错误或参数个数不对，因此它坚持要在使用函数之前要看到它的声明，
也就是为了方便编译器生成目标代码，不至于编译成功，运行的时候却失败。比如下面的例子：

```cpp
// file func.cpp
#include <stdio.h>

void func(int a, float b) {
    (void)a;
    (void)b;
    printf("func(int a, float b)\n");
}
```

```cpp
// file main.cpp
#include <stdio.h>

void func(int a, int b) {
    (void)a;
    (void)b;
    printf("func(int a, int b)\n");
}
// void func(int a, float b);

int main(int argc, char** argv) {
    func(1, 3.0f);
    return 0;
}
```

我们本意想调用`void func(int a, float b)`，但是程序执行的时候却调用了`void func(int a, int b)`，这种错误
很难发现，因为在编译的时候没有报错。

如果我们把`main.cpp`改成下面的：

```cpp
#include <stdio.h>

void func(int a, int b) {
    (void)a;
    (void)b;
    printf("func(int a, int b)\n");
}

void func(int a, float b);

int main(int argc, char** argv) {
    func(1, 3.0f);
    return 0;
}
```

程序就可以正确的调用`void func(int a, float b)`，因此前向声明还是很有必要的。也就是前向声明帮助编译器做出正确的决策。

#### 前向声明可以减少编译时间
我们可以通过`#include`来得到结构体、类或函数的声明，但是这样会减慢编译时间。如果一个`.h`文件包含很多声明，但是我们只需要使用其中的一两个，如果使用`#include`，这样生成的中间文件会很大，编译时间会增多。用前向声明可以避免这个问题。如果工程很大的话，这个问题更明显，使用前向声明往往可以把编译时间从几个小时减小到几分钟。

#### 打破循环`#include`
如果两个类的声明都需要用到对方，用通常的`#include`来引入各自的头文件，可能导致循环引用的问题。比如：

```cpp
// file Car.h
#include "Wheel.h"  // Include Wheel's definition so it can be used in Car.
#include <vector>

class Car
{
    std::vector<Wheel> wheels;
};
```

```cpp
// file Wheel.h
#include "Car.h"
class Wheel
{
    Car* car;
};
```
而且通过`#ifndefine`是解决不了这个问题的。这时候就需要通过前向声明来解决这个问题了。修改wheel.h如下：

```cpp
// file Wheel.h
class Car;
class Wheel
{
    Car* car;
};
```

#### 前向声明的局限
有些时候，class的完整定义是必需的，例如要访问calss的成员，或者要知道class的大小以便分配空间。这时候前向声明是不行的，
只能用`#include`, 以下几种不需要看到其完整定义的：

- 定义或声明`Foo*`和`Foo&`, 包括用于函数参数、返回类型、局部变量、类成员变量等。这是因为C++的内存模型是flat的，Foo的定义
无法改变Foo的指针或引用的含义。
- 声明一个以`Foo`为参数或返回类型的函数，如`Foo bar()`或`void bar(Foo f)`, 但是，如果代码里调用这个函数就需要知道Foo的定义，
因为编译器要使用Foo的拷贝构造函数和析构函数，因此至少需要看到他们的声明(虽然构造函数没有参数，但有可能位于`private`区)。