---
layout:     post
title:      Real-Time Raytracing
subtitle:   实时光线追踪技术的一些介绍
date:       2018-08-14
author:     John Young
header-img: img/tag-bg-o.jpg
catalog: true
tags:
    - Raytracing
    - Computer Graphics
---

## 前言

最近几年光线追踪在计算机图形学领域被越来越多的讨论。随着Nvidia的RTX系列显卡的问世，还有微软发布新的图形接口DirectX Raytracing，以及Unity、EA、Epic Games在游戏引擎层面的积极支持，光线追踪似乎从学术研究为主慢慢走向了工业应用。如果说光线追踪是未来，那么未来可能已经到来了。图形程序员了解光线追踪的必要性也在日益增加。

### 1、为什么光线追踪这么吸引人

![](https://johnyoung404.github.io/img/raytracing/meme.jpg)

上图是一张流行于图形程序员之间的一张meme(类似于我们的表情包)。可以看到，我们已经有了挺不错的rasterization（光栅化），为什么我们还对Real-Time Raytracing保持这么大的兴趣呢，即使后者看起来有很大的缺陷（充满了噪点）。

这是二者的本质上的差异决定的。

正如Nvidia的光线追踪教程里写的：
> “Ray tracing is the only technology we know of that enables the rendering of truly photorealistic images.”

现在的光栅化的渲染管线，基于的是局部照明模型。一些光影效果，不可避免地受到这一点的制约，无法真实地表现，只能借助一些奇技淫巧(trick)。而光线追踪则是全局照明模型，符合真实世界的物理规律，渲染出来的图片往往能达到让人惊讶的真实效果。由于光线追踪耗时巨大，因此之前往往只被用于离线渲染（一般是基于CPU的），例如电影工业光影渲染、建筑行业效果图渲染等等。

例如下图就是小时候当时带给我震撼的一部3D电影《变形金刚》，有些frame据说要花72小时去渲染，使用计算机集群才使得电影的制作时间缩短。由此可见在之前光线追踪就有应用，不过只能用于离线渲染领域，不可能用于游戏这种实时性强的领域。
![](https://johnyoung404.github.io/img/raytracing/transformer.png)

还有一个有些另类的光线追踪的例子，是《星际穿越》中黑洞的渲染，见下图。这个黑洞是根据物理学家们的建模，加上考虑了相对论的光线追踪计算出来的，被认为是第一次让我们“看见了”黑洞。这也说明了光线追踪本身就是依据现实中人眼成像的原理。
![](https://johnyoung404.github.io/img/raytracing/BlackHole.jpg)

### 2、光线追踪的原理是什么



### 6、延伸阅读及引用
https://blogs.msdn.microsoft.com/directx/2018/03/19/announcing-microsoft-directx-raytracing/
http://www.adriancourreges.com/blog/2015/11/02/gta-v-graphics-study/
https://devblogs.nvidia.com/introduction-nvidia-rtx-directx-ray-tracing/
https://www.ea.com/seed/news/seed-project-picapica
