---
layout:     post
title:      从Unity源码理解Monobehaviour
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


>关键词：官方防沉迷最为致命

### 一般认为的MonoBehaviour的生命周期

这张图可能入门Unity的时候大家都见过，当你对MonoBehaviour的执行顺序有疑问时，也往往会搜到这张图。

![](https://johnyoung404.github.io/img/monoBehaviour/lifeCycle.png)

 

