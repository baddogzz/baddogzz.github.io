---
layout: post
title: "移动端草海的渲染方案（五）"
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

前文介绍了草海的一些有趣的动态效果，本文是关于 **提升光照表现** 的一些做法。

Unity内置草的光照表现普普通通，经常会成为一些插件作者嘲讽的对象，下图是 **Unity** 和 [Advanced Terrain Grass](https://assetstore.unity.com/packages/tools/terrain/advanced-terrain-grass-100014?aid=1101l85Tr) 的草海对比：

*Unity的草*

![img](/img/unity-grass5/screenshot1.png) 

*ATG的草*

![img](/img/unity-grass5/screenshot2.png) 

下面就介绍一下提升草海光照表现的一些做法。

## 添加高光

Unity内置的草是没有 **高光** 计算的，如果没有影子，就会非常的平。

当然，即便没有高光，依然能做出非常漂亮的效果，比如大家可以参考一下这个场景 [The Illustrated Nature](https://assetstore.unity.com/packages/3d/vegetation/the-illustrated-nature-153939?aid=1101l85Tr) 的做法。

不过，我还是喜欢高光。

#### 我们的高光

我们游戏场景的光照还是传统的 **Blinn-Phong** 光照模型，为了模仿 **塞尔达** 草海的高光，我们会把草的法线 **全部向上**，草的 **光滑度** 可以控制高光的整体范围，再加上 **法线贴图** 就更好了。

此外，为了让水平视角下的高光 **更远离脚底** 从而更多的出现在镜头内，我们会对高光的位置做一定的 **偏移**，如下图：

![img](/img/unity-grass/screenshot2.png) 

最后，即便是 **Blinn-Phong** 光照模型，转到 **线性空间** 也比 **Gamma空间** 更容易出效果。

我们游戏的高光我是满意的，下面介绍一些其他做法。

#### 楚留香的高光

**楚留香** 的代码一直是我参考的对象，下面看一下 **楚留香** 的草。

*雨天*

![img](/img/unity-grass5/screenshot3.png) 

*月光*

![img](/img/unity-grass5/screenshot4.png) 

上图是楚留香 **草的高光** 截图，整体氛围还是很好的，不过第一张 **潮湿草的高光** 有点失真了。

**楚留香** 的渲染是全 **PBR** 的，草的高光部分做了相当程度的简化，没有 **IBL**，**BRDF** 也简化了，**直接高光** 的计算如下：

```
half3 sunSpec=half3(0,0,0);

float3 H=normalize(((V)+(L)));
float NoH=saturate(dot(N,H));
float D=((((((((NoH)*(m2)))-(NoH)))*(NoH)))+(1));
(D)=(((((D)*(D)))+(1e-06)));
(D)=(((((0.25)*(m2)))/(D)));
(sunSpec)=(((SpecularColor)*(D)));
        
(sunSpec)*=(((SunColor.rgb)*(saturate(((NoL)*(shadow))))));
```

大家可以一试。

#### ATG的高光

楚留香 **草的高光计算** 是简化的 **BRDF**，比如 **菲涅尔项** 直接省略掉了，当然这个效果手机上已经足够了。

回到Unity引擎，[Advanced Terrain Grass](https://assetstore.unity.com/packages/tools/terrain/advanced-terrain-grass-100014?aid=1101l85Tr) 的高光基本采用了Unity内置的 **BRDF1** 算法，也就是最高效果的 **BRDF**，如下图：

![img](/img/unity-grass5/screenshot5.png) 

此外，**ATG** 还增加了 **Light Scattering**，草叶变得更加通透，如下图：

![img](/img/unity-grass5/screenshot6.png) 

这里关于 **Light Scattering** 的计算，可以参考 [这篇文章](https://www.slideshare.net/colinbb/colin-barrebrisebois-gdc-2011-approximating-translucency-for-a-fast-cheap-and-convincing-subsurfacescattering-look-7170855)，这个算法计算量不大，也可以用它来模拟 **次表面散射** 的效果。

作者代码里还给出了 [一个参考链接](https://colinbarrebrisebois.com/2012/04/09/approximating-translucency-revisited-with-simplified-spherical-gaussian/)，是关于 **优化pow** 的，很有意思，值得一看。
 
这里直接附上代码： 

```
half4 c = BRDF1_ATG_PBS (s.Albedo, s.Specular, oneMinusReflectivity, s.Smoothness, /*NdotLDirect, */s.Normal, viewDir, gi.light, gi.indirect, specularIntensity);

//  For gi lighting we simply use the built in BRDF
c.rgb += UNITY_BRDF_GI (s.Albedo, s.Specular, oneMinusReflectivity, s.Smoothness, s.Normal, viewDir, s.Occlusion, gi);

//  Add Translucency – needs light dir and intensity: so real time only
#if !defined(LIGHTMAP_ON)
    //  Best for grass as the normal counts less
    //  //  https://colinbarrebrisebois.com/2012/04/09/approximating-translucency-revisited-with-simplified-spherical-gaussian/
    half3 transLightDir = gi.light.dir + s.Normal * 0.01;
    half transDot = dot( -transLightDir, viewDir ); // sign(minus) comes from eyeVec
    transDot = exp2(saturate(transDot) * s.TranslucencyPower - s.TranslucencyPower);
    half3 lightScattering = transDot * gi.light.color * 
    #if !defined(ISGRASS)
        (1.0 - NdotLDirect)
    #else
        1.0
    #endif
    ;
    c.rgb += s.Albedo * 4.0 * s.Translucency * lightScattering /* mask trans by spec */  * (1.0 - saturate(c.a));
#endif
```

## 添加AO 

卧槽，下班了！


































































































