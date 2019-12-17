---
layout: post
title: "标准管线下的unity_LODFade操作"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
  - SRP
  - Shader
---

### 标准管线下的unity_LODFade操作

前文分析了 **URP** 管线下 **LODDitheringTransition** 的实现细节。

回到标准管线，LOD的 **Dither** 过渡有一个类似的实现：**UNITY_APPLY_DITHER_CROSSFADE**。

以 [Riko](https://assetstore.unity.com/packages/3d/characters/humanoids/riko-74357?aid=1101l85Tr) 为例，看一下 **UNITY_APPLY_DITHER_CROSSFADE** 的效果：

![img](/img/shader-lod-fade/screenshot6.gif){:height="80%" width="80%"}

shader主要添加如下2句即可：

```
#pragma multi_compile _ LOD_FADE_CROSSFADE
```

```
UNITY_APPLY_DITHER_CROSSFADE(i.pos.xy);
```

更具体的细节，可以参考 **github** 上的一个 [CrassFade示例工程](https://github.com/keijiro/CrossFadingLod)，它不但演示了 **Dither** 过渡，还演示了 **Fade** 过渡。

### 关于示例工程的Fade过渡和Dither过渡

因为比较简单，这里直接贴代码：

#### Fade过渡

![img](/img/shader-lod-fade/screenshot3.gif){:height="30%" width="30%"}

```
Shader "Custom/CrossFadingLod"
{
    Properties
    {
        _Color ("Color", Color) = (1,1,1,1)
        _MainTex ("Albedo (RGB)", 2D) = "white" {}
        _Glossiness ("Smoothness", Range(0,1)) = 0.5
        _Metallic ("Metallic", Range(0,1)) = 0.0
    }
    SubShader
    {
        Tags { "RenderType"="Transparent" "Queue"="Transparent" }
        LOD 200
        
        CGPROGRAM

        #pragma multi_compile _ LOD_FADE_CROSSFADE
        #pragma surface surf Standard alpha:fade
        #pragma target 3.0

        sampler2D _MainTex;

        struct Input
        {
            float2 uv_MainTex;
        };

        half _Glossiness;
        half _Metallic;
        fixed4 _Color;

        void surf(Input IN, inout SurfaceOutputStandard o)
        {
            // Albedo comes from a texture tinted by color
            fixed4 c = tex2D (_MainTex, IN.uv_MainTex) * _Color;
            o.Albedo = c.rgb;
            // Metallic and smoothness come from slider variables
            o.Metallic = _Metallic;
            o.Smoothness = _Glossiness;
#ifdef LOD_FADE_CROSSFADE
            o.Alpha = c.a * unity_LODFade.x;
#else
            o.Alpha = c.a;
#endif
        }
        ENDCG
    }
    FallBack "Transparent/Diffuse"
}
```

#### Dither过渡

![img](/img/shader-lod-fade/screenshot4.gif){:height="30%" width="30%"}

```
Shader "Custom/CrossFadingLod (Dither)"
{
    Properties
    {
        _Color ("Color", Color) = (1,1,1,1)
        _MainTex ("Albedo (RGB)", 2D) = "white" {}
        _Glossiness ("Smoothness", Range(0,1)) = 0.5
        _Metallic ("Metallic", Range(0,1)) = 0.0
    }
    SubShader
    {
        Tags { "RenderType"="TransparentCutout" "Queue"="AlphaTest" }
        LOD 200
        
        CGPROGRAM

        #pragma multi_compile _ LOD_FADE_CROSSFADE
        #pragma surface surf Standard
        #pragma target 3.0

        sampler2D _MainTex;

        struct Input
        {
            float4 screenPos;
            float2 uv_MainTex;
        };

        half _Glossiness;
        half _Metallic;
        fixed4 _Color;

        void surf(Input IN, inout SurfaceOutputStandard o)
        {
            #ifdef LOD_FADE_CROSSFADE
            float2 vpos = IN.screenPos.xy / IN.screenPos.w * _ScreenParams.xy;
            UnityApplyDitherCrossFade(vpos);
            #endif
            // Albedo comes from a texture tinted by color
            fixed4 c = tex2D (_MainTex, IN.uv_MainTex) * _Color;
            o.Albedo = c.rgb;
            // Metallic and smoothness come from slider variables
            o.Metallic = _Metallic;
            o.Smoothness = _Glossiness;
            o.Alpha = c.a;
        }
        ENDCG
    }
    FallBack "Diffuse"
}
```

---

### Dither过渡的实现细节

**Fade** 过渡比较简单，各个LOD等级按照 **unity_LODFade.x** 给定的权重半透即可。 

这里关注一下 **Dither** 过渡，**Dither** 过渡主要由 **UNITY_APPLY_DITHER_CROSSFADE** 完成，我们看一下源码：

```
#ifdef LOD_FADE_CROSSFADE
    #define UNITY_APPLY_DITHER_CROSSFADE(vpos)  UnityApplyDitherCrossFade(vpos)
    sampler2D unity_DitherMask;
    void UnityApplyDitherCrossFade(float2 vpos)
    {
        vpos /= 4; // the dither mask texture is 4x4
        float mask = tex2D(unity_DitherMask, vpos).a;
        float sgn = unity_LODFade.x > 0 ? 1.0f : -1.0f;
        clip(unity_LODFade.x - mask * sgn);
    }
#else
    #define UNITY_APPLY_DITHER_CROSSFADE(vpos)
#endif
```

这里的思路和 **LODDitheringTransition** 类似，不过 **clipPos** 没有进行 **hash** 计算，而是去一张预生成的 **4x4** 纹理 **unity_DitherMask** 去采样，得到的结果再结合 **unity_LODFade.x** 做 **像素裁剪**。

![img](/img/shader-lod-fade/screenshot5.png){:height="25%" width="25%"}

前文提到 **unity_LODFade.x** 有正有负，这里也有类似 **CopySign** 的计算，代码很简单，就不细说了。

---

### 关于Dither

[Dither](https://en.wikipedia.org/wiki/Dither)

[Floyd–Steinberg_dithering](https://en.wikipedia.org/wiki/Floyd%E2%80%93Steinberg_dithering)

![img](/img/shader-lod-fade/screenshot7.png){:height="75%" width="75%"}




