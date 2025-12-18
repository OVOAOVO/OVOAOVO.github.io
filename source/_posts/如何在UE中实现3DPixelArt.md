---
title: 如何在UE中实现3DPixelArt
photos: images/3d-pixel-art-water.png
date: 2025-12-18 20:14:35
tags: [学习]
categories: 游戏技术
---

## 首先要说明的是不建议使用UE（❌️）

为什么？因为Godot才是最合适的，网上有很多实现3Dpixel的教程，最关键的是Godot是能所有像素对齐的，所以你可以通过默认功能规避很多麻烦:)

但我用UE:(

## 首先要说明的是如何使得画面像素化（✅️）

像素化的核心其实就是降低分辨率

目前的方式其实有很多
1. 后处理
2. 降低Scene Render Percentage
3. sceneCapture捕捉画面到texture缩小后放大

那都可以用吗?

>当然不可以，原因是降低分辨率后像素边缘在移动以及旋转时会闪烁，并且不断地抖动，能用的其实只有3

## 接下来就是如何保证像素稳定

如果你成功的实现了上述任意像素化方式，你肯定会看到物体边缘在不断地闪烁(如果你不清楚，可以试一下，你会知道我在说什么的)

>参见GDC:https://www.youtube.com/watch?v=ZW8gWgpptI8

《得益于》UE先进的3D渲染技术，有几个点需要注意

1. 相机需要正交，实现像素画面
2. 抗锯齿需要关闭，因为要实现像素风格，我们不能启用任何抗锯齿
3. 阴影要使用ShadowMap，UE的虚拟阴影贴图会导致阴影边缘的闪烁，光追我没有尝试，但理论上讲pixel相机正交的情况下，RayTracing因为正交相机没有透视效果，所有的射线都会平行，效果肯定是不正确的
4. cascadedShadowMap中的距离要与相机的springArm匹配，因为在shadowMap下，DirectionLight的cascadeShadowMap才是动态阴影，这个阴影质量我们必须要调
5. AO效果只有SSAO能用，原因同3
6. 等等基于正交相机下的影响

在上面这些配置好后，我们就可以开始了
```C++
void ABSPlayerController::Tick(float DeltaSeconds)
{
    Super::Tick(DeltaSeconds);
    APawn* ControlledPawn = GetPawn<APawn>();
    USpringArmComponent* SpringArm = ControlledPawn->FindComponentByClass<USpringArmComponent>();
    UCameraComponent* cameraA = ControlledPawn->FindComponentByClass<UCameraComponent>();
    USceneCaptureComponent2D* SceneCaptureCompoent = ControlledPawn->FindComponentByClass<USceneCaptureComponent2D>();

    FVector WorldLocation = cameraA->GetComponentLocation();
    FVector RightVector = cameraA->GetRightVector();

    FRotator SpringArmWorldLocation = SpringArm->GetRelativeRotation();
    FVector A = WorldLocation.RotateAngleAxis(SpringArmWorldLocation.Pitch, RightVector);
    FVector B = A.RotateAngleAxis(SpringArmWorldLocation.Yaw * (-1.f), ControlledPawn->GetActorUpVector());

    FVector SnappedLocation = B.GridSnap(SceneCaptureCompoent->OrthoWidth / SceneCaptureCompoent->TextureTarget->SizeX);
    //FVector subpixelOffset = B - SnappedLocation;

    //NOTE: screen space offset TO calculate UV offset(NOT USE)
    //ScreenDX = FVector::DotProduct(subpixelOffset, cameraA->GetRightVector());
    //ScreenDY = FVector::DotProduct(subpixelOffset, cameraA->GetUpVector());

    FVector C = SnappedLocation.RotateAngleAxis(SpringArmWorldLocation.Yaw, ControlledPawn->GetActorUpVector());
    FVector D = C.RotateAngleAxis(SpringArmWorldLocation.Pitch * (-1.f), RightVector);

    SceneCaptureCompoent->SetWorldLocation(D, false, nullptr, ETeleportType::None);
}
 ```
这是什么？

这个是在UE中如何正确的Snap到grid，这个其实就是原理，我们需要把相机的偏移换算成像素的偏移，而且偏移的大小是由本身的正交相机的size/RT的size(别忘了我们使用的方法3，SceneCapture场景到RT上)，这个大小其实就是折算后的屏幕分辨率下的onePixelSize。

但这还没完，我们要实现的是相机平滑移动，单纯吸附到像素，你的画面移动会变成一卡一卡的
>参见：https://www.youtube.com/watch?v=LQfAGAj9oNQ

我们需要subpixelMove，但UE里面没有这个功能。

还记的我们使用方法3吗？这时候我们render到RT后，可以通过新建材质给到Widget，同时，我们把springArm的worldSpace转换成screenSpace，再去setLocattion，这就能得到一个smooth相机了。

![alt text](images/BP_CaptureTickick.png)

### BUT？
虽然但是，我也想没有这个BUT，但他确实发生了，如果这时候你查看你的像素边缘，依然有一些抖动，他可以通过降低RT分辨率极大的减缓。

为什么会发生这个？

我~不~知~道~Ciallo～(∠・ω< ) 

截止2025/12/15，我确实不太清楚产生这个的原因，可能是因为本身在移动时不断snap导致的，那么我们也许可以通过将角色移动与相机移动拆开来，让相机delay下他的跟随


### 会有很多小问题值得注意
比如RT的采样方式应该设置为2Dpixel之类的，这样可以让你的material采样的时候能够避免线性插值带来的模糊(其实就是nearest采样，我单独写了个自定义最邻近采样发现没有什么必要)，或许这部分我应该总结下？但有点又臭又长还没必要:(

## 总而言之

如果你想做一个像素游戏，其实最好的选择是godot，其次是Unity(自定义管线方便)，最后才是UE

have fun :)