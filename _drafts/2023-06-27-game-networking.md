---
layout: post
title:  "游戏的网络同步技术基础课"
date:   2023-06-27
last_modified_at: 2023-06-27
categories: [game network]
---

## 原由
据说我以前做的那种休闲游戏不能被称为游戏，而我现在在做的 MMOARPG 才算是一种真正的游戏。好吧，这就是游戏行业的鄙视链。其实它们的核心差别就在于移动&战斗的逻辑上，往根源说，就是网络同步技术的轻度与重度使用的区别，而其他的方面呢？没啥区别了，反而休闲游戏往往是全区全服类型的，遇到的负载与存储的挑战会大很多，它往往需要做成一个正经的分布式系统。  

网上有很多讲网络同步技术的，但要么点到为止，要么过于啰嗦，要么比较陈旧。于是，我就想写一篇文章，帮助像我这样（有基础但对这块知识陌生）的人能快速切入这个领域。  

这篇文章不会是多么原创性的，就是一篇综述，会参考很多文章，但它会具备严谨性，起码这些知识都经过我实际的验证。  


## 需要探讨的问题
* 网络同步技术：帧同步、状态同步、快照同步
* aoi 算法
* 寻路算法
* 战斗
* 状态机的构建&维护


## 问题
* 游戏跟其他互联网应用的本质差别是什么？这种差别导致了技术上的什么差异？
游戏是关于互动的艺术，你可以跟环境互动（PVE）比如打怪打boss，也可以跟其他人互动（PVP）比如玩家pk，互动可以是1V1，1vN 或者 NVN 的。  
常见的互联网应用，其交互类型都是什么呢？


* 同步，到底是在同步什么？
其实就是游戏中各个实体的状态。状态包括什么？位置、动作、属性，就这些。  
那难在哪？实时性。
我们想要什么效果？


## 关于 AOI 算法
### 为什么需要 AOI 算法？
AOI = area of interest，它的目的就是只给特定对象发送它关注的消息。aoi 算法有灯塔法、九宫格法、十字链表法。


## 关于地图及地图的表示
1、常见的地图格式有哪些？  
2、2D 与 3D 的地图有何区别？  
3、地图数据在客户端与服务端之间有何区别？  


## 关于状态机
### 逻辑帧

### 渲染帧

### 帧率

### 逻辑帧跑不满会怎么样？


## 参考
[深入探索AOI算法](https://zhuanlan.zhihu.com/p/201588990)
[帧同步（LockStep）该如何反外挂](https://zhuanlan.zhihu.com/p/34014063)
[两种同步模式：状态同步和帧同步](https://zhuanlan.zhihu.com/p/36884005)
[网络同步技术](https://gameinstitute.qq.com/course/detail/10242)