---
title: 最终幻想16 shadow part III
photos: images/FFXVI.png
date: 2024-03-19 23:42:29
tags: [AAA,学习]
categories: 游戏技术
---
# FFXVI Shadow Part III

## 正文 

### Part III ：Find Lights
此第 3 阶段过滤来自 Tile Group Light Mask Buffer 的光index，对每个 Shadow Tile 进行更准确的每pixels测试。由于之前的结果由多个阴影图块共享，并且 Froxel 测试会产生误报，因此我们在下一步计算阴影之前尽可能多地消除阴影  
当每个计算线程处理 1 个pixels时，我们读取场景深度并迭代 Tile Group Light Mask Buffer 条目中的 Tile Group 灯光。shadow tiles考虑当光源在其至少一个pixels的范围内时，该光源是有效的。

![Froxel](images/Froxel.png "Froxel")    
在阴影图块中发现的光源近似值的精度误差。此处显示了两个 Froxel，由它们的 Shadow Tile 绑定加上最小和最大像素深度分隔。在Shadow Tile 1中，使用精确测试时，只有灯光A和D才能找到。通过更快的近似值，灯光 B 和 C 也会被错误地检测到（对这个阴影图块的像素没有影响）。Light B，因为它位于 Shadow Tile Froxel 内部。和光 C，因为它在邻居中有效，并且结果在 8x8 Tiles 之间共享。
>注：这张图主要说明查找光源时的误差，但我并没理清误差到底如何产生的,原文提到  
Few Shadow Tiles contain a high number of lights and allocating the same fixed amount of memory for all to handle the maximum count is wasteful. Instead, a flexible memory approach is used where each Shadow Tile reserves enough memory in a shared buffer to handles their approximated light count.   
大意为有一个共享的缓冲区存储所有的光index？加上part2中的算法为^=去除已经查过的光源，所以这里我猜测当光源B是因为在内部（范围大,一次查到了ABD）所以没有除去，而C是因为Froxel比较短里面并没有光源所以^=也没有除去，如果你看到后不对，请在下面评论指出谢谢 :）

![list](images/LightListBuffer.png "list")   

这两个为在每个shadowTile中查到的光源信息  
左图为Early Light List Buffer 每4byte存储3个index用来压缩（4个2^8次方->3个2^10次方）  
右边图为Final Light List Buffer 存储当前tile 的光index与可见

每个阴影图块生成的两种类型的光源列表的条目格式。每个列表的内存由“查找光源近似”阶段使用“图块近似光计数”分配，但存储的实际数量可能会更少，因为输出它们的相位更准确

### ’Find Lights Accurate’ compute shader pseudo code
```C++
 ByteAddressBuffer tTileShadowSlotIds
 ByteAddressBuffer tTileGroupLightMask ;
 RWByteAddressBuffer rw_TileLightListEarly ;
 RWByteAddressBuffer rw_TileLightCountEarly ;

 // Work Unit outputs 1 Tile (8 x8 pixels ), each thread evaluates 1 pixel
 [ numthreads ( SHADOWTILE_SIZE , SHADOWTILE_SIZE , 1) ]
 void CS_Phase3_FindLightsAccurate ( uint inThreadIdx : SV_GroupIndex ,
 uint2 inTileCoord : SV_GroupID ,
 uint2 inPixelCoord : SV_DispatchThreadID )
 {
 const uint tileIdx = GetTileIndex ( inTileCoord ) ;
 const uint shadowSlotID = tTileShadowSlotIds . Load ( tileIdx *4) ;
 const float depth = GetDepth ( inPixelCoord ) ;
 const float3 worldPos = GetWorldPosFromScreen ( inPixelCoord , depth ) ;
 const bool isValidPixel = ! IsFar ( depth ) &&
 all( inPixelCoord < cCommon . m_Resolution ) ;
 uint lightMaskAdr = GetLightMaskGroupAdress ( tileIdx ) ;
 uint lightGrpMask = tTileGroupLightMask . Load ( lightMaskAdr ) ;
 uint lightCount = 0; // Number of valid light found in Tile
 uint lightIdxPacked = 0; // Pack multiple light indices per uint
 // - ---- ---- ---- ---- ---- ---- ---- --- ---- ---- ---- ---- ---- ---- ---- --- --- ---
 // Process each valid group of 32 lights
 while ( lightGrpMask != 0) {
 uint lightGrpBit = firstbitlow ( lightGrpMask ) ;
 lightMaskAdr = GetLightMaskEntryAdress ( tileIdx , lightGrpBit ) ;
 uint lightMask = tTileGroupLightMask . Load ( lightMaskAdr ) ;
 lightGrpMask ^= (1 < < firstGroupBit ) ; // Remove processed group
 // -- ---- ---- ---- ----- ---- ---- ---- ----- ---- ---- ---- ----- ---- ---- --- ---
 // Process each valid light in current group
 while ( lightMask != 0) {
 uint lightBit = firstbitlow ( lightMask ) ;
 uint lightIdx = ( lightGrpBit * 32) + lightBit ;
 LightInfo lightInfo = GetLightInfo ( lightIdx ) ;
 lightMask ^= 1u << lightBit ; // Remove processed light
 // Check if pixel is in range of the light
 bool bValidLight = isValidPixel &&
 LightPixelTests ( lightInfo , worldPos ) ;
 // Add light indice to our list
 if( WaveActiveAnyTrue ( bValidLight ) && WaveIsFirstLane () ) {
 lightIdxPacked = lightIdxPacked | ( lightIdx << 10*( lightCount %3) ) ;
 lightCount ++;
 if( ( lightCount % 3) == 0 ) {
 uint outputAdr = GetListAddress ( shadowSlotID , lightCount ) ;
 rw_TileLightListEarly . Store ( outputAdr , lightIdxPacked ) ;
 lightIdxPacked = 0;
 }
 }
 }
 }
 // - ---- ---- ---- ---- ---- ---- ---- --- ---- ---- ---- ---- ---- ---- ---- --- --- ---
 // Store Tile ’s Light Count in early Light List
 if( WaveIsFirstLane () ) {
 rw_TileLightCountEarly . Store ( tileIdx *4 , lightCount ) ;
 if (( lightCount % 3) != 0) { // output pending packed indices
 uint outputAdr = GetListAddress ( shadowSlotID , lightCount ) ;
 rw_TileLightListEarly . Store ( outputAdr , lightIdxPacked ) ;
 }
 }
 }
```

>PDF:http://www.jp.square-enix.com/tech/library/pdf/2023_FFXVIShadowTechPaper.pdf