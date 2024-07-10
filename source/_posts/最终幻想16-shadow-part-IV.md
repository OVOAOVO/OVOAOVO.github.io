---
title: 最终幻想16 shadow part IV
photos: images/FFXVI.png
date: 2024-03-21 20:20:39
tags: [AAA,学习]
categories: 游戏技术
---
# FFXVI Shadow Part IV

## 正文 

### Part IV ：Generate
第 4 阶段使用从原始延迟照明着色器导入的代码计算每个光源的每像素可见性。使用Early Light List buffer中 Shadow Tile 条目中的光源列表，我们迭代它们以使用 Percentage Closer Soft Shadows 生成阴影，但可以用任何其他技术替换它。

 Output: VisibilityBuffer

 此缓冲区包含 8x8 内存块，用于存储shadow tile的每像素光可见性。这是延迟阴影系统所需的最大中间缓冲区，一旦照明阶段完成，就可以在帧的其余部分丢弃（其他缓冲区也是如此）。每个像素使用 8 位来表示光可见度的百分比，但 4 位可能就足够了，具体取决于项目的需要。

 ### ’Generate Deferred Shadows’ compute shader pseudo code
 ```C++
 ByteAddressBuffer tTileShadowSlotIds
 ByteAddressBuffer tTileLightListEarly ;
 ByteAddressBuffer tTileLightCountEarly ;
 RWByteAddressBuffer rw_TileLightListFinal ;
 RWByteAddressBuffer rw_TileLightCountFinal ;
 RWByteAddressBuffer rw_TileLightVisibility ;

 // Work Unit outputs 1 Tile (8 x8 pixels ), each thread evaluates 1 pixel
 [ numthreads ( SHADOWTILE_SIZE , SHADOWTILE_SIZE , 1) ]
 void CS_Phase4_GenerateShadows ( uint inThreadIdx : SV_GroupIndex ,
 uint2 inTileCoord : SV_GroupID ,
 uint2 inPixelCoord : SV_DispatchThreadID )
 {
 const uint tileIdx = GetTileIndex ( inTileCoord ) ;
 const uint shadowSlotID = tTileShadowSlotIds . Load ( tileIdx *4) ;
 const uint earlyLightCount = tTileLightCountEarly . Load ( tileIdx *4) ;
 const float depth = GetDepth ( inPixelCoord ) ;
 const float3 worldPos = GetWorldPosFromScreen ( inPixelCoord , depth ) ;
 const bool isValid = ! IsFar ( depth ) &&
 all( inPixelCoord < cCommon . m_Resolution ) ;
 // - ---- ---- ---- ---- ---- ---- ---- --- ---- ---- ---- ---- ---- ---- ---- --- --- ---
 // Iterate every light previously detected (in Early List light )
 uint finalLightCount = 0;
 uint entryIdx = 0;
 while ( entryIdx < earlyLightCount ) {
 uint lightIdx = GetNextEarlyLight ( shadowSlotID , entryIdx ) ;
 // Standard Shadow generation per pixel happen here
 float viz = ComputeLightVisibility ( lightIdx , worldPos ) ;
 bool isAllLit = WaveActiveAllTrue (viz > 0.9999 || ! isValid ) ;
 bool isAllShadow = WaveActiveAllTrue (viz < 0.0001 || ! isValid ) ;
 bool hasPenumbra = ! isAllLit && ! isAllShadow ;
 // -- ---- ---- ---- ----- ---- ---- ---- ----- ---- ---- ---- ----- ---- ---- -- ---
 // Only write out light if not entirely in shadow
 if( ! isAllShadow ) {
 uint vizSlotID = SLOTID_ALL_LIT ;
 // Per Pixel visibility generation when partial shadow detected
 if( hasPenumbra ) {
 vizSlotID = AllocateVisibilitySlotID () ; // 1 Slot / tile
 StoreVisibility ( vizSlotID , inThreadIdx , visibility ) ;
 }
 // Output packed light index and VisibilitySlotID / Flag
 uint packedValue = PackLightIndexFinal ( lightIdx , vizSlotID ) ;
 uint outputAdr = GetListFinalAddress ( shadowSlotID , finalLightCount ) ;
 rw_TileLightListFinal . Store ( outputAdr , packedValue ) ;
 finalLightCount ++;
 }
 entryIdx ++;
 }
 // - ---- ---- ---- ---- ---- ---- ---- --- ---- ---- ---- ---- ---- ---- ---- -- --- ---
 // Store Tile ’s Light Count in Final Light List
 if( WaveIsFirstLane () ) {
 rw_TileLightCountFinal . Store ( tileIdx *4 , finalLightCount ) ;
 }
 }
 ```
 
 >PDF:http://www.jp.square-enix.com/tech/library/pdf/2023_FFXVIShadowTechPaper.pdf