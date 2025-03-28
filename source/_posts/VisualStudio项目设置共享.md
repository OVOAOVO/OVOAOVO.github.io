---
title: VisualStudio项目设置共享
photos: images/VSE.png
date: 2025-03-28 21:00:18
tags: [学习]
categories: 游戏技术
---

经过很长时间的去世(啊不是)，我又滚回来啦( •̀ ω •́ )y

# 项目设置

*那么众所周知，搭建某项目，必须要进行环境配置  
那么配置了一次后，如果再创建一个这样的项目我们要再配一次吗？  
是的再配一次(这篇结束)

开个玩笑哈。。。

VisualStudio目前其实自己有一个Props可以进行共享  
* View->OtherWindow->PropertiesManager  
  
通过创建新项目属性表，可以共享在其他项目的设置

![system](images/属性管理器.png "属性管理器")  

>起码官方是这么推荐的：https://learn.microsoft.com/zh-cn/cpp/build/create-reusable-property-configurations?view=msvc-170

但它并没有想象中那么好用，因为他是详细划分Debug/Release下的每个系统每一个会有一个Props

## .vcxproj

那么我们直接上答案

>https://learn.microsoft.com/zh-cn/cpp/build/project-property-inheritance?view=msvc-170

VisualStudio导入现有项目实际是导入的.vcxproj，所以我们只需要新建个文件夹，里面单独放入，你复制的当前设置好的.vcxproj

改个名，然后改下 RootNamespace 标签，再在解决方案下导入，让VisualStudio自己添加新项目就好了
