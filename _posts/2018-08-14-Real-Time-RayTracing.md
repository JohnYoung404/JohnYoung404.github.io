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

例如下图的这部3D电影《变形金刚》就运用了光线追踪来渲染，在我小时候给我带来了很大震撼。其中有些frame据说要花72小时去渲染，使用计算机集群才使得电影的制作时间缩短。由此可见在之前光线追踪就有应用，不过只能用于离线渲染领域，不可能用于游戏这种实时性强的领域。

![](https://johnyoung404.github.io/img/raytracing/transformer.png)

还有一个有些另类的光线追踪的例子，是《星际穿越》中黑洞的渲染，见下图。这个黑洞是根据物理学家们的建模，加上考虑了相对论的光线追踪计算出来的，被认为是第一次让我们“看见了”黑洞。这也说明了光线追踪本身就是依据现实中人眼成像的原理。
![](https://johnyoung404.github.io/img/raytracing/blackHole.jpg)

### 2、光线追踪的原理是什么

下面这张图描述了光线追踪是怎么实现的：

![](https://johnyoung404.github.io/img/raytracing/principle.png)

从图中可以看到，我们从视点向屏幕任意方向发射一条射线（视点相当于视网膜，屏幕相当于凸透镜）。射线与物体透明物体O1相交于P1，在P1处既会发生反射又会发生折射（反射和折射遵从反射定律和折射定律，以及能量守恒）。这些反射光线和折射光线又会发生反射、折射。每次反射和折射后，我们都假定有一定的能量损失，因此衍生次数越多的光线，我们认为贡献度越低。通常的场景下，我们认为五次bounce之后的光线贡献度很小，可以不用继续追踪了。对于每条追踪下去的光线，如果它和自发光物体碰撞了，那它就对屏幕上对应的那一个像素的颜色有贡献；如果超过了反射次数还没碰到自发光物体，我们就认为它对屏幕的这个像素没有贡献。

可以看出，我们是用反向的方式去追踪光线的，也就是从相机发射光线，去寻找光源，即下图的右边。然而为什么我们不用更直观的“Forward”的追踪方式：从光源向各个方向发出光线，然后和相机相交。这在原理上也是可行的，因为光路是可逆的，正向反向都是可以的，但是在计算代价上，Backward的方式要远远好于Forward，因为后者有大量无用的计算。

![](https://johnyoung404.github.io/img/raytracing/principle2.JPG)

### 3、如何实现一个蒙特卡洛path tracer

对于每个像素，我们随机向各个方向发射N条射线，这N条射线跟踪到的亮度之和为I，那么这一个像素的颜色可以近似表示为：I / N。
这其中就包含了**蒙特卡洛方法**(Monte Carlo method)。用比较通俗的话来说就是：用频率来估计概率。例如π的计算：

![](https://johnyoung404.github.io/img/raytracing/MonteCarlo.jpg)

光线追踪是一种类型的渲染方法，是一个比较大的概念，而使用蒙特卡洛方法是其中一个具体实现，一般被称为path tracer(路径追踪)。即使是对于比较简单的场景，这种方法也需要大量的光线才能达到比较好的效果，否则噪点会比较多。每个像素射出的光线数目称为spp(samples per pixel)，对于一般场景，1000 spp可以达到不错的效果，10000 spp就基本没有噪点了。
路径追踪的代码可以很简洁，网上有一个100行的实现，非常好懂也很有简洁之美，非常推荐了解一下。唯一不足，是效率上还有很大的优化空间。

笔者在比较早之前也实现过一个path tracer，代码库在这儿：[path tracer](https://github.com/JohnYoung404/JLib/tree/master/JLib/jRayTracing)，做了一些效率上的改进：

* 运用并行技术：每一条光线都是相对独立的，可以用一个线程去计算；OpenMP是一个很好用的线程库，在VS中只需要简单设置就可以启用
* 减少求交运算：运用包围盒技术(BVH, Bounding Volume Hierarchy)构建索引，减少总的求交次数
* 一些其他的trick: 使用Russian Roulette(俄罗斯转盘)来计算衰减。并不每次计算百分比衰减，而是掷骰子，不通过则终止射线。在总体期望上，是一样的，但是可以减少射线条数。

效果如下，使用的是1000spp：

![](https://johnyoung404.github.io/img/raytracing/rayTracer.JPG)

### 4、光线追踪的State-of-the-art如何

上述讨论里，光线追踪是在CPU上跑的，速度太慢，不能用于实时游戏领域，要么是离线渲染，要么是理论研究。但是前段时间，游戏工业界关于GPU上的光线追踪的进展迅速：
* At GDC 2018, NVIDIA unveiled RTX, a high-performance implementation that will power all ray tracing APIs supported by NVIDIA on Volta and future GPUs.
* Microsoft announced the integration of ray tracing as a first-class citizen into their industry standard DirectX API on March 19, 2018.
* EA, Epic games, Futuremark and Unity are planning to integrate DXR support into their games and engines.

我们来看看DX12的光线追踪渲染管线：
![](https://johnyoung404.github.io/img/raytracing/DX12Pileline.png)
和传统的光栅化的管线相比，差异还是蛮大的。不过思路是类似的，都是由可编程的阶段（上图绿色）和不可编程阶段（求交、BVH遍历）构成的。

Epic Games的光线追踪Demo：

![](https://johnyoung404.github.io/img/raytracing/Unreal.JPG)

EA的光线追踪Demo：

![](https://johnyoung404.github.io/img/raytracing/EA_PICAPICA.JPG)

### 5、光线追踪可以从哪些方面帮助我们提高渲染品质

这一段话可以回答这个问题：
>“The beauty of ray tracing is that it preserves the 3D world and visual effects like shadows, reflections and indirect lighting are a natural consequence of the ray tracing algorithm, not special effects.”
><p align="right">—Microsoft announcing DXR</p>
那光线追踪很自然的就解决了之前局部照明中我们需要使用trick来解决的问题。接下来我们讨论几个典型的益处（在EA的project PICA PICA中也有讨论）：

#### AO(Ambient Occulusion)
AO（环境光遮蔽）是之前局部光照明模型中，我们模拟物体之间互相遮蔽产生阴影的现象：

![](https://johnyoung404.github.io/img/raytracing/AO.JPG)

虽然有很多技术，但一般我们是在屏幕空间去做，称为SSAO（Screen Space Ambient Occlusion），这个方法利用屏幕空间的深度缓存，计算像素周围和它的深度差，越深则遮罩效果越强，该像素的颜色越暗。这是一种近似的方法，可以达到一定的环境光遮蔽效果，但不是很完美，有时候甚至很错误。

利用光线追踪，我们可以得到更正确的AO ，对比如下：

![](https://johnyoung404.github.io/img/raytracing/RTAO.JPG)

#### DOF(Depth of field)

DOF（景深）是如何产生的？

![](https://johnyoung404.github.io/img/raytracing/DOF.JPG)

在真实世界中，我们的眼睛是一个凸透镜，也就是说，在焦平面上的物体会清晰地在我们的视网膜上成像；在焦平面以外，多个点会对应我们视网膜上的同一个点，因此这些点的颜色在这一点混合了，因此我们就觉得焦平面以外的物体会模糊。这就是景深产生的原因。
在传统的做法中，我们也是在屏幕空间利用深度缓存做景深效果的，其实就是根据深度差做高斯模糊。例如GTA5中的景深效果：

![](https://johnyoung404.github.io/img/raytracing/GTA5.JPG)

在光线追踪中，我们可以模拟凸透镜，达到自然又真实的景深效果：

![](https://johnyoung404.github.io/img/raytracing/DOF_PICAPICA.png)

#### SSS(Subsurface Scattering)

SSS是次表面散射，是由于光线进入半透明介质（如皮肤、玉石）时，光线在次表面多次反射，使材质的一定深度的地方也和光线有明显的交互。

这是GTA中的SSS：

![](https://johnyoung404.github.io/img/raytracing/SSS.JPG)

这是光线追踪的SSS：

![](https://johnyoung404.github.io/img/raytracing/RTSSS.png)

#### Shadows

光线照不到的地方，能量小，所以看起来就暗，这是自然世界中阴影产生的原因。光线追踪很自然的解决了阴影的问题，不管是软阴影还是硬阴影。

而在局部照明的渲染管线中，我们要运用各种trick去让阴影看起来真实一些。

![](https://johnyoung404.github.io/img/raytracing/Shadows.png)

### 6、延伸阅读及引用
[https://blogs.msdn.microsoft.com/directx/2018/03/19/announcing-microsoft-directx-raytracing/](https://blogs.msdn.microsoft.com/directx/2018/03/19/announcing-microsoft-directx-raytracing/)
[http://www.adriancourreges.com/blog/2015/11/02/gta-v-graphics-study/](http://www.adriancourreges.com/blog/2015/11/02/gta-v-graphics-study/)
[https://devblogs.nvidia.com/introduction-nvidia-rtx-directx-ray-tracing/](https://devblogs.nvidia.com/introduction-nvidia-rtx-directx-ray-tracing/)
[https://www.ea.com/seed/news/seed-project-picapica](https://www.ea.com/seed/news/seed-project-picapica)