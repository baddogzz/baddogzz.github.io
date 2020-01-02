---
layout: post
title: "关于ComputeScreenPos和ComputeGrabScreenPos的差别"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
  - Shader
---

### 一个Bug

今天QA报了一个渲染相关的bug：一个用了 **扭曲** 效果的翅膀特效在场景相机下显示正常，但是在UI相机上却有问题，截图如下：

![img](/img/grabuv-bug/screenshot1.png){:height="45%" width="45%"} 

扭曲背景 **上下颠倒** 了。

---

### Bug的修正

这里用的 **扭曲shader** 是我们的美术同学从他们前项目搬过来的，代码很简单：

+ 用 **GrabPass** 抓取当前屏幕做为扭曲背景。
+ 添加 **UV扰动** 后再采样屏幕，即可达到扭曲效果。

问题是，这里采样 **GrabTexture** 的时候用的是 **screenUV** 而非 **grabUV**，代码如下：

*顶点着色器:*

```
	o.screenPos = ComputeScreenPos (o.pos);
``` 

*像素着色器：*

```
	float2 sceneUVs = (i.screenPos.xy / i.screenPos.w) + (_Value * diffuseTex.a * float2(diffuseTex.r, diffuseTex.g) * i.vertexColor.a);
	half4 sceneColor = tex2D(_GrabTexture, sceneUVs);
```

修正这个问题很简单，把 **ComputeScreenPos** 换成 **ComputeGrabScreenPos** 即可，修正后的代码如下：

*顶点着色器：*

```
o.screenPos = ComputeGrabScreenPos (o.pos);
```

调整完之后就正常了，如下图：

![img](/img/grabuv-bug/screenshot2.png){:height="45%" width="45%"} 

---

### 关于ComputeScreenPos和ComputeGrabScreenPos的差别

修正容易，但是搞清楚 **ComputeScreenPos** 和 **ComputeGrabScreenPos** 的差别却要费一些功夫。我们看一下相关代码：

```
inline float4 ComputeNonStereoScreenPos(float4 pos) {
    float4 o = pos * 0.5f;    
    o.xy = float2(o.x, o.y*_ProjectionParams.x) + o.w;
    o.zw = pos.zw;
    return o;
}

inline float4 ComputeScreenPos(float4 pos) {
    float4 o = ComputeNonStereoScreenPos(pos);
#if defined(UNITY_SINGLE_PASS_STEREO)
    o.xy = TransformStereoScreenSpaceTex(o.xy, pos.w);
#endif
    return o;
}

inline float4 ComputeGrabScreenPos (float4 pos) {
    #if UNITY_UV_STARTS_AT_TOP
    float scale = -1.0;
    #else
    float scale = 1.0;
    #endif
    float4 o = pos * 0.5f;    
    o.xy = float2(o.x, o.y*scale) + o.w;
#ifdef UNITY_SINGLE_PASS_STEREO
    o.xy = TransformStereoScreenSpaceTex(o.xy, pos.w);
#endif
    o.zw = pos.zw;
    return o;
}
```

通过分析可以发现，这两个函数的主要差别就是 **UNITY_UV_STARTS_AT_TOP** 和 **_ProjectionParams.x** 的差别。

我们知道 **RenderTexture** 的纹理坐标在 **Direct3D-like** 平台和 **OpenGL-like** 平台存在差异：

+ Direct3D-like平台，UNITY_UV_STARTS_AT_TOP = 1，纹理坐标0在顶部，并往下增长。
+ OpenGL-like平台，UNITY_UV_STARTS_AT_TOP = 0，标识纹理坐标0在底部，并往上增长。

当渲染到纹理的时，Unity遵从 **OpenGL-like** 平台的约定。当工作在 **Direct3D-like** 平台时，为了向上隐藏这个平台差异，Unity会 **翻转投影矩阵** 从而翻转 **RenderTexture**，这样既遵从了 **OpenGL-like** 平台的约定，又可以获取正确的采样结果。

**_ProjectionParams.x** 标识了投影矩阵是否经过翻转。

+ _ProjectionParams.x = 1表示没有翻转。
+ _ProjectionParams.x = -1表示翻转。
 
那么，是不是 **Direct3D-like** 平台下 **RenderTexture** 一定会进行翻转操作呢？如果没有翻转，而Unity又采用了 **OpenGL-like** 平台的约定，这种情况要怎么处理呢？

事实上，Unity在一些情况下确实不会翻转 **RenderTexture**，它的帮助文档 **Platform-specific rendering differences** 这一章节列举了 **Direct3D-like** 平台下不翻转 **RenderTexture** 的几种情况：

