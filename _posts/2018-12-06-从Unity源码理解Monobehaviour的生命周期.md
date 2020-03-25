---
layout:     post
title:      从Unity源码理解Monobehaviour的生命周期
subtitle:   Update一定比LateUpdate先执行吗?
date:       2018-12-06
author:     John Young
header-img: img/tag-bg-o.jpg
catalog: true
tags:
    - Unity
    - MonoBehaviour
---

## 前言

Unity的Monobehaviour每个Unity程序员都不可能陌生，update, fixedUpdate, lateUpdate等等built-in函数也是信手拈来。Unity间程序员流传的一张MonoBehaviour的图更是被奉为圭臬。
然而这篇文章的结论可能会超过你的想象：一些看似self-explanatory的接口，可能并不按你想象的顺序执行。

而这一切，都是从Unity的源码中得到的答案。

### 一般认为的MonoBehaviour的生命周期

这张图可能入门Unity的时候大家都见过，当你对MonoBehaviour的执行顺序有疑问时，也往往会搜到这张图：
![](https://johnyoung404.github.io/img/monoBehaviour/lifeCycle.png)
其中比较重要的几个信息：

* 当MonoBehaviour被实例化时，会依次执行Awake、OnEnable、Start(如果没被调用过)
* 接着会依次执行FixedUpdate(1至n次，取决于fixed time step), Update和LateUpdate
* 其中一些协程的yield result也会在如图位置被调用

如果你不想深入了解这背后的机制，那这张图已经足够了，因为它描述了99%的事实。
但是如果你手头有Unity源码，并且研究过相关的实现，你可能会有不一样的理解。

### 从PlayerLoop说起

如果你想开始阅读Unity的代码，那么你一定离不开PlayerLoop这个函数。这里Player指的播放器，可以是Editor Player，Android Player，或者Windows Player等等，它最后会被编译为跨平台的代码。在Player的一次Loop里，做的就是一次绘制，或者用另一种说法，一次loop就是一帧。在一次PlayerLoop的循环里，包括脚本、物理、音频、图形、垂直同步(vSync)等等几乎所有模块的更新。

这个函数大概长这样：
![](https://johnyoung404.github.io/img/monoBehaviour/PlayerLoop.png)

对于这篇文章的主题，我们要关注其中几个函数的执行顺序。在后面的DelayedCallManager中我们再具体讨论。