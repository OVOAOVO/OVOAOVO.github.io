---
title: 最终幻想16 shadow part I
photos: images/FFXVI.png
date: 2024-03-13 23:13:45
tags: [AAA,学习]
categories: 游戏技术
---
# FFXVI Shadow Part I
>PDF:http://www.jp.square-enix.com/tech/library/pdf/2023_FFXVIShadowTechPaper.pdf
## 前置补充
### Tile-Based Shading  
Tile-Based Shading(也被叫做屏幕空间分块着色)的主要思想是将屏幕空间划分为多个2D瓦片，然后对于每个2D瓦片，计算出和它相交的光源列表。在渲染时，对于每个2D瓦片，只使用与其相交的光源列表中的光源进行着色。

### TBDR(Tile-Based Deferred Rendering)
两个TBDR:  
一个是SIGGRAPH2010上提出的，通过分块降低宽带内存用量
一个是PowerVR基于手机GPU的TBR架构提出的通过HSR减少OVERDRAW

### Clustered Deferred Rendering
分簇延迟渲染-待补充

### Tile-Based Deferred Shading
Tile-Based Deferred Shading将传统的Deferred Shading的lighting阶段替换为light culling阶段来减少不必要的光源着色计算。目前有多个使用了这一思想的Tile-Based Deferred Shading实现。  
PlayStation 3 实现  
PS3的微处理器由1个3.2GHz的基于PowerPC的"Power Processing Unit(PPU)"和6个协处理器(SPU)构成。此外，PS3还包含了一个NVIDIA G70图形处理单元(GPU)。  
它的SPU被专门设计可以快速进行128位的SIMD向量运算操作。  
PS3的SPU和GPU可以并行执行运算。  
得益于PS3硬件的并行设计，Tile-Based Deferred Shading的实现最先出现在PS3硬件上。GDC2008“The technology of Uncharted: Drake Fortune”首次提到了这一实现，然而缺乏实现细节，根据他们的描述，算法大致过程如下：  
1. 渲染不透明的需要动态光源着色的对象的几何信息：世界坐标系下的法线信息+屏幕空间的镜面指数
2. 将屏幕划分为一个个单元格
3. 计算出和每个单元格相交的光源列表
4. 对每个单元格渲染可以覆盖这一单元格的四边形，在一个pass进行最多8个光源的着色计算，并将着色结果存储在一个累积缓冲中  
   
![TBDS](images/TBDS.png "TBDS")    

PS3的另一个Tile-Based Deferred Shading实现在2009年由PhyreEngine小组发表。他们的实现充分利用了SPU。根据他们的描述，算法大致过程如下：

1. 计算每个tile受到影响的光源列表  
使用每个tile的最小和最大深度值构造视锥体  
使用构造的视锥体对光源的包围体进行视锥体剔除  
使用tile的平均法线值计算光照着色  
2. 基于tile的内容选择渲染路径  
如果tile没有受到光源影响使用最快的渲染路径  
如果tile包含有需要光照着色的像素，使用光源材质信息进行光照着色  
3. 对每个tile选择是否进行MSAA处理
   
Tile-Based Deferred Shading的优点：

1. 常量大小的带宽需求。对于每个像素，只读取了一次G-buffer
2. 不需要使用累积缓冲
3. 可以处理包含大量重叠光源的情况
   
Tile-Based Deferred Shading的不足之处：

1. 需要支持DirectX 11以上的硬件(Compute Shader 5.0)
2. 和传统延迟着色相同的MSAA处理问题
3. 透明物体的处理问题

>基于【Siggraph 2019】A Scalable Real-Time Many-Shadowed-Light Rendering System 扩展
https://www.jianshu.com/p/3f76b8986564
https://zhuanlan.zhihu.com/p/370951892

## 正文 

### Part I ：Prepare
在此初始阶段，会在 CPU 上生成多个光源列表，以用作以下阶段的缓冲区输入，从而加速阴影处理。这是唯一未在 GPU 上处理的步骤。  

Zbin  
视锥体被划分为 1024 个均匀的深度范围，从而创建 zBin，用于存储与其相交的第一个和最后一个光指数

ZBin (ranged)  
包含与特定范围的连续 zBins 相交的第一个和最后一个光索引
![FFXVI](images/FFXVI_Zbin.png "FFXVI")    

zBin Buffer
存储 zBin 和 zBin 的 GPU 缓冲区（范围）

Tile Light Mask Buffer  
此缓冲区用于快速丢弃图块屏幕区域之外的灯光。为了减少每个 Tile 支持 1024 盏灯的内存要求，同时保持对结果的可预测直接访问，使用 32 字节位域和额外的 4 字节位域来跳过一组连续 32 个无效灯

虽然 Tiled Deferred Shadow 使用的 Shadow Tile 大小设置为 8x8 像素，但我们通过使用更大的 32x32 像素 Tiles 来降低此缓冲区的 CPU 计算成本（参见图 5）。在 4K 分辨率下，这意味着仅处理 8,100 个 Tile，低于原来的 129,600 个，代价是下一阶段丢弃的误报略有增加

![FFXVI](images/FFXVI_LightMaskBuffer.png "FFXVI")    

形成 32 个连续光源组，并在 Tile Light Mask Buffer 的前 4 个字节中使用 1 位来指示它们何时至少有一个有效光源。我们使用它通过忽略空范围来加速灯光的迭代。  

![FFXVI](images/FFXVI_4X4ShadowMap.png "FFXVI")    


>参考引用：https://blog.csdn.net/ddgf01/article/details/118529852
https://zhuanlan.zhihu.com/p/85447953