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

Unity内置草的光照表现普普通通，经常会成为一些插件作者嘲讽的对象，下图是 **Unity** 和 [Advanced Terrain Grass](https://assetstore.unity.com/packages/tools/terrain/advanced-terrain-grass-100014?aid=1101l85Tr) 草海的光照对比：

*Unity的草*

![img](/img/unity-grass5/screenshot1.png) 

*ATG的草*

![img](/img/unity-grass5/screenshot2.png) 

没有对比就没有伤害，下面介绍一下提升 **光照表现** 的一些做法。

## 添加高光

Unity内置的草是没有 **高光** 的，如果没有影子，就显得没有立体感，非常的平。

当然，即便没有 **高光**，依然能做出非常漂亮的草海效果，比如大家可以参考一下这个场景 [The Illustrated Nature](https://assetstore.unity.com/packages/3d/vegetation/the-illustrated-nature-153939?aid=1101l85Tr) 的做法： **纯色贴图 + Lambert漫反射 + Color Grading**。

![img](/img/unity-grass5/screenshot11.jpg) 

**风格化** 其实蛮难做的，另外，我喜欢 **高光**。

#### 我们的高光

我们游戏场景的光照还是传统的 **Blinn-Phong** 光照模型，为了模仿 **塞尔达** 草海的高光，我们会把草的法线 **全部向上**，草的 **光滑度** 可以控制高光的整体范围，再加上 **法线贴图** 表现就很好了。

此外，为了让水平视角下的高光 **更远离脚底** 从而更多的 **出现在镜头** 内，我们会对高光的位置做一定的 **偏移**，如下图：

![img](/img/unity-grass/screenshot2.png) 

题外话，即便是 **Blinn-Phong** 光照模型，转到 **线性空间** 也比 **Gamma空间** 更容易出效果。

上图的高光我是满意的，下面介绍一些其他做法。

#### 楚留香的高光

**楚留香** 的代码一直是我参考的对象，下面看一下楚留香 **草的高光**。

*雨天*

![img](/img/unity-grass5/screenshot3.png) 

*月光*

![img](/img/unity-grass5/screenshot4.png) 

**楚留香** 场景的水准还是非常高的，不过就草的高光而言，可能是 **面片草** 没有 **法线贴图** 的缘故，表现不是太好。

追了一下代码，**楚留香** 的渲染是全 **PBR** 的，草的高光计算对 **BRDF公式** 做了相当程度的简化，代码如下：

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

回到Unity引擎，大部分草的插件都支持 **PBR**，以 [Advanced Terrain Grass](https://assetstore.unity.com/packages/tools/terrain/advanced-terrain-grass-100014?aid=1101l85Tr) 为例，**ATG** 的高光基本沿用了Unity内置的 **BRDF1** 算法，也就是最高效果的 **BRDF**，如下图：

![img](/img/unity-grass5/screenshot5.png) 

此外，**ATG** 还增加了 **Light Scattering**，草叶变得更加通透，如下图：

![img](/img/unity-grass5/screenshot6.png) 

关于 **Light Scattering** 的计算方式，可以参考 [这篇文章](https://www.slideshare.net/colinbb/colin-barrebrisebois-gdc-2011-approximating-translucency-for-a-fast-cheap-and-convincing-subsurfacescattering-look-7170855)，这个算法的计算量不大，也可以用它来模拟 **次表面散射**。

![img](/img/unity-grass5/screenshot12.png) 

此外，作者根据 [这篇文章](https://colinbarrebrisebois.com/2012/04/09/approximating-translucency-revisited-with-simplified-spherical-gaussian/) 对上面的算法做了优化，主要的优化点是关于 **pow** 的，很有意思，值得一看。
 
这里直接附上代码： 

```
//  Best for grass as the normal counts less
//  https://colinbarrebrisebois.com/2012/04/09/approximating-translucency-revisited-with-simplified-spherical-gaussian/
half3 transLightDir = gi.light.dir + s.Normal * 0.01;
half transDot = dot( -transLightDir, viewDir ); // sign(minus) comes from eyeVec
transDot = exp2(saturate(transDot) * s.TranslucencyPower - s.TranslucencyPower);
half3 lightScattering = transDot * gi.light.color;
c.rgb += s.Albedo * 4.0 * s.Translucency * lightScattering /* mask trans by spec */  * (1.0 - saturate(c.a));
```

最后，无论是 **Blinn-Phong** 还是 **PBR**，我们都可以通过调整 **光滑度**，**金属度**，**漫反射颜色**，**高光颜色** 来实现特定的效果需求。

下图是我模拟 **雨天湿滑** 的效果，程序员的审美，将就看，：）

![img](/img/unity-grass5/screenshot13.png) 

## 添加AO

各个插件对草的光照改进，主要还是 **高光** 部分。

此外，不开 **自阴影** 的前提下，**环境光遮蔽（AO）** 对草海整体表现的提升也是很明显的。

关于 **AO** 有几种做法。

**烘培的AO** 对于 **非静态的** 的草来说有点不合适，**后处理的AO** 则过于昂贵，对于手游来说更不合适。

我们可以通过 **顶点色** 来记录 **AO强度**，或者直接草的贴图增加一个 **AO通道**，性价比还是很高的。

以 [Lux LWRP Essentials](https://assetstore.unity.com/packages/vfx/shaders/lux-lwrp-essentials-150355?aid=1101l85Tr) 的草为例，我把 **阴影** 和 **AO** 全都关闭，效果如下：

![img](/img/unity-grass5/screenshot7.png) 

很平有没有？

如果打开 **AO**，但是依然 **关闭阴影**，表现就好了不少：

![img](/img/unity-grass5/screenshot8.png) 

当然，提升最大的还是 **自阴影**，如下图：

![img](/img/unity-grass5/screenshot9.png)

## 结尾

关于提升草海的 **光照表现**，大致就是以上的内容了，整个 **移动端草海的渲染方案** 也就写到这里。

在写这些文章的过程中，难免会 **怀个旧**。

最后附一张我们前项目草原的截图，2018年做的东西，用 **主美哥** 的话说，*放到今天，还能一战*，：）

![img](/img/unity-grass5/screenshot10.jpg)

好了，拜拜！


























































































































