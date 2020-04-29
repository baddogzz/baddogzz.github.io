---
layout: post
title: "关于Mali GPU的浮点数异常"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
  - Shader
---

## 一个华为手机上的Bug

今天查一个 **辉光抖动** 的问题：我们一个PBR的摩托车，在开辉光后高光处闪烁的厉害，并且这个闪烁只出现在华为手机上(Mali GPU)。

用RenderDoc分析了一下，闪烁处的高光值已经逆天了，如下图：

![](/img/mali-float-presion/screenshot1.png)

由上图可见，红框标记的颜色值达到了 **65504**，由于我们开启了 **FP16 HDR**，这里的 **65504** 刚好是 **FP16** 能表示的最大值。

![](/img/mali-float-presion/screenshot2.png)

> 0 11110 1111111111=(-1)^0 * 2^15 * (1+1-2^-10)=65504

直觉上这里是 **浮点数精度** 的问题，因为之前没少吃 **Mali GPU** 的亏，：）

## 修正

要堵这个问题很简单，只需要对最终的高光值用 **clamp大法** 即可。

不过作为一个强迫症患者，我还是想找到具体是哪里出了问题，于是做了一番调试，最后发现问题代码如下：

```
half perceptualRoughness = SmoothnessToPerceptualRoughness(smoothness);
half roughness = PerceptualRoughnessToRoughness(perceptualRoughness);

half V = SmithJointGGXVisibilityTerm(NoL, NoV, roughness); 
half D = GGXTerm(NoH, roughness);
half specularTerm = V * D * UNITY_PI;
```

这里 **PBR** 的高光项计算直接摘了Unity的 **BRDF1** 算法，去掉了 **菲涅尔项**，上述代码中 **roughness** 的 **精度** 影响了最终高光的计算结果。

我们看一下法线分布函数 **GGXTerm** 的代码：

```
inline float GGXTerm (float NdotH, float roughness)
{
    float a2 = roughness * roughness;
    float d = (NdotH * a2 - NdotH) * NdotH + 1.0f; // 2 mad
    return UNITY_INV_PI * a2 / (d * d + 1e-7f); 
    // This function is not intended to be running on Mobile,
    // therefore epsilon is smaller than what can be represented by half
}
```

参数都是 **float**，并且函数结尾有一个清楚的注释，说这个函数没打算在移动设备上跑，因为这里 **1e-7f** 并没考虑兼容 **half** 的精度：

> This function is not intended to be running on Mobile, therefore epsilon is smaller than what can be represented by half

半精度浮点数能表示的最小值为 **6.10×10^(-5)**：

> 0 00001 0000000000=2^-14 = 6.10*10^-5

所以把 **roughness** 的精度从 **half** 变成 **float**，这个问题也就修正了。

## URP管线对BRDF的简化

在移动设备直接用 **Standard管线** 的 **BRDF1** 算法，计算量会略高。

这里我们也可以参考 **BRDF2** 的写法，或者参考 **URP管线** 对于 **DirectBDRF** 的简化方式，代码如下：

```
// Based on Minimalist CookTorrance BRDF
// Implementation is slightly different from original derivation: http://www.thetenthplanet.de/archives/255
//
// * NDF [Modified] GGX  
// * Modified Kelemen and Szirmay-Kalos for Visibility term
// * Fresnel approximated with 1/LdotH 
half3 DirectBDRF(BRDFData brdfData, half3 normalWS, half3 lightDirectionWS, half3 viewDirectionWS)
{
#ifndef _SPECULARHIGHLIGHTS_OFF
    float3 halfDir = SafeNormalize(float3(lightDirectionWS) + float3(viewDirectionWS)); 

    float NoH = saturate(dot(normalWS, halfDir));
    half LoH = saturate(dot(lightDirectionWS, halfDir));

    // GGX Distribution multiplied by combined approximation of Visibility and Fresnel 
    // BRDFspec = (D * V * F) / 4.0
    // D = roughness^2 / ( NoH^2 * (roughness^2 - 1) + 1 )^2 
    // V * F = 1.0 / ( LoH^2 * (roughness + 0.5) )
    // See "Optimizing PBR for Mobile" from Siggraph 2015 moving mobile graphics course
    // https://community.arm.com/events/1155

    // Final BRDFspec = roughness^2 / ( NoH^2 * (roughness^2 - 1) + 1 )^2 * (LoH^2 * (roughness + 0.5) * 4.0)
    // We further optimize a few light invariant terms 
    // brdfData.normalizationTerm = (roughness + 0.5) * 4.0 rewritten as roughness * 4.0 + 2.0 to a fit a MAD. 
    float d = NoH * NoH * brdfData.roughness2MinusOne + 1.00001f;

    half LoH2 = LoH * LoH;    
    half specularTerm = brdfData.roughness2 / ((d * d) * max(0.1h, LoH2) * brdfData.normalizationTerm);

    // On platforms where half actually means something, the denominator has a risk of overflow
    // clamp below was added specifically to "fix" that, but dx compiler (we convert bytecode to metal/gles)
    // sees that specularTerm have only non-negative terms, so it skips max(0,..) in clamp (leaving only min(100,...))
#if defined (SHADER_API_MOBILE) || defined (SHADER_API_SWITCH)
    specularTerm = specularTerm - HALF_MIN;
    specularTerm = clamp(specularTerm, 0.0, 100.0); // Prevent FP16 overflow on mobiles
#endif

    half3 color = specularTerm * brdfData.specular + brdfData.diffuse;
    return color;
#else
    return brdfData.diffuse;  
#endif
}
```

代码注释写得很清楚，简化方式参考了 [SIGGRAPH 2015](https://community.arm.com/events/1155) 之 **Optimizing PBR for Mobile**。

经典的微表面高光 **BRDF** 公式如下：

![](/img/mali-float-presion/screenshot3.png)

按照 **Optimizing PBR for Mobile** 的方式，可以对 **V * F** 合并和近似：

> BRDFspec = (D * V * F) / 4.0

> D = roughness^2 / ( NoH^2 * (roughness^2 - 1) + 1 )^2 

> V * F = 1.0 / ( LoH^2 * (roughness + 0.5) )

最终结果如下：

![](/img/mali-float-presion/screenshot4.png)

最后，上面的代码也兼顾了 half 的精度：

```
#define HALF_MIN 6.103515625e-5  // 2^-14, the same value for 10, 11 and 16-bit: https://www.khronos.org/opengl/wiki/Small_Float_Formats

// On platforms where half actually means something, the denominator has a risk of overflow
// clamp below was added specifically to "fix" that, but dx compiler (we convert bytecode to metal/gles)
// sees that specularTerm have only non-negative terms, so it skips max(0,..) in clamp (leaving only min(100,...))

#if defined (SHADER_API_MOBILE) || defined (SHADER_API_SWITCH)
    specularTerm = specularTerm - HALF_MIN;
    specularTerm = clamp(specularTerm, 0.0, 100.0); // Prevent FP16 overflow on mobiles
#endif
```

好了，拜拜。











































































































































































































































