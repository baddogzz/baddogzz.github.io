---
layout: post
title: "华为手机关于LOAD_TEXTURE2D的Bug"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
  - Shader
---

## 一个华为手机专属Bug

今天又发现了一个 **华为手机** 专属bug，如下图：

![](/img/hw-loadx-bug/screenshot1.jpg)

原本 [LUX](https://assetstore.unity.com/packages/vfx/shaders/lux-urp-essentials-150355?aid=1101l85Tr) 表现正常的水在我的 **华为 Meta 30** 上表现呈 **条带状**。

从表现上看跟 **深度计算** 相关，定位了一下原因，发现问题代码如下：

```
#if defined(SHADER_API_GLES)
    float refractedSceneDepth = SAMPLE_DEPTH_TEXTURE_LOD(_CameraDepthTexture, sampler_CameraDepthTexture, screenUV + offset, 0);
#else
    float refractedSceneDepth = LOAD_TEXTURE2D_X(_CameraDepthTexture, _CameraDepthTexture_TexelSize.zw * saturate(screenUV + offset) * 0.9999f ).x;
#endif
```

**SHADER_API_GLES** 代表的是 **gles2.0** 设备，走的是 **SAMPLE_TEXUTRE** 的分支。

而我们工程的 **Graphics APIs** 首选项是 **OpenGLES3**，并且我的华为手机也支持，所以最终代码走的是 **LOAD_TEXTURE** 这个分支。

## Texture Sample Vs Texture Load

那么 **SAMPLE_TEXTURE** 和 **LOAD_TEXTURE2D** 有什么区别呢？

以 **GLES3.0** 代码为例：

```
#define SAMPLE_DEPTH_TEXTURE_LOD(textureName, samplerName, coord2, lod)    SAMPLE_TEXTURE2D_LOD(textureName, samplerName, coord2, lod).r
```

```
#define LOAD_TEXTURE2D(textureName, unCoord2)    textureName.Load(int3(unCoord2, 0))
```

关于 **Texture Sample** 和 **Texture Load** 的差别，可以参考这个帖子：[Difference between texture.Sample and texture.Load](https://gamedev.stackexchange.com/questions/65845/difference-between-texture-load-and-texture-sample-methods-in-directx/65853)，简单来说，两者差别如下：

+ Texture Sample是纹理采样，会应用 **纹理寻址** 和 **纹理过滤**。

+ Texture Load只是加载某一个特定位置的 **texel**。

道理上，作者希望 **gles3.0** 以上的设备都用 **LOAD_TEXTURE** 的方式来获取深度值，但是偏偏遇到华为手机就跪了。

## 原因

华为手机的 **浮点数** 问题我不是第一次遇到了，之前也写过，[https://baddogzz.github.io/2020/04/27/Mali-Float-Presion/](https://baddogzz.github.io/2020/04/27/Mali-Float-Presion/)。

这次这个问题应该还是浮点数计算的问题，使用 **Texture Load** 方式，我们需要根据 **UV坐标** 计算出 **texel坐标**，代码如下：

```
_CameraDepthTexture_TexelSize.zw * saturate(screenUV + offset) * 0.9999f
```

如果 **texel坐标** 超出纹理范围，**LOAD_TEXTURE2D** 会返回 **0**。

针对这种情况，作者已经对 **screenUV + offset** 做了 **saturate** 保护并且还乘了 **0.9999f**，想必也是为了规避浮点数计算的误差，但是华为手机依然还是跪了。

## 修正

修正的方式也简单，这里不再区分 **SHADER_API_GLES**，统一走 **SAMPLE_TEXTURE** 即可，如下：

```
float refractedSceneDepth = SAMPLE_DEPTH_TEXTURE_LOD(_CameraDepthTexture, sampler_CameraDepthTexture, screenUV + offset, 0);
```

这样改之后，我的 **Meta 30** 终于正常了，如下图：

![](/img/hw-loadx-bug/screenshot2.jpg)

此外，我发现如果 **Graphics APIs** 选 **Vulkan**，我的华为手机也正常，好吧，我服了，拜拜。


























































































































































































































































