---
layout: post
title: "Force Field效果的实现"
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

趁着有空，我做了一个类似的实现，丢到 **github** 上去了，地址：[https://github.com/fatdogsp/Unity-Force-Field-Effect](https://github.com/fatdogsp/Unity-Force-Field-Effect)。

这个效果不难做，我们只需要检测出护盾的 **边缘** 以及护盾和场景的 **交界**，并依此来调整护盾的 **透明度** 即可。

下图是我的第一版实现：

![img](/img/force-field/screenshot2.png)

和 **异界锁链** 相比，三角形背面的交界没了。

如果需要显示背面交界，我们首先需要 **Cull Off**，然后把背面的 **边缘强度** 设置为 **0**，只保留背面的 **交界强度**。

下图是我添加背面交界后的效果：

![img](/img/force-field/screenshot1.png)

好了，下面来看看实现细节。

## 透明度的计算公式

我们需要一个 **半透明** 的shader，**边缘** 和 **交界** 的透明度低，其他地方透明度高。

这里 **透明度** 的计算方式有参考 [Brackeys的教程](https://www.youtube.com/watch?v=NiOGWZXBg4Y)，有兴趣的可以先看一下视频。

整体流程用 **surface shader** 来写的话，代码如下：

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

这里涉及到如下三个函数：

+ ComputeDepthDelta：计算 **交界强度**。

+ ComputeFresnel：计算 **边缘强度**。

+ ComputePattern：计算 **整体强度** 的贴图效果和UV动画。

## ComputeDepthDelta

我们可以用 **当前像素的深度** 和 **背景像素的深度** 做比较，如果深度相距较近，则可以认为是 **交界**，代码如下：

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

这里提供一个 **_DepthOffset**，可以对 **深度距离** 做额外偏移，从而调整 **交界** 区域的大小。

## ComputeFresnel

关于 **边缘强度** 的计算，我们要区分是正面还是背面，背面的 **边缘强度** 永远为 **0**。

如果是 **fragment shader**，我们可以通过 **VFace** 来区分正面背面：

```
fixed4 frag (fixed facing : VFACE) : SV_Target
{
    // VFACE input positive for frontbaces,
    // negative for backfaces. Output one
    // of the two colors depending on that.
    return facing > 0 ? _ColorFront : _ColorBack;
}
```

可恼的是，**surface shader** 我没找到类似的设值，这里我通过 **dot(IN.viewDir, IN.worldNormal)** 的 **正负** 来判断正面背面。

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

## ComputePattern

计算出 **边缘强度** 和 **边界强度** 后，我们把这2个值相加，就可以得到 **整体透明度**。

这里我们还可以对 **整体透明度** 做一些贴图效果，比如下图：

![img](/img/force-field/screenshot3.png)

原理很简单，把 **ComputePattern** 的结果和 **整体透明度** 做 **乘法** 即可，**ComputePattern** 的代码如下：

```
inline half ComputePattern(Input IN)
{
    float2 uv = IN.uv_PatternTex + half2(_PatternOffsetU, _PatternOffsetV) * _Time.y * _ScrollSpeed;
    return tex2D(_PatternTex, uv).r;
}
```

## 结尾

只要明白原理，代码还是很简单的。

当然，这只是基础功能，如果需要更多特性，可以参考Unity商店的 [ForceField Effects](https://assetstore.unity.com/packages/vfx/particles/spells/forcefield-effects-123431?aid=1101l85Tr&utm_source=aff) 这个插件。

好了，拜拜！























































































































































