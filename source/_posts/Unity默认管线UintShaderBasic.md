---
title: Unity默认管线UintShaderBasic
photos: images/FirstShader.png
date: 2024-09-11 16:32:50
tags: [学习,Unity]
categories: 游戏技术
---
# UnityShader基础知识

## UnityCG.cginc
![images1](images/UnityCG_cginc.png "subshader")  
UnityCG.cginc是与Unity捆绑在一起的shader包含文件之一  
UnityShaderVariables.cginc定义了渲染所需的一大堆着色器变量，如变换、相机和灯光数据。这些都由 Unity 在需要时设置    
HLSLSupport.cginc进行了平台设置，因此无论你针对哪个平台，都可以使用相同的代码。因此无需担心使用特定于平台的数据类型等  
UnityInstancing.cginc专门用于实例化支持，虽然它不直接包含该文件，但它依赖于UnityShaderVariables

## subshader
![images1](images/subshader.png "subshader")  
subshader可以使用多个shader变体组合在一起,允许构建不同平台跟细节,例如为windows使用一个subshader,移动设备使用另一个subshader

## pass
![images1](images/subshader_pass.png "subshader")
subshader必须至少包含一个pass,shaderpass是实际被渲染的地方,可以有多个,有多个pass意味着被渲染多次但对于效果来说是必须的

## CG
![images1](images/PASS_CG.png "subshader")
Unity的shaderLanguage HLSL跟CG的变体,必须用关键词指示CGPROGRAM, ENDCG

## #program
![images1](images/SUBSHADER_PROGRAM.png "subshader")
![images1](images/shader函数.png "subshader")
必须用#program告诉编译器需要哪些程序

## SV_XXX
![images1](images/SV_POSITION_TARGET.png "subshader")
标注函数要写入到哪里SV_TARET是默认的shader目标, SV指的是SystemValue
```C++
	float4 MyVertexProgram (
				float4 position : POSITION,
				out float3 localPosition : TEXCOORD0
			) : SV_POSITION {
				return mul(UNITY_MATRIX_MVP, position);
			}

    float4 MyFragmentProgram (
				float4 position : SV_POSITION,
				float3 localPosition : TEXCOORD0
			) : SV_TARGET {
				return float4(localPosition, 1);
			}
```
也可以这样给position赋值,但要注意Vertex到Fragment需要有out

## Property
![images1](images/shader_property.png "subshader")
Property中生命的变量一般都是_开头+大写字母+小写字母(约定),

```C++
	float4 _Tint;

    float4 MyVertexProgram (float4 position : POSITION) : SV_POSITION {
        return mul(UNITY_MATRIX_MVP, position);
    }

    float4 MyFragmentProgram (
        float4 position : SV_POSITION
    ) : SV_TARGET {
        return _Tint;
    }
```
使用时要注意名称一样

## Struct
![images1](images/Interpolators.png "subshader")
可以声明一个,或者多个Struct来定义数据使代码更简洁

_ST后缀的变量可以代表Unity贴图的tiling跟Offset
>_ST后缀代表Scale 和 Translation，或类似的东西。为什么不使用_TO来表示Tiling 和 Offset？因为 Unity 一直使用_ST，并且向后兼容性要求它保持这种方式，即使术语可能已更改。
