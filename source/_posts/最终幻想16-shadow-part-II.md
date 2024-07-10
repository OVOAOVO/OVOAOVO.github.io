---
title: 最终幻想16 shadow part II
photos: images/FFXVI.png
date: 2024-03-17 22:18:08
tags: [AAA,学习]
categories: 游戏技术
---
# FFXVI Shadow Part II

## 正文 

### Part II ：Approx

主要有两个目标  
1. 查找每个阴影图块的潜在有效光源
2. 储存在构建的光源列表里

从depthBuffer 1 获取每个shadow tile的最小和最大场景深度，通过他们于屏幕坐标组合生成Froxel(视锥体素，使用tile的坐标跟最近/远深度的位置定义视锥体的子部分)

从Tile Light Mask Buffer 和 zBin Buffer中找到Froxel有效的光源下标，通过使用快速球体测试以及Froxel的范围点判断每个光源
该测试取自文章
>https://bartwronski.com/2017/04/13/cull-that-cone/

输出1: Tile Group Light Mask Buffer（tile组用于查找近似的光的可见）  
输出2: Tile Light SlotID Buffer（Tile Light SlotID Buffer GPU 缓冲区，用于存储在近似阶段分配后每个 Shadow Tile 的 Light SlotID。）  
（Tile Light SlotID：用于计算 Tile 的灯光列表在内存缓冲区中的起始位置。标识符对 Early Light List 缓冲区和 Final Light List 缓冲区都有效）  
（Early Light List 缓冲区 GPU 缓冲区，用于存储每个阴影图块的有效光源列表。当光范围内有像素而不进行阴影测试时，将灯光添加到其中）  
（Final Light List Buffer GPU 缓冲区，用于存储每个 Shadow Tile 的有效灯光列表。当光源范围内有像素时，将光源添加到其中，如果整个tile处于阴影中，则将其排除）

   1. 当检测到与场景视图远平面（例如天空盒的像素）匹配的最大场景深度时，我们单独读取每个像素的场景深度以找到最大有效深度。这大大减少了导致的误报灯的数量。  
   2. 单线程接入，大大减少了原子争用，提升了性能。
   
