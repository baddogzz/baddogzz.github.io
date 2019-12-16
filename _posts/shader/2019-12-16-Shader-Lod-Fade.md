---
layout: post
title: "关于unity_LODFade的骚操作"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
  - Shader
---

### 关于unity_LODFade的骚操作

玩 [The Illustrated Nature](https://assetstore.unity.com/packages/3d/vegetation/the-illustrated-nature-153939?aid=1101l85Tr)，发现他的植物做了Lod，并且Lod的过渡并不是硬切的，有一个渐变，如下：

![img](/img/shader-lod-fade/screenshot1.gif){:height="90%" width="90%"}

跟了一下代码，发现主要的逻辑如下：

```
#ifdef LOD_FADE_CROSSFADE
	LODDitheringTransition( IN.clipPos.xyz, unity_LODFade.x );
#endif
```

上述代码里的 **unity_LODFade** 是Unity内置变量，**unity_LODFade.x** 表示是 **当前LOD等级的权重**，Unity文档对其有一个简单说明：

> Unity doesn’t provide a default built-in technique to blend LOD geometries. You need to implement your own technique according to your game type and Asset production pipeline.

> Unity calculates a blend factor from the GameObject’s screen size and passes it to your shader
 in the unity_LODFade.x uniform variable. Depending on the Fade Mode you choose, use either the LOD_FADE_PERCENTAGE or LOD_FADE_CROSSFADE keyword for GameObjects rendered with LOD fading.

再来看一下 **LODDitheringTransition** 函数。因为我使用的是 **URP** 管线，直接下载 [源码](https://github.com/Unity-Technologies/ScriptableRenderPipeline/)，看一下 **LODDitheringTransition** 到底做了什么。

```
// LOD dithering transition helper
// LOD0 must use this function with ditherFactor 1..0
// LOD1 must use this function with ditherFactor -1..0
// This is what is provided by unity_LODFade
void LODDitheringTransition(uint2 fadeMaskSeed, float ditherFactor)
{
    // Generate a spatially varying pattern.
    // Unfortunately, varying the pattern with time confuses the TAA, increasing the amount of noise.
    float p = GenerateHashedRandomFloat(fadeMaskSeed);

    // This preserves the symmetry s.t. if LOD 0 has f = x, LOD 1 has f = -x.
    float f = ditherFactor - CopySign(p, ditherFactor);
    clip(f);
}
```

**LODDitheringTransition** 函数先对 **clipPos** 做了一个hash操作，再把hash的结果结合 **unity_LODFade.x** 做 **clip** 操作， 丢弃一些像素以达到 **平滑过渡** 的效果。

### 实现细节

#### 关于unity_LODFade.x的设置

关于LOD，如果作上面的 **平滑过渡** 效果，那么在过渡的阶段，存在2个LOD等级模型共存的情况，这里有一定的性能问题，并且 **clip** 操作在很多平台是很昂贵的。

我们姑且不考虑性能问题，**LODDitheringTransition** 的代码注释写的比Unity文档清楚的地方在于，他告诉了我们 **unity_LODFade.x** 到底是怎么设值的。

> LOD0 must use this function with ditherFactor 1..0 <br>
> LOD1 must use this function with ditherFactor -1..0 <br>
> This is what is provided by unity_LODFade

#### 关于GenerateHashedRandomFloat

GenerateHashedRandomFloat的代码如下：

```
float GenerateHashedRandomFloat(uint x)
{
    return ConstructFloat(JenkinsHash(x));
}

float GenerateHashedRandomFloat(uint2 v)
{
    return ConstructFloat(JenkinsHash(v));
}

float GenerateHashedRandomFloat(uint3 v)
{
    return ConstructFloat(JenkinsHash(v));
}

float GenerateHashedRandomFloat(uint4 v)
{
    return ConstructFloat(JenkinsHash(v));
}
```

这里首先对 **clipPos** 做一个简化的 [Jenkins hash](https://en.wikipedia.org/wiki/Jenkins_hash_function)，考虑1维的情况，算法如下：

```
// A single iteration of Bob Jenkins' One-At-A-Time hashing algorithm.
uint JenkinsHash(uint x)
{
    x += (x << 10u);
    x ^= (x >>  6u);
    x += (x <<  3u);
    x ^= (x >> 11u);
    x += (x << 15u);
    return x;
}
```

Hash完毕后，**ConstructFloat** 函数利用Hash的结果再生成一个 **[0,1)** 区间的浮点数用于最后的 **clip** 操作。

```
// Construct a float with half-open range [0, 1) using low 23 bits.
// All zeros yields 0, all ones yields the next smallest representable value below 1.
float ConstructFloat(int m) {
    const int ieeeMantissa = 0x007FFFFF; // Binary FP32 mantissa bitmask
    const int ieeeOne      = 0x3F800000; // 1.0 in FP32 IEEE

    m &= ieeeMantissa;                   // Keep only mantissa bits (fractional part)
    m |= ieeeOne;                        // Add fractional part to 1.0

    float  f = asfloat(m);               // Range [1, 2)
    return f - 1;                        // Range [0, 1)
}
```

考虑一下 **单精度浮点数的内存布局**

![img](/img/shader-lod-fade/screenshot2.png)

**ConstructFloat** 风骚的利用Hash结果的 **低23位** 做 **尾数**，和 **0x3F800000(浮点数1.0的int表示)** 做 **或操作**，再把得到的结果从int转为float，就得到了 **[1,2)** 区间的浮点数，再减去1.0f，就得到了**[0,1)** 区间的浮点数结果。






