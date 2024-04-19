---
layout: post
title: "现代 C++"
date: 2024-04-15
last_modified_at: 2024-04-15
categories: [lang]
---

学习一门语言，我觉得分两方面，一方面是了解如何使用，另一方面是了解内部原理。本文主要讲如何使用，会介绍现代 C++ 的一些最有用的新特性，并尝试筛选出一个适合日常开发的特性子集。  

关于现代 C++ 的书和文章遍地都是，但无论如何，每个人都应该归纳总结一下，形成自己的工具箱，于是有了此文。  

所谓现代 C++，指的是从 C++11 开始的 C++，从 C++11 开始，加入一些比较现代的特性，使得用 C++ 开发少了很多心智负担，程序也更加健壮。从 C++11 开始，每 3 年发布一个新版本，到今年（2024）已经有 5 个版本了，分别是 C++11、C++14、C++17、C++20、C++23，这 5 个版本引入了上百个新特性，这些特性使得 C++ 看起来像一门新语言了。   

不过，我时常还是感慨，以 C++ 作为主力语言的程序员有点可怜，整天要吃 C++ 这碗屎，还得夸它香。   

悲伤过后，下面分别列举各个版本的重要新特性。  

---

## C++11 新特性

C++11 是一个 major 版本，有不少新特性，有些是有用但不算很重要，个人认为最重要的特性包括：  

* 智能指针: unique_ptr, shared_ptr
* 移动语义
* 新的内存模型
* lambda 表达式
* auto 和 decltype 推导


#### 智能指针

#### 移动语义


#### lambda 表达式

---

## C++14 新特性
C++14 是一个 minor 版本，没什么重要的新特性，主要是在给 C++11 打补丁，为使用者 “带来极大方便”，实现 “对新手更为友好” 这一目标。 

与方便使用相关的特性，就是支持函数返回值推导了，在 C++11 中这么写是编译不过的，在 C++14 中支持了：  
```
auto func(int i) {
    return i;
}
```

---

## C++17 新特性
C++17 是一个 major 版本，

重要的特性包括：  
* 结构化绑定


#### 结构化绑定



---

## C++20 新特性
C++20 也是一个 major 版本，有很重要的更新，"The Big Four"，即四个重要的特性，分别是：概念、范围、协程和模块。  

由于平常用 lua 多，所以对于协程这个特性比较关注。C++20 实际上提供的是协程的基础设施，相当原始，是面向库开发者的，普通程序员应该使用基于这些基础开发的各种便于使用的库。比如 C++23 就引入了这样的库，叫 generator。  

我有点不太理解，在 lua 里面，协程挺简单的，到了 C++ 这里，怎么变得那么复杂，以至于人们要写那么冗长的文章来介绍它。  



---

## C++23 新特性

---


下面列举一些 C++ 常用的优化手段

## Pimpl





