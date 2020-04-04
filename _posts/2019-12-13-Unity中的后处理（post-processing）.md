---
layout:     post
title:      Unity中的后处理（post-processing）
subtitle:   简单介绍一下Unity中的后处理（post-processing）
date:       2019-12-13
author:     John Young
header-img: img/tag-bg-o.jpg
catalog: true
tags:
    - Unity
    - Post-processing
    - Computer Graphics
---

## 前言

后处理是一种常见的技术。本文主要介绍一下在Unity中后处理一般是如何使用的，以及常见的后处理效果。

### 1. Unity中的后处理

在Unity中，我们通过Monobehavior.OnRenderImage、Graphics.Blit这两个内置接口，再加上想应用的后处理效果的shader，就能实现后处理效果。

#### Monobehavior.OnRenderImage

>“This message is sent to all scripts attached to the camera.”
><p align="right">- Unity Document</p>

根据Unity官方文档的说法，在所有渲染完成之后，最终的图像绘制之前，相机上所有脚本的这个函数（如果有）会被调用。
``` C#
void OnRenderImage(RenderTexture src, RenderTexture dest) 
```
其中src是后处理之前的RenderTexture，dest是处理之后的RenderTexture。

#### Graphics.Blit

>“Blit sets dest as the render target, sets source _MainTex property on the material, and draws a full-screen quad .”
><p align="right">- Unity Document</p>

Blit做的事情简单来说，就是将输入的纹理作为_MainTex传给某个材质，再将这个材质用于RenderTexture。

``` C#
public static void Blit(Texture source, RenderTexture dest, Material mat, int pass = -1);
```

多个Blit可以串联使用，将第一次的输出作为下一次的输入。不过从效率上来讲，不推荐这么做，多个后处理效果尽量合在一个shader里做是最好的。

#### 常见用法

一段很简单的使用后处理的示例代码：

``` C#
using UnityEngine;
public class ExampleClass : MonoBehaviour 
{ 
    public Material mat;
    void OnRenderImage(RenderTexture src, RenderTexture dest) 
    { 	// Copy the source Render Texture to the destination, 
    // applying the material along the way. 
    Graphics.Blit(src, dest, mat); 
    } 
} 
```

只需要把它挂在需要后处理效果的相机，然后给material赋上相应的后处理shader即可。后处理shader一般不需要背面剔除、深度检查和深度写入： Cull Off 、ZTest Always、 ZWrite Off

### 2. 常见后处理效果

后处理的表达能力是很强的，除了纹理，还有各种buffer可以使用。常见的后处理效果及实现：

* **屏幕扭曲**（使用噪声图生成扰动的uv，使fragment shader中的uv取值有一定扰动，从而达成扭曲的效果）

* **运动模糊**（累积缓存、速度缓存）

* **景深**（运用深度缓存+高斯模糊）

* **Bloom**（根据一个阈值提取出图像中的较亮区域，把它们存储在一张渲染纹理中，再利用高斯模糊对这张渲染纹理进行模糊处理，模拟光线的扩散，最后再将其与原图像进行混合，得到最终的效果）

* **屏幕空间的抗锯齿（Anti-aliasing）**、**屏幕空间反射（Screen Space Reflections）**、**屏幕空间环境光遮蔽（Screen Space Ambient Occlusion）**

* 其他图像处理能做的事，调个**饱和度**、**亮度**、**伽马矫正**、**直方图均衡化**……

### 3. 边沿检测实现描边

对于很多艺术效果，例如漫画、水墨、粉笔画等等，风格各异的描边对于实现效果都比较重要。

《Real-Time Rendering》中将描边技术分为五大类，几种技术的效果各不相同：

* **基于法线和视角的描边**( Shading Normal Contour Edges)		
（计算NdotV，越小越接近边沿）

* **过程式的几何描边**( Procedural Geometry Silhouetting)	
（又称shell method、halo method，做法是使模型朝着法线方向放大一些，着上背景色，然后再画模型，就能形成描边）

* **基于图片处理的描边**( Edge Detection by Image Processing)
（下文详细讲）

* **基于轮廓线检测的描边**( Geometric Contour Edge Detection)	
（ silhouette edges： (n0⋅v>0)!=(n1⋅v>0)，遍历模型，如果两个相邻的面一个面向观察者一个背向观察者，那么它们公有的边就是边缘）

* **混合以上几种描边方法**(Hybrid Silhouetting)

这里介绍一种，使用后处理的描边方法，主要原理是图像处理领域里的边沿检测。

#### 边沿检测的原理

* **卷积**：卷积操作指的是使用一个卷积核，对一张图像中的每个像素进行一系列操作。卷积核通常是一个四方形网格结构，每个方格都有一个权重，进行卷积时，我们将卷积核的中心放在目标像素上，翻转核之后再依次计算核中每个元素和其覆盖的图像像素值的乘积并求和，得到的结果就是该位置的新像素值。
* **梯度**：边的形成的核心性质就是在边的两侧其差值较大，这种差值的绝对值叫做梯度。基于这个内容，我们使用几种不同的边缘检测算子来计算梯度。
    - Roberts
    - Prewitt
    - Sobel
* 每个算子都包含x,y两个方向上的卷积核，每次我们需要对一个像素分别计算两个方向上的梯度，再求出总梯度G。
* 出于性能考虑一般使用G=Gx+Gy来代替平方根。

#### 一种基于roberts算子的shader实现

roberts算子如下：

![](https://johnyoung404.github.io/img/post-processing/roberts.png)

在fragment shader中我们实现描边：

``` glsl
float _EdgeWidth = 2.0f;
float2 texel = _MainTex_TexelSize.xy;
fixed3 col0 = tex2D(_MainTex, Input.texcoord.xy + _EdgeWidth * texel*float2(1, 1)).xyz; 
fixed3 col1 = tex2D(_MainTex, Input.texcoord.xy + _EdgeWidth * texel*float2(1, -1)).xyz; 
fixed3 col2 = tex2D(_MainTex, Input.texcoord.xy + _EdgeWidth * texel*float2(-1, 1)).xyz; 
fixed3 col3 = tex2D(_MainTex, Input.texcoord.xy + _EdgeWidth * texel*float2(-1, -1)).xyz;//rgb2gray将rgb转换为灰度值，0.2125 * r + 0.7154 * g + 0.0721 * b
float edge = rgb2gray(pow(col0 - col3, 2) + pow(col1 - col2, 2));
edge = pow(edge, 0.2f); //edge介于0~1之间，将它放大一些
fixed3 _EdgeColor = fixed3(1.0f, 1.0f, 1.0f); //描边颜色
return fixed4((edge)*_EdgeColor.xyz, 1.0);
```

基本原理就是使用Roberts算子计算梯度，确定pixel是否是边缘，效果如下：

![](https://johnyoung404.github.io/img/post-processing/edge.png)

为了描边明显一些，给edge增加一个阈值：

``` glsl
float _Sensitive = 0.35f; 
if (edge < _Sensitive) edge = 0;
```

优化之后的效果：

![](https://johnyoung404.github.io/img/post-processing/threshold.png)

### 4. 引用

* [描边效果的几种实现方式](https://gameinstitute.qq.com/community/detail/114228)
* [非真实感渲染](https://zhuanlan.zhihu.com/p/84075550)