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

## 前言

PBR管线已经是现代引擎的标配了，本文简单讨论一下PBR相关概念，以及Unity中的PBR

### 1. 什么是PBR

PBR即基于物理的渲染，Wikipedia上对于PBR的定义是这样的：
>“Physically based rendering (PBR) is an approach in computer graphics that seeks to render graphics in a way that more accurately models the flow of light in the real world. Many PBR pipelines have the accurate simulation of photorealism as their goal. Feasible and quick approximations of the bidirectional reflectance distribution function and rendering equation are of mathematical importance in this field. Photogrammetry(摄影测量法) may be used to help discover and encode accurate optical properties of materials. Shaders may be used to implement PBR principles.”
><p align="right">—— From Wikipedia</p>

其中可以得到几个信息：

* 按照真实世界中的光照建模
* 可以用双向反射分布函数（BRDF）描述
* 通常由shader实现

从这段描述可以看出来，PBR是一个很宽泛的概念，描述的不是某种具体的实现，而是一系列原理相似的方法。就像这句话说的：
>“more of a concept than a strict set of rules.“
><p align="right">—— Joe Wilson</p>

一般的，当一个渲染模型满足下面三个条件时，我们就可以称它是“基于物理的”，也就是PBR：

* 1、基于微表面理论
* 2、能量守恒
* 3、渲染方程（BRDF、BSSRDF等）满足物理定律（反射、折射定律）

### 2. 基于物理渲染的优势

先来看看几种经典的渲染模型：

* Lambert/Half Lambert（ambiant + diffucse）
* Phong/Blin-Phong（ambiant + diffucse + specular）

兰伯特，半兰伯特光照考虑了环境光和漫反射，phong模型或者blin-phong考虑了环境光、漫反射和高光。他们都是经验模型，也就是说：只要看起来是对的，就是对的（when it looks right, it’s right）。在计算机图形学中这是很正常的，渲染效果是唯一衡量标准。

但这些模型有一些问题。首先是这些参数的调整，需要美术根据视觉效果来调。这些参数往往没有特殊的物理意义，因此调整起来往往更多的是经验+运气。“越少人工就越少出错”，在软件工程中我们一般不喜欢依赖人工，因为它增加了不稳定性，增加了出错的机会。其次，在光照条件改变的时候，之前看起来正常的参数，很可能会变得奇怪。也就是说，渲染效果和光照条件是强耦合的。

而PBR在这些方面是有优势的。首先是参数，PBR工作流的参数往往具有物理意义，而这些参数又能通过之前维基百科里提到的Photogrammetry(摄影测量法)来获得。例如下图展示了常见材料的测量值：

