---
title: Wwise-101
photos: images/Wwise101.png
date: 2024-06-11 23:32:27
tags: [学习,音乐]
categories: 游戏技术
---

# Part I

重点是介绍基础的怎么使用，有以下的重要结构   
> 所有的workUnit里面的，不同类型的音乐如场景，人声可能由不同的公司制作，所以有一个workUnit方便集合（本身是xml文件）

有三个重点部分
1. Audio目录存储所有音乐资源
2. Events事件管理音乐内部的逻辑（播放循环等，且事件底下统一按小写处理，不区分大小写）
3. SoundBank主要是用来与代码交互，代码要找到相应的SoundBank 才可以调用下面的事件
   
soundbank生成目录就是游戏需要调用的目录，event要扔在SoundBank里，Audio通过Event管理

![system](images/audio_event_sound.png "结构")  

# Part II

重点在于编辑器的使用 音高，音频编辑（裁剪，淡入淡出），随机类型的音乐（虚文件夹），序列类型的音乐（虚文件夹）等

>感觉用起来很舒服，起码比FLstudio要舒服的多，左下角有所有的功能提示，FLstudio只有简介，Wwise甚至有说明，各种编辑起来也很方便，不知道是不是因为本身属于一种编程思想的音频引擎？O_o

### 有一个值得注意的地方是，在编辑器里面Audio目录里面Copy是新建的引用，而不是新建文件夹，引用的是同一个文件，这方面Wwise自己也说很注重性能，前面第一部分开头就是性能分析怎么分析，感觉确实做的很好，易用且考虑的多

# Part III
主要是GameSync目录(游戏同步器的使用)
1. Switch(跟程序中的Switch是一样的,但选择的是不同的播放音频,要满足自己设置好的条件(比如引擎call草就播放草相关的音效))
2. GameParameter(游戏参数): 血条什么的,比如可以设置血条为0的时候播放哪些
3. RTPC曲线:对声音的调整曲线
4. State:就跟之前创建的其他 Game Sync 一样，我们需要将 State Group 的名称以及其中包含的 State 对象告知游戏引擎程序员，以确保游戏调用与这些对象完全一致。

最后还是要Event事件去控制游戏参数的逻辑(按照之前的Audio->Event->SoundBank->引擎)


