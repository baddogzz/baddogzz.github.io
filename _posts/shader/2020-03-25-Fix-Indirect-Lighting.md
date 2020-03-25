---
layout: post
title: "固定角色的间接光照"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
  - Shader
---

## 一个美术需求

今天美术大哥要我把角色的间接光照固定，不受场景影响，即把下图的 **环境光** 和 **环境反射** 从 **Lighting Settings** 面板搬到 **角色材质球** 上：

![](/img/fix-indirect-lighting/screenshot1.png)

这种环境光的反差，也是突出角色的方法之一。

## 固定环境光

美术的要求是，无论场景如何布置，程序只认下图这个预设的环境光：

![](/img/fix-indirect-lighting/screenshot2.png)

这里的做法很简单，打开美术预烘培好的场景，用 **FrameDebugger** 截取 **球谐光照** 的参数值，然后在任何情况下都用这些值计算环境光即可：

![](/img/fix-indirect-lighting/screenshot3.png)

把上图中的 **unity_SHXXX** 带入下面的函数就大功告成了：

```
// normal should be normalized, w=1.0
half3 SHEvalLinearL0L1 (half4 normal)
{
    half3 x;

    // Linear (L1) + constant (L0) polynomial terms
    x.r = dot(unity_SHAr,normal);
    x.g = dot(unity_SHAg,normal);
    x.b = dot(unity_SHAb,normal);

    return x;
}

// normal should be normalized, w=1.0
half3 SHEvalLinearL2 (half4 normal)
{
    half3 x1, x2;
    // 4 of the quadratic (L2) polynomials
    half4 vB = normal.xyzz * normal.yzzx;
    x1.r = dot(unity_SHBr,vB);
    x1.g = dot(unity_SHBg,vB);
    x1.b = dot(unity_SHBb,vB);

    // Final (5th) quadratic (L2) polynomial
    half vC = normal.x*normal.x - normal.y*normal.y;
    x2 = unity_SHC.rgb * vC;

    return x1 + x2;
}

// normal should be normalized, w=1.0
// output in active color space
half3 ShadeSH9 (half4 normal)
{
    // Linear + constant polynomial terms
    half3 res = SHEvalLinearL0L1 (normal);

    // Quadratic polynomials
    res += SHEvalLinearL2 (normal);

#   ifdef UNITY_COLORSPACE_GAMMA
        res = LinearToGammaSpace (res);
#   endif

    return res;
}
```

当然，这样做之后，场景的 **光照探头** 对角色就失效了，不过美术要的就是这个效果......

## 固定环境反射

固定环境光，只需要在shader里写死参数，美术不需要做任何材质设置。

下面开始固定环境反射。

首先，把用于环境反射的 **Cubemap** 从 **Lighting Settings** 面板搬到 **角色材质球** 上，如下图：

![](/img/fix-indirect-lighting/screenshot4.png)

然后，间接高光的计算直接认角色材质指定的环境，即上图中的 **Env Cubemap**，主要代码如下：

```
half envRoughness = perceptualRoughness * (1.7 - 0.7 * perceptualRoughness);
half envMip = envRoughness * UNITY_SPECCUBE_LOD_STEPS;
half4 envColor = texCUBElod(_EnvCubemap, half4(R, envMip)) * _EnvReflectStrength;
envColor.rgb = DecodeHDR(envColor, _EnvCubemap_HDR).rgb;
```

用上面代码的计算结果 **envColor** 替代Unity全局光照的 **gi.indirect.specular** 就大功告成了。

最后，需要注意的是，环境图是需要经过 **烘培** 的，不要直接把 **Skybox** 指给角色材质。

## 固定尼玛

好了，你不是在做小甜甜。

还好，哥的新地板快写好了，舒服一下：

![](/img/fix-indirect-lighting/screenshot5.png)

拜拜。













































































































































































