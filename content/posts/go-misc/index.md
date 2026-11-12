+++
date = '2026-11-12T18:42:55+08:00'
title = 'Go 杂项'
weight = -1
categories = ['Language']
tags = ['Go']
+++

## 隐式 interface implementation

Go 的 interface 是鸭子类型，不需要显式声明一个类型实现了某个 interface，只要该类型实现了 interface 中定义的所有方法，就被认为实现了该 interface

因此，Go interface 一般在调用它的包中定义，而不是在实现它的包中定义：调用方不需要在代码中写出对实现方的依赖，只需要 main（或者说，上层）将两者关联起来即可，因而容易写出低耦合的代码

比如，正在撰写一个包 b，需要一个能提供 `StoreStr(s string)` 方法的东西，只需撰写以下内容：

```go
package b

type B struct {
    ... // other things
    storer StringStorer
}

type StringStorer interface {
    StoreStr(s string)
}

func NewB(s StringStorer) *B {
    return &B{storer: s}
}
```

那么，只要有任意实现了 `StoreStr(s string)` 方法的类型 A，上层就可以用它来构造 B，给 B 提供能力，而 B 并不关心这个 A 是什么东西

```go
package main

import (
    "app/internal/a"
    "app/internal/b"
)

func main() {
    aType := a.NewA()
    bType := b.NewB(aType)
    ...
}
```

## 变量逃逸分析

Go 编译器会自动分析变量的生命周期，决定它们是分配在栈上还是堆上，考虑以下很傻的函数：

```go
func int2Float64(i int) *float64 {
    var f float64 = float64(i)
    return &f
}
```

以及它的等价 C++ 版本：

```cpp
double* int2Float64(int i) {
    double f = (double)i;
    return &f;
}
```

作为带 gc 的语言，Go 编译器会将变量 `f` 分配在堆上；而 C++ 只会将非 static 的局部变量 `f` 分配在栈上，函数返回后已被释放，返回悬垂指针

在这种情况下，C++ 应使用 `double* f = new double(i)` 来分配堆内存，并手动管理该变量的生命周期（当然，这是一种很不好的做法）

> Go 中的 new() 函数的作用仅仅是分配内存并返回指针，等价于 `var p T; return &p`，与该内存分配在哪无关
