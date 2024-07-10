---
title: 次序无关的半透明渲染与Unity渲染管线
photos: images/chess.png
date: 2023-12-27 22:59:45
tags: [学习,Unity]
categories: 游戏技术
---
# Unity中部分渲染管线概念
## Unity自定义管线
* SRP: 全称为Scriptable Render Pipeline（可编程渲染管线/脚本化渲染管线）它主要把渲染管线拆分成二层:一层是比较底层的渲染API层，像OpenGL，D3D等相关的都封装起来。另一层是渲染管线上层，上层代码使用C#来编写。在C#这层不需要关注底层在不同平台上渲染API的差别，也不需要关注具体如何做一个Draw Call
* URP: 全称为Universal Render Pipeline(通用渲染管线), 它是Unity官方基于SRP提供的模板，它的前身是LWRP(Lightweight RP即轻量级渲染管线), 在2019.3开始改名为URP
* HDRP: 全称为High Definition Render Pipeline（高清晰度渲染管线），它也是Unity官方基于SRP提供的模板，它更多是针对高端设备，如游戏主机和高端台式机，它更关注于真实感图形和渲染
  
Baches（批处理）其实就理解成DrawCall值就可以个Batch至少包含一个DrawCall  
原因 1：Unity引擎开启批处理情况下将把满足条件的多个对象的渲染组合到一个内存块中以便减少由于资源切换而导致的 CPU 开销，也就是把多个DrawCall合并成一个DrawCall，来减少调用DrawCall的开销（主要是调用DrawCall之前的一系列设置），这个操作就是批处理。  
原因 2：把数据加载到显存，设置渲染状态及数据，CPU调用DrawCall的这一整个过程就是一个Batch。这个过程当中主要性能消耗点在于上传物体数据（加载到显存）和设置渲染状态及数据这一块，而不是DrawCall命令本身。  
SetPass calls 官网解释：渲染 pass 的数量。每个 pass 都需要 Unity 运行时绑定一个新的着色器。
>内置渲染管线：是所有材质球的渲染pass的数量。这里可能会有疑惑，拿Unity内置的 Lit Shader来举例，这Shader里有好多个Pass通道，SetPass calls值却跟Pass通道数量始终对不上，个人猜测这是因为Unity渲染时会选择Pass通道渲染，比如关闭阴影，那么Unity就不会选择阴影渲染通道也就是 ShadowCaster通道。  
URP：渲染不同pass的数量，跟内置渲染管线不一样的是在URP不局限于材质球，也就是假如有五个材质球，但Shader和关键字都一样，这个Shader有两个Pass通道，那么这些物体的SetPass Calls值就是2。  
其实想想也是合理的，SetPass calls值个人猜测官网主要是想呈现跟批处理相关的值，在内置渲染管线只能批处理使用同样材质球的物体，所以这个值为所有材质球的渲染Pass通道数量，在URP开启SRP Batcher情况下呢，是根据相同Shader变体的pass进行合批。 --知乎龙笑

ShaderVariant（shader变体）  
在写shader时，往往会在shader中定义多个宏，并在shader代码中控制开启宏或关闭宏时物体的渲染过程。最终编译的时候也是根据这些不同的宏来编译生成多种组合形式的shader源码。其中每一种组合就是这个shader的一个变体(Variant)。

# 顺序无关的半透明混合（OIT问题）

## 深度剥离 Depth Peeling

基本思想是利用 N 个 Pass 分别渲染出距离摄像机第 N 近的片元，这即是深度剥离的过程，最后再从远到近依次将剥离出的 N 个图层根据透明度混合叠加到屏幕缓冲中。

实现深度剥离中的一个 Pass 需要2个深度缓冲和1个颜色缓冲，其中一个深度缓冲储存上一层剥离的深度值，另一个储存当前剥离层的深度值，颜色缓冲储存当前剥离层的片元色彩。

## 双深度剥离

Nvidia 提出了一个可以在一次 Pass 中剥离两层的算法，即每一次剥离出距离摄像机第 i 近和 第 i 远的两层，这样可以用 (N/2+1) 个 Pass 实现 N 层深度剥离的效果

## 延迟表面渲染

可以使用类似 Deferred 的方式对半透明表面进行复杂着色， 即在每次剥离时除了深度值以外，还需要储存当前层的法线、颜色等数据，在最终的混合 Pass 中再进行半透明表面的复杂着色渲染。而这样我们需要类似 N 个 GBuffer 的空间，以空间换时间的目的。

## 逐像素的链表（Per-Pixel Linked Lists）

逐像素的链表，是运用用到一个被成为“GPU 并行链表“的技术，在 Pixel Shader 中使用两个可写的纹理（DX11，SM 5.0，UAV），一个屏幕大小的链表头纹理，一个屏幕大小 N 倍的链表节点纹理。链表头纹理的每个像素存储每个像素链表在节点纹理中的偏移量。

这项技术可以用来解决图形渲染中的很多问题，用来解决 OIT 问题只是其诸多应用之一。核心做法是遇到半透明片元就将深度、颜色等写入对应像素的链表。最后再用一个 PASS 对每个像素的链表进行深度排序和颜色混合。

## Adaptive Transparency
Adaptive Transparency，简称 AT，是 Intel 提出的一种 OIT 技术。其核心思想是通过修改经典混合公式本身来改进 Per-Pixel Linked Lists 方法的内存和性能问题。  
![images1](images/blendfor.png "公式")  
在经典的混合公式中，颜色一次一次有序的迭代混合到目标色上的。如果我们使用某种方法，打破这种有序迭代，我们就可以不用再对片段排序了，即做到了真正的顺序无关。在这个方法中引入了一个能见度函数来简化这个顺序无关的混合公式：  
![images2](images/blendfor2.png "公式")  


>参考引用：https://zhuanlan.zhihu.com/p/353687806  
https://zhuanlan.zhihu.com/p/92337395  
https://zhuanlan.zhihu.com/p/353940259