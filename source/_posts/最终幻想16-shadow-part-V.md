---
title: 最终幻想16 shadow part V
photos: images/FFXVI.png
date: 2024-03-21 20:20:57
tags: [AAA,学习]
categories: 游戏技术
---
# FFXVI Shadow Part V

## 正文 

### Part V ：HiQuality

最后的第 5 阶段处理重要角色的更高质量的阴影,绘制在专用的Closeup Shadowmap中,在现有的 VisibilityBuffer 上合成角色阴影

Closeup Shadowmap:
与我们的高质量阴影技术一起使用。与普通阴影贴图一样运行，但视图仅限于一个感兴趣的对象，而不是整个光源的视野，以获得更好的分辨率精度。

Output: Updated Final Light List Buffer  
当阴影图块从完全亮起变为部分亮起时，需要分配一个Visibility Slot并保存Visibility SlotID。相反，需要通过将 Visibility SlotID 替换为完全点亮的标志。这两种情况都通过更新最终光源列表缓冲区中的阴影图块条目来处理。

Output: Updated VisibilityBuffer  
使用特写阴影贴图生成阴影。当灯光/像素可见性降低时，其在 VisibilityBuffer 中的值将更新。

 Output: Updated Standard Shadowmap
更新了标准阴影贴图 我们跳过了标准阴影贴图，只在Closeup Shadowmap中渲染了具有高质量阴影的角色，从而避免了增加角色的绘制次数。但是，前向渲染照明对象不能依赖于延迟渲染照明，因此标准阴影贴图需要将每个Closeup Shadowmap合成到其内容上。这使得角色阴影可供前向渲染器使用，但没有更高质量的优势。

基于我们的 Tiled Deferred Shadow 实现，我们有一个高质量阴影系统，使用单个重复使用的特写阴影贴图增量处理每个角色/光影。此阶段包含以下描述的 5 个子步骤。

![prepare](images/5prepare.png "prepare")    

Step: Prepare:   
计算了一个新的摄像机视图，其视锥体受角色边界和光线边界之间的最小视场的约束。由于我们使用的是单独的阴影贴图，因此其分辨率可以高于灯光的标准阴影贴图，从而通过进一步降低每个像素的立体角来提高质量  
![prepare](images/two_close.png "prepare")  
通过为光/仙人掌对分配高质量阴影，我们改进了阴影贴图分辨率的使用，视野更窄，因此像素处理的立体角更少（参见像素覆盖的红色区域）。通过将近平面和远平面移近拍摄对象，深度精度也会提高。

Step: Closeup Shadowmap Draws:  
通过将角色网格的绘制调用从光源的标准阴影贴图重定向到高质量阴影的Closeup Shadowmap，我们避免了对每个光源多次绘制角色网格。执行此操作的方式因每个渲染引擎的实现而异。

 Step: Closeup Shadowmap Resolve:  
 完成特Closeup Shadowmap绘制后，我们解析相关的分层深度缓冲区并生成阴影贴图的mipmapped版本，在每个 2x2 像素处保持与光线的最近距离。

Step: Shadows Composition:  
在 CPU 上，我们通过将光线从光源延伸到每个边界的位置，直到达到光半径来构建角色的本影体积。在屏幕坐标中投影后，我们发现启动计算工作单元的最小和最大位置仅限于受影响的阴影图块  
在computer shader中，每个工作单元在其最终光照列表缓冲区条目中查找关联的light index。当没有被发现或标记为完全处于阴影中时，我们会在这里停止工作。否则，我们会评估每个像素的光可见性，如果整个 Shadow Tile 完全点亮，我们就到此为止。
![prepare](images/charactor_bounding.png "prepare")  
computer shader 伪代码详见原文

Step: Standard Shadowmap Transfer
每个高质量阴影都是单独处理的，重用共享的Closeup Shadowmap渲染目标。这样做的好处是计算成本是唯一的限制因素，因为内存开销是固定的。但是，阴影贴图的内容会在计算每个角色/光源对之后被丢弃，因此前向渲染器无法使用它。由于其中不会绘制具有高质量阴影的角色，因此我们将每个特写阴影贴图结果传输回关联的标准阴影贴图。  
通过将Closeup Shadowmap的近平面四边形投影到标准阴影贴图坐标中来实现。一个标准阴影贴图像素包含许多Closeup Shadowmap像素，读取每个像素的成本高得令人望而却步。这个高带宽问题通过使用特写阴影贴图的 mipmapped 版本来解决，每个像素选择需要两个样本来覆盖该区域最小 XY 维度的 mipmap 级别。然后，我们迭代读取 mipmapped Closeup Shadowmap 值，直到覆盖整个区域。例如，在特写阴影贴图中投影到 15 x 37 像素区域的像素，我们选择 mipmap 级别 3。  
![prepare](images/mipmap_close.png "prepare")  

### Result:  
![prepare](images/hiquality_charactor.png "prepare")  
没有 （a） 和有 （b） 高质量阴影的角色阴影的结果。

 >PDF:http://www.jp.square-enix.com/tech/library/pdf/2023_FFXVIShadowTechPaper.pdf