### "SlotID 分配器"计算着色器伪代码
```C++
globallycoherent RWByteAddressBuffer rw_SlotCounters ;

// ========================================================================
// Allocate a ShadowSlotID that will be used to store list of lights
uint AllocateShadowSlotID (in uint inTileIndex , in uint inTileLightCount )
{
// Reserve fixed amount of memory for debuging with predictable location
if( DEBUG_FIXED_LIGHTCOUNT > 0 ) {
return inTileIndex * RoundUp ( DEBUG_FIXED_LIGHTCOUNT , 3) ;
}

// Round up to next multiple becase 3 indices stored per uint
inTileLightCount = RoundUp ( inTileLightCount , 3) ;
uint tileGrpLightCount = WaveActiveSum ( tileLightCount ) ;
uint tileGrpSlotID = 0;

// Single thread allocates space for entire TileGroup
if( WaveIsFirstLane () ) {
static const uint kShadowCounterOffset = 0;
rw_SlotCounters . InterlockedAdd ( kShadowCounterOffset ,
tileGrpLightCount ,
tileGrpSlotID ) ;
}

// Tile get its SlotID by offsetting the starting location of the
// Group reserved memory with the sum of previous Tile Slots count
uint shadowSlotID = WaveReadLaneFirst ( tileGrpSlotID ) ;
shadowSlotID += WavePrefixSum ( inTileLightCount ) ;
return shadowSlotID ;
}

// ========================================================================
// Allocate a VisibilitySlotID that will be used to store the visibility
// of one light at each pixel of a Tile (1 slot = 8x8 pixels (64 bytes ))
uint AllocateVisibilitySlotID ()
{
// Single thread allocates space for entire Tile
uint slotID ;
if( WaveIsFirstLane () ) {
static const uint kVisibilityCounterOffset = 4;
rw_SlotCounters . InterlockedAdd ( kVisibilityCounterOffset , 1 , slotID ) ;
}

uint visibilitySlotID = WaveReadLaneFirst ( slotID ) ;
return visibilitySlotID ;
}
```
### “Find Lights Approximate”计算着色器伪代码
```C++
RWByteAddressBuffer rw_TileShadowSlotIds ;
RWByteAddressBuffer rw_TileGroupLightMask ;

// Work unit outputs 1 TileGroup (8 x8 Tiles ), each thread evaluate 1 Tile
[ numthreads ( TILE_LIGHTGROUP_SIZE , TILE_LIGHTGROUP_SIZE , 1) ]
void CS_Phase2_FindLightsApprox ( uint2 inTileCoord : SV_DispatchThreadID )
{
bool isValidTile = all ( inTileCoord < cCommon . m_TileResolution .xy) ;
if( isValidTile ) {
const uint tileIdx = GetTileIndex ( inTileCoord ) ;
const float2 tileNearFar = GetTileNearFarDepth ( inTileCoord ) ;
const int2 zBinIndexMinMax = GetZBinIndexFromDepth ( tileNearFar ) ;
const ZBin binRange = GetRangeZBinFromIndex ( zBinIndexMinMax ) ;
uint lightGroupMask = GetTileLightGroupMask ( inTileCoord ) ;
lightGroupMask = IntersectLightGroup ( binRange , lightGroupMask ) ;
uint tileLightCount = 0;
uint validLightGrpMask = 0;
// -- ---- ---- ---- ----- ---- ---- ---- ----- ---- ---- ---- ----- ---- ---- --- ---
// Process each valid group of 32 lights
while ( lightGroupMask ) {
uint validLightMask = 0;
uint lightGrpBit = firstbitlow ( lightGroupMask ) ;
uint lightMask = GetTileLightMask ( lightGrpBit ) ;
lightMask = IntersectLightMask ( binRange , lightMask ) ;
// -- ----- ---- ----- ----- ---- ----- ----- ----- ---- ----- ----- ---- ----- -
// Process each valid light in current group of 32 lights
while ( lightMask ) {
uint lightBit = firstbitlow ( lightMask ) ;
uint lightIdx = lightGrpBit *32 + lightBit ;
LightInfo lightInfo = GetLightInfo ( lightIdx ) ;
// All intersection tests between ’Light / Froxel ’ here
bool bValidLight = LightFroxelTests ( lightInfo , inTileCoord ) ;
lightMask ^= 1u << lightBit ; // Remove processed light
if( bValidLight ) {
tileLightCount += 1;
validLightMask |= 1 << lightBit ;
}
}
// Store light indices validity of current group with 32 bits mask
uint tileGroupLightMask = WaveActiveBitOr ( validLightMask ) ;
if( WaveIsFirstLane () ) {
uint lightMaskAdr = GetLightMaskEntryAdress ( tileIdx , lightGrpBit ) ;
rw_TileGroupLightMask . Store ( lightMaskAdr , tileGroupLightMask ) ;
}
// -- ----- ---- ----- ----- ---- ----- ----- ----- ---- ----- ----- ---- ----- --
// Detect if current 32 lights group is valid in any Tile of TileGroup
validLightGrpMask |= tileGroupLightMask != 0 ? 1 << lightGrpBit : 0;
lightGroupMask ^= 1 << lightGrpBit ; // Remove processed group
}
// -- ---- ---- ---- ----- ---- ---- ---- ----- ---- ---- ---- ----- ---- ---- --- ---
// Store LightGroup mask entry for the TileGroup
if( WaveIsFirstLane () ) {
uint lightMaskAdr = GetLightMaskGroupAdress ( tileIdx ) ;
rw_TileGroupLightMask . Store ( lightMaskAdr , validLightGrpMask ) ;
}
// Allocate and store each Tile ShadowSlotIDs
uint shadowSlotID = AllocateShadowSlotID ( tileIdx , tileLightCount ) ;
rw_TileShadowSlotIds . Store ( tileIdx *4 , shadowSlotID ) ;
}
}
```

>PDF:http://www.jp.square-enix.com/tech/library/pdf/2023_FFXVIShadowTechPaper.pdf