![](https://johnyoung404.github.io/img/pbr/photogrammetry.png)

其次，由于PBR是基于物理的，所以即使是在不同的光照条件下，我们也能获得不错的效果。例如下图是Unity官方Demo里展示的不同光照下的PBR渲染效果：

![](https://johnyoung404.github.io/img/pbr/lighting.png)


### 3. Microfacet Theory(微表面理论)和Cook-Torrance BRDF

微表面理论、BRDF和能量守恒，是PBR需要满足的特征。能量守恒会在BRDF中体现，所以不作介绍。

#### 微表面理论

![](https://johnyoung404.github.io/img/pbr/microfacet.png)

微表面理论认为：不是完全光滑的平面，是有许许多多完全光滑的微平面组成的，每个微平面都会完美地反射光线。

对于各向同性材料（即从不同方向观察，具有相同的光学性质），可以认为这些微平面的偏向角度是对称的，即有一个朝北倾斜30°的，就有朝南倾斜30°的。我们可以通过描述不同倾斜程度平面的分布情况，来描述宏观表面的粗糙度。越粗糙，倾斜程度大的微平面越多，反射光被遮挡越多，越不容易进入视线，因此高光看起来越模糊，如下图：

![](https://johnyoung404.github.io/img/pbr/roughness.png)

可以看到微平面理论也是对真实世界的合理简化，这种简化可以保留一部分现实中的特性，同时做了一些不那么正确的假设，方便我们建立数学模型。

#### Cook-Torrance BRDF

Cook-Torrance BRDF的公式如下：

![](https://johnyoung404.github.io/img/pbr/BRDF.png)

这个公式的推导比较复杂，需要用到球面积分相关知识。（当然，实现的时候大家基本都是抄公式），因此只对物理意义作简要介绍。

#### D项

![](https://johnyoung404.github.io/img/pbr/D.png)

D项是镜面高光（Normal Distribution Function
），由光线、视线、法线、粗糙度m这几个参数决定。这一项类似于phong或者blin-phong等高光模型。

#### G项

![](https://johnyoung404.github.io/img/pbr/G.png)

G项是几何遮蔽（Geometrical Attenuation Factor
），也由光线、视线、法线、粗糙度m这几个参数决定。这一项是修正项，越粗糙受光线影响越大；但入射光越垂直受到几何遮挡的影响越小。这可以通过之前说过的微表面模型去理解。越粗糙，倾斜度高的微平面越多。

#### F项

![](https://johnyoung404.github.io/img/pbr/F.png)

F项是菲涅尔项（Fresnel Equations），由视线、法线、粗糙度m这几个参数决定，描述的是菲涅尔效应。当我们视线与法线越垂直时，菲涅尔效应越明显。例如观察水面的时候，远处是菲涅尔效应，是亮的一片；近处主要是折射，可以看到水底。

### 4. Unity PBR工作流

#### Unity中的两种PBR工作流：

![](https://johnyoung404.github.io/img/pbr/workflow.png)

* Standard -> Metallic workflow（金属工作流）
![](https://johnyoung404.github.io/img/pbr/metallic.png)
* Standard(Specular setup) -> Specular workflow（高光工作流）
![](https://johnyoung404.github.io/img/pbr/specular.png)

#### 两种工作流的区别

二者采用的纹理不同：
![](https://johnyoung404.github.io/img/pbr/diff.png)

金属工作流：
* Albedo（固有色） sRGB
* Metallic（金属度）grayscale/linear space
* Roughness（粗糙度） grayscale/linear space

高光工作流：
* Albedo（固有色） sRGB
* Specular(镜面反射图） sRGB
* Glossiness(光滑度）grayscale/linear space

共有：
* Height（高度图）linear space
* Normal（法线）linear space
* Ambient Occlusion（环境光遮蔽） sRGB
* Emissive Map （自发光）

可以看到高光工作流占用的内存多一些，因为它用了两张sRGB，一张灰度图；而金属工作流用一张sRGB,两张灰度图

#### 该选那种工作流

根据Unity官方文档，二者都可以得到理想的效果，请选择适合自己美术团队的：

![](https://johnyoung404.github.io/img/pbr/unityDiff.png)

两种工作流下的橡胶塑料：

![](https://johnyoung404.github.io/img/pbr/rubbery.png)

#### 线性空间

一般来说，PBR都需要在线性空间中进行才能达到理想的效果。所以要确保正确地使用了Unity的PBR工作流，需要了解线性空间。线性空间是什么呢？

![](https://johnyoung404.github.io/img/pbr/linearSpace.png)

我们的显示设备所在的空间是Gamma2.2空间（有时我们需要Gamma校正也是类似的原因）。当电压线性增长时（1倍），光强是以2.2的指数倍增长的。因此一般我们的图片是在Gamma0.45空间储存的，也就是sRGB，这样显示到显示器上，就是0.45 * 2.2 = 1，正好还原。

我们不能使用Gamma0.45空间中的图片去进行PBR渲染，因为现实世界中，光强是线性变化的，我们不能用非线性的图片。

在Unity中开启线性空间的渲染，只需要勾选：

![](https://johnyoung404.github.io/img/pbr/setting.png)

还需要注意的是图片导入设置的处理，对于sRGB图片（也就是一般的图片），我们需要勾选这一项：

![](https://johnyoung404.github.io/img/pbr/sRGB.png)

勾选这一项后，Unity的线性工作流会自动在图片输入之前进行“去伽马校正”的操作，也就是一个2.2次幂的操作，将sRGB图片转到线性空间。如果我们不勾这一项，就不会进行此操作。如果我们导出的金属度贴图、法线贴图等本身就是线性空间中的图片，那我们不需要勾选这一项。

假如我们没有勾选sRGB选项(上方)：我们得到了一个更亮的但是错误的结果。因为我们少了一次“去伽马校正”的操作，因此图片RGB值偏大，看起来也就更亮。

![](https://johnyoung404.github.io/img/pbr/left.png)
![](https://johnyoung404.github.io/img/pbr/right.png)

Substance软件也是在线性空间中处理的，可以看到Unity的还原效果相当好，这减少了美术和开发之间的冲突：

![](https://johnyoung404.github.io/img/pbr/substance.png)

### 5. Unity PBR实现

如果你想找Unity PBR的实现，可以先找这个文件：**UnityStandardCore.cginc**，一般放在\Editor\Data\CGIncludes
，也就是内置的Unity 5以后的standard shader：

![](https://johnyoung404.github.io/img/pbr/1.png)

可以看到，fragment的最后输出大概 = 带GI(Global Illumination)的BRDF + 自发光 + 雾。

我们看看UNITY_BRDF_PBS这个宏：

![](https://johnyoung404.github.io/img/pbr/2.png)

它其实针对不同的shader model进行了不同的实现，总共三级。

#### Level 1

![](https://johnyoung404.github.io/img/pbr/3.png)
Level 1是最完整的BRDF，从disney的PBR相关工作衍生而来。效果如下：
![](https://johnyoung404.github.io/img/pbr/level1.png)

#### Level 2

![](https://johnyoung404.github.io/img/pbr/4.png)
Level 2是精简版本的Cook-Torrance BRDF，对某些项进行了近似。效果如下（和1对比没有明显差别）：
![](https://johnyoung404.github.io/img/pbr/level2.png)

#### Level 3

![](https://johnyoung404.github.io/img/pbr/5.png)
Level 3是主要从性能考虑的BRDF，F项被省去，可以认为是fallback版本。效果如下（和1、2对比，没有了菲涅尔效应）：
![](https://johnyoung404.github.io/img/pbr/level3.png)

### 6. 引用

[https://www.cnblogs.com/wbaoqing/p/8931646.html](https://www.cnblogs.com/wbaoqing/p/8931646.html)
[https://blog.csdn.net/yangxuan0261/article/details/88749969](https://blog.csdn.net/yangxuan0261/article/details/88749969)
[https://zhuanlan.zhihu.com/p/33464301](https://zhuanlan.zhihu.com/p/33464301)
[https://en.wikipedia.org/wiki/Physically_based_rendering](https://en.wikipedia.org/wiki/Physically_based_rendering)
[https://zhuanlan.zhihu.com/p/21376124](https://zhuanlan.zhihu.com/p/21376124)
[https://docs.unity3d.com/Manual/StandardShaderMetallicVsSpecular.html](https://docs.unity3d.com/Manual/StandardShaderMetallicVsSpecular.html)
[https://zhuanlan.zhihu.com/p/66558476/](https://zhuanlan.zhihu.com/p/66558476/)
