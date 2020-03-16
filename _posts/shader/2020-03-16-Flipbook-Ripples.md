---
layout: post
title: "用ShaderGraph的Flipbook节点实现水花效果"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
  - Shader
---

## 一个有趣的教程

昨天看了一个老外的视频教程：[Rain Drop Ripples](https://www.youtube.com/watch?v=R6EX6dN1BOs&t=818s)，教程用 **ShaderGraph** 实现了游戏中常见的下雨天地面的水花效果。

视频中水花动画的实现和我们游戏的实现方式有所不同，他通过 **ShaderGraph** 的 **Flipbook** 节点实现了水花的 **序列图** 动画：

![](/img/flipbook-ripples/screenshot1.png)

上图中的 **Flipbook** 节点对应的代码如下：

```
float2 _Flipbook_Invert = float2(FlipX, FlipY);

void Unity_Flipbook_float(float2 UV, float Width, float Height, float Tile, float2 Invert, out float2 Out)
{
    Tile = fmod(Tile, Width * Height);
    float2 tileCount = float2(1.0, 1.0) / float2(Width, Height);
    float tileY = abs(Invert.y * Height - (floor(Tile * tileCount.x) + Invert.y * 1));
    float tileX = abs(Invert.x * Width - ((Tile - Width * floor(Tile * tileCount.x)) + Invert.x * 1));
    Out = (UV + float2(tileX, tileY)) * tileCount;
}
```

代码比较好理解，根据当前动画的帧号 **Tile** 换算出对应的纹理坐标。

以下面这张 **4x4** 的水花序列图（需转为法帖）为例，传入 **Width = 4**，**Height = 4**，**Tile = _Time.x * AnimSpeed**，我们就能算出 **Out** UV。

![](/img/flipbook-ripples/screenshot2.png)

有了 **Out** UV后，我们解码水花法帖得到当前帧水花法线，再和地表的法线 **混合**，就大功告成了，如下图：

![](/img/flipbook-ripples/screenshot3.gif)

当然，这只是最简单的效果，下面我们看一下更多的细节。

## 更多细节

#### 关于UV的frac操作

从 **Unity_Flipbook_float** 的代码可知，传入的参数 **UV** 必须在 **[0,1]** 之间，而我们一般会对 **UV** 做 **Tiling** 操作，**Tiling** 的结果往往是超过1的。

这时，我们需要对 **Tiling** 的结果做一个 **frac** 操作，只保留小数部分，否则动画会抽风。

![](/img/flipbook-ripples/screenshot7.png)

#### 多层水花叠加和加入噪音

为了让水花更加自然，一般我们会根据 **Tiling** 不同至少做 **2层水花叠加**，并且会引入 **噪声** 让它的变化不那么规律。

这里作者通过 **Voronoi noise** 来控制两层水花的 **法线强度**：

![](/img/flipbook-ripples/screenshot4.png)

效果如下：

![](/img/flipbook-ripples/screenshot5.gif)

#### 混合水花和水流

单独有水花还是不够的，**水花配合水流** 才是王道。

水流的实现可以参考前文 [潮湿效果之水纹](https://baddogzz.github.io/2020/02/05/Wet-Waves/)。

此外，有的时候，美术并不希望水流有明显的流向，这个时候我们可以做2张水流贴图，让他UV动画对冲，示意代码如下：

```
half3 SGameWetBump(float3 worldPos, half3 worldNormal)
{
    half2 flowUV = half2(worldPos.x, worldPos.z);

    half3 flowBump1 = tex2D(_SG_WetBumpMap1, flowUV * _SG_WetBumpTilingOffset1.xy + _SG_WetBumpTilingOffset1.zw * _Time.y);
    half3 flowBump2 = tex2D(_SG_WetBumpMap2, flowUV * _SG_WetBumpTilingOffset2.xy + _SG_WetBumpTilingOffset2.zw * _Time.y);

    half3 flowBump = flowBump1 - flowBump2;
    half3 bumpNormal = lerp(half3(0,0,1), flowBump, _SG_WetFactor);

    return bumpNormal;
}    
```

这样，我们的水花只做一层，效果也能接受，如下图：

![](/img/flipbook-ripples/screenshot6.gif)

好了，拜拜！














































































































































































