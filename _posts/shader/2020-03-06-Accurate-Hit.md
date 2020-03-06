---
layout: post
title: "更精确SSR的交点检测"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
  - Shader
---

## Unreal的SSR交点检测

关于 **屏幕空间反射**，可以参考文章很多，我觉得这篇写得还是蛮好的：[Screen Space Glossy Reflections](http://roar11.com/2015/07/screen-space-glossy-reflections/)，借个图用用：

![](/img/accurate-hit/screenshot1.png)

本文不考虑 **粗糙度**，也不考虑 **多条反射射线**，只借鉴一下 **Unreal** 引擎在处理 **单条射线光线步进** 时对 **交点** 的计算方式，用来改进我的插件 [LWRP/URP SSR Water](https://assetstore.unity.com/packages/vfx/shaders/lwrp-urp-ssr-water-155402?aid=1101l85Tr) 的反射表现。

直接看下面这段最简单的 **光线步进** 代码片段：
 
```
#if SCALAR_BRANCHLESS    

    float MinHitTime = 1;
    float LastDiff = 0;

    float SampleTime = StepOffset * Step + Step; 

    UNROLL
    for( int i = 0; i < NumSteps; i++ )
    {
        float3 SampleUVz = RayStartUVz + RayStepUVz * SampleTime;
        
        // Use lower res for farther samples
        float Level = Roughness * (i * 4.0 / NumSteps) + HZB_LEVEL_OFFSET;
        float SampleDepth = Texture.SampleLevel( Sampler, SampleUVz.xy, Level ).r;

        float DepthDiff = SampleUVz.z - SampleDepth;
        bool Hit = abs( DepthDiff + CompareTolerance ) < CompareTolerance;

        // Find more accurate hit using line segment intersection 
        float TimeLerp = saturate( LastDiff / (LastDiff - DepthDiff) ); 
        float IntersectTime = SampleTime + TimeLerp * Step - Step; 
        float HitTime = Hit ? IntersectTime : 1;
        MinHitTime = min( MinHitTime, HitTime );

        LastDiff = DepthDiff;     

        SampleTime += Step;       
    }

    float3 HitUVz = RayStartUVz + RayStepUVz * MinHitTime;

    Result = float4( HitUVz, MinHitTime );
```

这里有一句比较有意思的注释：

> Find more accurate hit using line segment intersection 

在判断出射线和场景相交后，**Unreal** 并不着急返回 **当前射线终点** 对应的 **屏幕坐标**，而是根据上一段射线终点和当前射线终点相对于场景深度的偏移 **插值** 出一个更加准确的 **屏幕坐标**。

有点绕口，画个图就明了了：

![](/img/accurate-hit/screenshot2.jpg)

上图显示的是射线深度刚超过场景深度时的情形，图中 **CurrentDiff** 是正数，**LastDiff** 是负数，如果考虑正负号，则交点的屏幕坐标计算公式如下：

```
HitScreenUV = lerp(LastScreenUV, CurrentScreenUV, -LastDiff / (CurrentDiff - LastDiff)))
```

上面代码中的 **LastScreenUV** 即上一段射线终点对应的 **屏幕坐标**，**CurrentScreenUV** 即当前射线终点对应的 **屏幕坐标**。

把 **-LastDiff / (CurrentDiff - LastDiff)** 的分子分母都 **乘以-1** 即：

```
HitScreenUV = lerp(LastScreenUV, CurrentScreenUV, LastDiff / (LastDiff - CurrentDiff)))
```

这样就和 **Unreal** 的代码对应上了。

## 效果对比

我在 [LWRP/URP SSR Water](https://assetstore.unity.com/packages/vfx/shaders/lwrp-urp-ssr-water-155402?aid=1101l85Tr) 的光线步进交点计算中并没有上面的 **插值** 操作，而是判断出相交后直接返回当前射线终点对应的屏幕坐标。

配合 **抖动**，在 **采样Step** 和 **屏幕分辨率** 比较高时，这样的做法表现其实也还不错。

不过，当我把分辨率调到 **1200 x 600** 时候，之前的表现就一般般了，如下图：

![](/img/accurate-hit/screenshot3.png)

添加插值后，还是 **1200 x 600** 的分辨率，表现就好多了：

![](/img/accurate-hit/screenshot4.png)

如果把分辨率提到 **2160 x 1080** 这种常见的手机分辨率，表现就更好了：

![](/img/accurate-hit/screenshot5.png)

做为一个水的shader，这样就差不多了，因为加上水波纹后，一切都是浮云：

![](/img/accurate-hit/screenshot6.png)

好了，拜拜！












































































































































































