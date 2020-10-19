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

## 华为专属Bug

今天又发现了一个 **华为手机** 专属bug，如下图：

原本 [LUX](https://assetstore.unity.com/packages/vfx/shaders/lux-urp-essentials-150355?aid=1101l85Tr) 表现正常的水在我的 **华为 Meta 30** 上表现呈 **条带状**。

从表现上看是 **深度计算** 相关的问题，定位了一下原因，发现问题代码如下：

```
#if defined(SHADER_API_GLES)
    float refractedSceneDepth = SAMPLE_DEPTH_TEXTURE_LOD(_CameraDepthTexture, sampler_CameraDepthTexture, screenUV + offset, 0);
#else
    float refractedSceneDepth = LOAD_TEXTURE2D_X(_CameraDepthTexture, _CameraDepthTexture_TexelSize.zw * saturate(screenUV + offset) * 0.9999f ).x;
#endif
```

**SHADER_API_GLES** 代表的是 **gles2.0** 设备，走 **SAMPLE_TEXUTRE** 的分支，而我的手机属于 *高端* 设备，走的是 **LOAD_TEXTURE** 这个分支。

## Texture Sample Vs Texture Load

那么 **SAMPLE_DEPTH_TEXTURE_LOD** 和 **LOAD_TEXTURE2D_X** 有什么区别呢？

以 **GLES3.0** 代码为例：

```
#define SAMPLE_DEPTH_TEXTURE_LOD(textureName, samplerName, coord2, lod)    SAMPLE_TEXTURE2D_LOD(textureName, samplerName, coord2, lod).r
```

```
#define LOAD_TEXTURE2D(textureName, unCoord2)    textureName.Load(int3(unCoord2, 0))
```

关于 "Texture Sample" 和 "Texture Load" 的差别，可以参考这个帖子：[Difference between texture.Sample and texture.Load](https://gamedev.stackexchange.com/questions/65845/difference-between-texture-load-and-texture-sample-methods-in-directx/65853)

+ Texture Sample是纹理采样，会应用纹理寻址和纹理过滤。

+ Texture Load只是加载某一个特定位置的 **texel**。

所以华为手机用 **Texture Load** 就会表现异常？

## 



























































































































































































































































