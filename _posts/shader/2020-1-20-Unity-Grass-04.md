---
layout: post
title: "移动端草海的渲染方案（四）"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
  - Shader
  - 小甜甜
---

## 书接上文

前文介绍了我们游戏草海的基本渲染方案，本文将介绍一些更加有趣的动态效果。

## 碰撞弯曲

**碰撞弯曲** 在手游中比较常见，下图是我们游戏里的效果：

![img](/img/unity-grass4/screenshot1.gif)

[uNature](https://assetstore.unity.com/packages/vfx/shaders/unature-gpu-grass-and-interactable-trees-43129?aid=1101l85Tr) 的实现方式 **类似用一个球体压迫草面，使得草的顶点远离球体中心**。

[uNature](https://assetstore.unity.com/packages/vfx/shaders/unature-gpu-grass-and-interactable-trees-43129?aid=1101l85Tr) 一共支持 **20个碰撞源**，不过手游的话只计算 **主角的碰撞** 就够了。

我们可以在 **顶点着色器** 中添加如下代码：

```
inline float4 CalculateTouchBending(float4 vertex)
{
    float4 current = _InteractionTouchBendedInstances;

    if (distance(vertex.xyz, current.xyz) < current.w)
    {                
        float WMDistance = 1 - clamp(distance(vertex.xyz, current.xyz) / current.w, 0, 1);
        float3 posDifferences = normalize(vertex.xyz - current.xyz);

        float3 strengthedDifferences = posDifferences * (_TouchBendingStrength + _TouchBendingStrength);

        float3 resultXZ = WMDistance * strengthedDifferences;

        vertex.xz += resultXZ.xz;
        vertex.y -= WMDistance * _TouchBendingStrength;

        return vertex;
    }

    return vertex;
}
```

上面代码中的 **_InteractionTouchBendedInstances** 存贮的是 **碰撞源** 的信息，其 **xyz** 分量是 **球心的世界坐标**， **w** 分量是球的 **半径**，**_InteractionTouchBendedInstances** 的值由脚本传入。

这里的代码很简单，不废话。

## 割草

**割草** 是另外一个好玩的东西，自从玩过 **塞尔达**，看到什么都想砍两刀，：）

下图是我们游戏的割草效果：

![img](/img/unity-grass4/screenshot3.gif)

**割草** 的原理也很简单：在运行时修改 **草的密度**。

前文提过，Unity内置的 **Terrain** 提供了一个 **GetDetailLayer** 接口，这个接口返回一个二维数组，这个二维数组和 **栅格化** 后的地表网格是相对应的，数组每个元素的值即当前格 **草的密度**。

[uNature](https://assetstore.unity.com/packages/vfx/shaders/unature-gpu-grass-and-interactable-trees-43129?aid=1101l85Tr) 的实现方式和 **Terrain** 类似，场景依然会被 **栅格化**，不同的是，**uNature** 用一张纹理 **GrassMap** 来存贮密度。

**割草** 主要的逻辑就是计算出 **攻击范围** 覆盖的单元格，并且修改这些单元格的密度。

常见的技能攻击形状包括：圆形攻击，扇形攻击，矩形攻击等。

下图是一个 **扇形攻击** 覆盖的单元格示意：

![img](/img/unity-grass4/screenshot2.png)

趁着年前有空，我把 **割草** 的代码整理了一翻，丢到 **Asset Store** 赚点小钱，**Terrain版本** 和 **uNature版本** 的割草都有：

+ [Easy Grass Cutter](https://assetstore.unity.com/packages/tools/particles-effects/easy-grass-cutter-156255?aid=1101l85Tr&utm_source=aff)

+ [uNature Cutter](https://assetstore.unity.com/packages/tools/integration/unature-cutter-156603?aid=1101l85Tr&utm_source=aff)

欢迎参考。

## 烧草

**塞尔达** 的草不但可以割，还可以烧。

下图是我们模仿 **塞尔达** 的烧草效果，包括火势蔓延，：）

![img](/img/unity-grass4/screenshot4.gif)

原理上，**烧草** 和 **割草** 类似，不同的是，割草需要 **密度**，而烧草需要 **燃烧度**。

这里的 **燃烧度** 是我们自己定义的，就是 **草发黑的程度**。

我们当然可以模仿 **密度图** 生成一张 **燃烧度图**，不过从优化资源占用的角度考虑，我们最终的做法是：**复用密度图**。

我们可以对 **密度** 做一个编码，让他 **即能表示密度，也能表示燃烧度**。

比如 **密度** 的最大值我们定为 **10**，**燃烧度** 的最大值我们定为 **23**，**10 * 23 + 22 = 252** 还是在 **255以内**，这样 **GrassMap** 的一个通道就能保存这个信息，表现上也足够。

举个例子，比如当前格的密度是 **3**，燃烧度是 **12**，那么编码后的值就是 **3 * 23 + 12 = 81**。

这样，无论是 **割草** 还是 **烧草**，我们要做的都是 **更新密度**，即更新 **GrassMap** 的 **像素信息**：

```
public Color32[] mapPixels
{
    get
    {
        if (_mapPixels == null)
        {
            _mapPixels = map.GetPixels32();
        }

        return _mapPixels;
    }

    internal set
    {
        _mapPixels = value;
    }
}
```

小贴士：用 **Texture2D.GetPixels32** 接口和 **Color32** 来计算，速度更快，：）

> For most textures, even faster is to use GetPixels32 which returns low precision color data without costly integer-to-float conversions.


## 随风摆动

**uNature** 草的摆动算法如下：

```
void FastSinCos(float4 val, out float4 s, out float4 c) 
{
    val = val * 6.408849 - 3.1415927;
    float4 r5 = val * val;
    float4 r6 = r5 * r5;
    float4 r7 = r6 * r5;
    float4 r8 = r6 * r5;
    float4 r1 = r5 * val;
    float4 r2 = r1 * r5;
    float4 r3 = r2 * r5;
    float4 sin7 = { 1, -0.16161616, 0.0083333, -0.00019841 };
    float4 cos8 = { -0.5, 0.041666666, -0.0013888889, 0.000024801587 };
    s = val + r1 * sin7.y + r2 * sin7.z + r3 * sin7.w;
    c = 1 + r5 * cos8.x + r6 * cos8.y + r7 * cos8.z + r8 * cos8.w;
}

float4 ApplyFastWind(float4 vertex, float texCoordY)
{
    if (_WindSpeed == 0) return vertex;

    float speed = _WindSpeed;

    float4 _waveXmove = float4 (0.024, 0.04, -0.12, 0.096);
    float4 _waveZmove = float4 (0.006, .02, -0.02, 0.1);

    const float4 waveSpeed = float4 (1.2, 2, 1.6, 4.8);

    float4 waves;
    waves = vertex.x * _WindBending;
    waves += vertex.z * _WindBending;

    waves += _Time.x * (1 - 0.4) * waveSpeed * speed;

    float4 s, c;
    waves = frac(waves);
    FastSinCos(waves, s, c);

    float waveAmount = texCoordY * (1 + 0.4);
    s *= waveAmount;

    s *= normalize(waveSpeed);

    s = s * s;
    float fade = dot(s, 1.3);
    s = s * s;

    float3 waveMove = float3 (0, 0, 0);
    waveMove.x = dot(s, _waveXmove);
    waveMove.z = dot(s, _waveZmove);

    vertex.xz -= waveMove.xz;

    //vertex -= mul(_World2Object, float3(_WindSpeed, 0, _WindSpeed)).x * _WindBending * _SinTime;

    return vertex;
}
```

上面的代码和Unity内置的 **WavingGrass** 版本差不多，也是自己摆自己的，不受 **WindZone** 影响。

我们项目并未采用上述算法，而是选择了 [Lux LWRP Essentials](https://assetstore.unity.com/packages/vfx/shaders/lux-lwrp-essentials-150355?aid=1101l85Tr&utm_source=aff) 的方案，请参考前文 [Lux的风和WindTexture](https://baddogzz.github.io/2019/12/06/Lux-Wind-Texture/)，这里略过。

需要注意的是，添加了 **碰撞弯曲** 后，我们应该先计算 **随风摆动**，后计算 **碰撞弯曲**。

```
v.vertex = CalculateTouchBending(v.vertex);
v.vertex = ApplyFastWind(v.vertex, v.texcoord.y);
```

如果颠倒，风会把草吹到 **碰撞体** 的肚子里去，：）

## 一个浮点数比较的Bug

最后说一个 [uNature](https://assetstore.unity.com/packages/vfx/shaders/unature-gpu-grass-and-interactable-trees-43129?aid=1101l85Tr) 的bug。

第一次在手机上跑 **uNature** 的时候，我发现部分 **Mali GPU** 的手机 **草不见了**，追了半天，最后发现是下面这个函数导致的：

```
float GetDensity(float4 pixel)
{
    pixel *= 255;
    if (pixel.r == _PrototypeID) return pixel.g;
    if (pixel.b == _PrototypeID) return pixel.a;

    return 0;
}
```

这个函数根据 **GrassMap** 获取 **指定类型草的密度**，这里的 **_PrototypeID** 即 **草的类型ID**。

这里关于ID的 **浮点数比较** 直接用了 **==**，违背了老师对我们的教导，十分的可疑。

换成下面的写法后，这个bug就修正了：

```
float GetDensity(float4 pixel)
{
    pixel *= 255;

    if(abs(pixel.r - _PrototypeID) < 0.01f) return pixel.g;
    if(abs(pixel.b - _PrototypeID) < 0.01f) return pixel.a;

    return 0;
}
```

好了，本文就到这里。

下篇文章会介绍提升草的 **光照表现** 的一些做法，拜拜！
























































