+ Image Effects + 抗锯齿
+ GrabPass

对于 **GrabPass**，Unity文档做了特别说明：在 **Direct3D-like** 平台下，**GrabPass** 不会进行 **RenderTexture** 的翻转操作，因此我们需要在shader中手工翻转uv以获取正确的采样结果。

因此，**ComputeGrabScreenPos** 这里只需要判断 **UNITY_UV_STARTS_AT_TOP** 的取值：如果是 **Direct3D-like** 平台(UNITY_UV_STARTS_AT_TOP = 1)，我们就需要手工翻转uv，如果是 **OpenGL-like** 平台(UNITY_UV_STARTS_AT_TOP = 0)，则无需翻转uv。

```
inline float4 ComputeGrabScreenPos (float4 pos) {
    #if UNITY_UV_STARTS_AT_TOP
    float scale = -1.0;
    #else
    float scale = 1.0;
    #endif
    float4 o = pos * 0.5f;    
    o.xy = float2(o.x, o.y*scale) + o.w;
#ifdef UNITY_SINGLE_PASS_STEREO
    o.xy = TransformStereoScreenSpaceTex(o.xy, pos.w);
#endif
    o.zw = pos.zw;
    return o;
}
```

**Direct3D-like** 平台下，如果我们用 **_ProjectionParams.x** 来判断是否需要手工翻转uv就错了，因为 **RenderTexture** 并未发生翻转，此时 _ProjectionParams.x = 1。

---

### 关于投影矩阵的翻转

bug是修正了，也知道了原因：对于**GrabPass**，我们应该用 **UNITY_UV_STARTS_AT_TOP** 而非 **_ProjectionParams.x** 去判断是否要手工进行uv翻转。

但是还有一个疑问没有解开：前文说了，这个 **扭曲** 特效在场景相机下工作正常，在UI相机下才有问题，这又是为什么呢？

说到相机，我们游戏内一共3个相机，渲染顺序如下：

场景相机（开后处理） --> UI相机1（关后处理） --> UI相机2（关后处理）

出现问题的相机是 **UI相机2**，此时我们并没有开 **抗锯齿**，并且 **场景相机** 处于 **关闭** 状态。

如果我们打开 **场景相机**，或者把 **UI相机1** 的后处理打开，那么这个bug也不会出现。

似乎 **多相机** 以及 **Image Effect的开关** 也会影响 **_ProjectionParams.x** 的设值。

可惜的是，Unity文档对 **_ProjectionParams.x** 就简单的一句说明：

> x is 1.0 (or –1.0 if currently rendering with a flipped projection matrix)

没有源码的情况下，这个问题就比较难说清楚了，反正 **-1** 代表 **投影矩阵翻转**。

早前在写 [Fantastic SSR Water](https://assetstore.unity.com/packages/vfx/shaders/fantastic-ssr-water-154020?aid=1101l85Tr) 这个插件的时候，我也遇到过类似的问题。

[Fantastic SSR Water](https://assetstore.unity.com/packages/vfx/shaders/fantastic-ssr-water-154020?aid=1101l85Tr) 是一款Unity水的插件，用 **屏幕空间反射** 去计算水的反射。

+ 因为需要在屏幕空间计算 **光线步进**，因此我需要屏幕坐标 **screenUV**。
+ 因为用了 **GrabPass** 去抓取屏幕颜色以计算反射颜色，因此我还需要 **grabUV**，用于 **GrabTexture** 的采样。

当时，我错误的把 **screenUV** 和 **grabUV** 等同了，然后发现只有在特定的设置下渲染才正确，包括：

+ 平台的选择
+ 前向渲染/延迟渲染的选择
+ 抗锯齿开关的选择

后面，我用 **ComputeScreenPos** 去计算 **screenUV**，用 **ComputeGrabScreenPos** 去计算 **grabUV**，终于在各种设置组合下渲染都正确了。

所以，尽管Unity文档对 **_ProjectionParams.x** 的说明不够清晰，但是记住以下几点，代码至少不会出错：

+ 尽量用Unity封装好的函数来处理平台差异。
+ ComputeScreenPos 不等同于 ComputeGrabScreenPos，要正确区分应用场合。
+ UNITY_UV_STARTS_AT_TOP = 1 代表 **Direct3D-like** 平台，但 **Direct3D-like** 平台下 **_ProjectionParams.x** 未必等于 -1，两者意涵是不同的。

好了，拜拜。





























