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

``` C#
public static void Blit(Texture source, RenderTexture dest, Material mat, int pass = -1);
```

>“Blit sets dest as the render target, sets source _MainTex property on the material, and draws a full-screen quad .”
><p align="right">- Unity Document</p>



#### 常见用法

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

### 2. 边沿检测实现描边

![](https://johnyoung404.github.io/img/pbr/ting.png)

### 3. 降低色阶以模拟漫画的质感


### 6. 引用

* [描边效果的几种实现方式](https://gameinstitute.qq.com/community/detail/114228)
* [非真实感渲染](https://zhuanlan.zhihu.com/p/84075550)