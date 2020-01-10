---
layout: post
title: "关于间接高光的水平遮蔽(Horizon Occlusion)"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
  - Shader
---

## Horizon Occlusion

第一次看到 **Horizon Occlusion** 是在读 [Lux Plus](https://assetstore.unity.com/packages/vfx/shaders/lux-plus-physically-based-shader-framework-74897?aid=1101l85Tr) 源码的时候。

**Horizon Occlusion** 是关于 **间接高光** 的 **水平遮蔽** 计算，典型的代码如下：

```
// Horizon Occlusion
#if defined (UNITY_PASS_FORWARDBASE)
    #if LUX_HORIZON_OCCLUSION
	    gi.indirect.specular *= GetHorizonOcclusion(viewDir, s.Normal, s.worldNormalFace, HORIZON_FADE);	
    #endif
#endif

// Direct lighting uses the Lux BRDF
half4 c = Lux_BRDF1_PBS(s.Albedo, s.Specular, oneMinusReflectivity, s.Smoothness, s.Normal, viewDir,
		// Deferred expects these inputs to be calculates up front, forward does not. So we simply fill the input struct with zeros.
		half3(0, 0, 0), 0, 0, 0, 0,
		nl,
		ndotlDiffuse,
		gi.light, gi.indirect, specularIntensity, s.Shadow);

c.rgb += UNITY_BRDF_GI (s.Albedo, s.Specular, oneMinusReflectivity, s.Smoothness, s.Normal, viewDir, s.Occlusion, gi);
```

在物理着色之前，先计算 **间接高光的水平遮蔽**，即这里的 **GetHorizonOcclusion** 函数，函数就3行，代码如下：

```
// Horizon Occlusion for Normal Mapped Reflections: http://marmosetco.tumblr.com/post/81245981087
float GetHorizonOcclusion(float3 V, float3 normalWS, float3 vertexNormal, float horizonFade)
{
    float3 R = reflect(-V, normalWS);
    float specularOcclusion = saturate(1.0 + horizonFade * dot(R, vertexNormal));
    // smooth it
    return specularOcclusion; // * specularOcclusion;
}
```

题外话，今天在读 [Advanced Terrain Grass](https://assetstore.unity.com/packages/tools/terrain/advanced-terrain-grass-100014?aid=1101l85Tr) 代码的时候又看到了这个函数，再一看作者，原来是同一个人，：）

作者 **forst** 有很多经典的插件，确实让我学到了很多，这里给他做一个广告：[forst的商店主页](https://assetstore.unity.com/publishers/408)。

好，现在回归主题，那么什么是 **间接高光的水平遮蔽** 呢？

## 原理和实现

关于 **Horizon Occlusion**，[这篇文章](https://marmosetco.tumblr.com/post/81245981087) 写的很清楚了，这里再啰嗦一遍。

我们知道Unity **间接高光** 的计算依赖 **视线向量** 相对 **法线** 的 **反射向量**，**反射向量** 的计算代码如下：

```
g.reflUVW   = reflect(-worldViewDir, Normal);
```

考虑一个完全的镜面，我们不可能接收到镜面下面的环境反光，但是当我们引入 **法线贴图** 后，法线会偏转，这个时候我们的反射射线可能会达到镜面的下面，如下图所示：

![img](/img/horizon-occlusion/screenshot1.png)

如果以这个反射射线去计算环境高光，就会 **漏光** 了，如下图：

![img](/img/horizon-occlusion/screenshot2.png)

**Horizon Occlusion** 就是为了处理这种 **漏光** 表现，加了 **Horizon Occlusion** 后效果如下：

![img](/img/horizon-occlusion/screenshot3.png)

下面看一下作者的计算方式：

```
float GetHorizonOcclusion(float3 V, float3 normalWS, float3 vertexNormal, float horizonFade)
{
    float3 R = reflect(-V, normalWS);
    float specularOcclusion = saturate(1.0 + horizonFade * dot(R, vertexNormal));
    // smooth it
    return specularOcclusion; // * specularOcclusion;
}
```

代码中的 **R** 即前文提到的 **视线向量** 相对于 **法线** 的反射向量。

注意，这里的法线是经过 **法线贴图** 偏转后的法线，可能会导致 **反射向量** 指向模型表面的下方。 

这个时候我们还需要 **顶点法线**，**顶点法线** 可以做为模型正确的面向参考。

我们把 **反射向量** 和 **顶点法线** 做 **点乘**，如果是一个负值，就表明反射向量指向了表面下方，这个时候，我们就需要做反射光遮蔽了。

作者这里给了一个参数 **horizonFade** 来控制遮蔽强度，代码还是很简单的。

## 结尾

好了，**Horizon Occlusion** 就介绍到这里。

最后吹一下 [Advanced Terrain Grass](https://assetstore.unity.com/packages/tools/terrain/advanced-terrain-grass-100014?aid=1101l85Tr)，代码看得差不多了，还是有不少值得学习的，后面慢慢写，先留图一张，拜拜！


![img](/img/horizon-occlusion/screenshot4.png)






