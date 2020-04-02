---
layout:     post
title:      PBR简介以及Unity中的PBR
subtitle:   简单介绍一下PBR，以及PBR在Unity中的实现
date:       2019-11-04
author:     John Young
header-img: img/tag-bg-o.jpg
catalog: true
tags:
    - Unity
    - PBR
    - Computer Graphics
---

### 1. 什么是PBR
PBR即基于物理的渲染，Wikipedia上对于PBR的定义是这样的：
>“Physically based rendering (PBR) is an approach in computer graphics that seeks to render graphics in a way that more accurately models the flow of light in the real world. Many PBR pipelines have the accurate simulation of photorealism as their goal. Feasible and quick approximations of the bidirectional reflectance distribution function and rendering equation are of mathematical importance in this field. Photogrammetry(摄影测量法) may be used to help discover and encode accurate optical properties of materials. Shaders may be used to implement PBR principles.”
><p align="right">—— From Wikipedia</p>
其中可以得到几个信息：
* 按照真实世界中的光照建模
* 可以用双向反射分布函数（BRDF）描述
* 通常由shader实现

>“more of a concept than a strict set of rules.“
><p align="right">—— Joe Wilson</p>

>“Everything is shiny.”
><p align="right">——  John Hable</p>

### 2. Microfacet Theory(微表面理论)

### 3. Cook-Torrance BRDF

### 4. Unity PBR工作流

### 5. Unity PBR实现

### 6. Unity PBR三种实现效果对比

### 7. 引用
[https://www.cnblogs.com/wbaoqing/p/8931646.html](https://www.cnblogs.com/wbaoqing/p/8931646.html)
[https://blog.csdn.net/yangxuan0261/article/details/88749969](https://blog.csdn.net/yangxuan0261/article/details/88749969)
[https://zhuanlan.zhihu.com/p/33464301](https://zhuanlan.zhihu.com/p/33464301)
[https://en.wikipedia.org/wiki/Physically_based_rendering](https://en.wikipedia.org/wiki/Physically_based_rendering)
[https://zhuanlan.zhihu.com/p/21376124](https://zhuanlan.zhihu.com/p/21376124)
[https://docs.unity3d.com/Manual/StandardShaderMetallicVsSpecular.html](https://docs.unity3d.com/Manual/StandardShaderMetallicVsSpecular.html)
[https://zhuanlan.zhihu.com/p/66558476/](https://zhuanlan.zhihu.com/p/66558476/)
