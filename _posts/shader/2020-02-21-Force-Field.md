---
layout: post
title: "Force Field"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
  - Shader
---

## 斧式雷基恩的护盾效果

下图是 **异界锁链** 斧式雷基恩的护盾效果：

![img](/img/force-field/screenshot0.jpg)

我做了一个类似的实现，丢到 **github** 上去了，地址：[https://github.com/fatdogsp/Unity-Force-Field-Effect](https://github.com/fatdogsp/Unity-Force-Field-Effect)。

这个效果不难做，主要代码包括：

+ 检测护盾的边缘

+ 检测护盾和场景的交界

**边缘** 和 **交界** 共同决定了护盾的 **透明度**。

下图是我的第一版实现：

![img](/img/force-field/screenshot2.png)

和 **异界锁链** 相比，三角形背面的交界没了。

如果需要显示背面交界，我们首先需要 **Cull Off**，然后背面的边缘我们无需计算，下图是我添加背面交界后的效果：

![img](/img/force-field/screenshot1.png)

好了，下面来看看实现细节。

## 透明度的计算公式

我们需要一个 **半透明** 的shader，**边缘** 和 **交界** 处的透明度低，其他地方透明度高。

这里 **透明度** 的计算方式有参考 [Brackeys的教程](https://www.youtube.com/watch?v=NiOGWZXBg4Y)，有兴趣的可以先看一下视频。

用 **surface shader** 来写的话，大概的代码如下：

```
void surf (Input IN, inout SurfaceOutputStandard o)
{
    half alpha = ForceFieldAlpha(IN, o);

    o.Albedo = half3(0, 0, 0);
    o.Metallic = _Metallic;
    o.Smoothness = _Smoothness;
    o.Occlusion = 1;
    o.Emission = _EmissionColor.rgb;
    o.Alpha = alpha;
}
```

**ForceFieldAlpha** 函数主要用来计算 **透明度**，其他都是常规操作，颜色主要依靠 **自发光**。

**ForceFieldAlpha** 代码如下：

```
half ForceFieldAlpha(Input IN, inout SurfaceOutputStandard o)
{
    half depthDelta = ComputeDepthDelta(IN);
    half fresnel = ComputeFresnel(IN);
    half pattern = ComputePattern(IN);

    return (depthDelta + fresnel) * pattern * _AlphaStrength;
}
```

涉及到的三个函数如下：

+ ComputeDepthDelta：计算 **交界** 强度。

+ ComputeFresnel：计算 **边缘** 强度。

+ ComputePattern：采样透明度贴图，有了它，我们就能实现各种形状，并且做UV动画。

## 计算交界

我们可以用 **当前像素的深度** 和 **背景像素的深度** 做比较，深度相距较近，则可以认为是 **交界**，代码如下：

```
inline half ComputeDepthDelta(Input IN)
{
    float2 screenUV = IN.screenPos.xy / IN.screenPos.w;
    float screenDepth = LinearEyeDepth(SAMPLE_DEPTH_TEXTURE(_CameraDepthTexture, screenUV));

    float depth = IN.screenPos.w - _DepthOffset;
    float depthDelta = abs(screenDepth - depth);
    depthDelta = 1 - depthDelta;

    return smoothstep(0, 1, depthDelta);
}
```

这里提供一个 **_DepthOffset**，可以做一些额外的调节。

## 计算边缘

关于边缘的计算，我们要区分是正面还是背面，如果是 **fragment shader**，我们可以通过 **VFace** 来区分：

```
fixed4 frag (fixed facing : VFACE) : SV_Target
{
    // VFACE input positive for frontbaces,
    // negative for backfaces. Output one
    // of the two colors depending on that.
    return facing > 0 ? _ColorFront : _ColorBack;
}
```

可恼的是，**surface shader** 我没找到类似的设置，所以我只能通过 **dot(IN.viewDir, IN.worldNormal)** 的正负来判断正面背面了。

当然，这样判断有一定的瑕疵，具体可以参考 [这篇帖子](https://forum.unity.com/threads/using-vface-in-surface-shader.460941/)，不过做为示例，这样已经OK了，下面贴代码：

#### 只考虑正面

```
Cull Back
```

```
inline half ComputeFresnel(Input IN)
{
    half rim = 1 - saturate(dot(IN.viewDir, IN.worldNormal));
    return pow(rim, _FresnelPower);
}
```

#### 正面背面都考虑

```
Cull Off
```

```
inline half ComputeFresnel(Input IN)
{
    half rim = dot(IN.viewDir, IN.worldNormal);
    rim = lerp(1 - saturate(rim), 0, rim < 0);
    return pow(rim, _FresnelPower);
}
```

这里如果是背面，我们的边缘强度为 **0** 即可。

## ComputePattern

计算出 **边缘** 和 **边界** 后，我们把这2个值相加，就可以得到整体的 **透明度**。

这里我们还可以对 **透明度** 做一些贴图效果，比如下图：

![img](/img/force-field/screenshot3.png)

原理很简单，把 **ComputePattern** 的结果和整体 **透明度** 做 **乘法** 即可，**ComputePattern** 的代码如下：

```
inline half ComputePattern(Input IN)
{
    float2 uv = IN.uv_PatternTex + half2(_PatternOffsetU, _PatternOffsetV) * _Time.y * _ScrollSpeed;
    return tex2D(_PatternTex, uv).r;
}
```

## 结尾

实现细节就介绍到这里。

当然，这只是基础功能，如果需要更多特性，可以参考Unity商店的 [ForceField Effects](https://assetstore.unity.com/packages/vfx/particles/spells/forcefield-effects-123431?aid=1101l85Tr&utm_source=aff) 这个插件。

好了，拜拜！























































































































































