---
layout: post
title: "ShaderOne的体积雾"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
  - Shader
---

## ShaderOne的体积雾

[ShaderOne](https://assetstore.unity.com/packages/tools/visual-scripting/shaderone-beta-120661?aid=1101l85Tr&utm_source=aff) 是Unity商店的一个shader资源，作者期望把所有的渲染特性集合到一个shader上去，所以起个名字叫 **ShaderOne**。

本人其实比较反对这种 **All In One** 的shader，因为变种实在太多，优化起来比较蛋疼，不过作为一个学习资料还是挺好的。

今天我们来看一下他的 **体积雾** 的实现，先上截图：

![img](/img/volumetric-fog/screenshot1.gif){:height="60%" width="60%"} 

---

## 核心代码

说到体积雾，大家的第一个印象就是费。

没错了，指望 [ShaderOne](https://assetstore.unity.com/packages/tools/visual-scripting/shaderone-beta-120661?aid=1101l85Tr&utm_source=aff) 的 **体积雾** 能在手机上跑得飞起是有点强人所难的，因为计算涉及到 **射线步进** 和 **3d纹理采样**。

不过如果光线数设置的不那么高，高端手机还是可以一战的，文章的最后会给出一个测试。

在介绍 **体积雾** 之前，我们先看一下 [ShaderOne](https://assetstore.unity.com/packages/tools/visual-scripting/shaderone-beta-120661?aid=1101l85Tr&utm_source=aff) 提供的全部雾效类型，应用雾效的入口代码如下：

```
inline void FogApply ( inout SOLightingData i_sold, in v2f i )
{

#ifdef SO_GD_BASE_PASS
    #ifdef SO_GD_FOG_SOLID
        FogApplySolid ( i_sold, i );
    #endif

    #ifdef SO_GD_FOG_VOLUMETRIC
        FogApplyVolumetric ( i_sold, i );
    #endif

    #ifdef SO_GD_FOG_VOLUMETRIC_3D
        FogApplyVolumetric3D ( i_sold, i );
    #endif

#endif
}
```

可以看到 **ShaderOne** 提供了3种雾：

+ FogApplySolid：线性雾
+ FogApplyVolumetric：线性雾 + 指数高度雾
+ FogApplyVolumetric3D： 线性雾 + 3d指数高度雾

### FogApplySolid

**FogApplySolid** 就是最简单的**线性雾**，提供的参数就三个：颜色、开始距离、结束距离。

![img](/img/volumetric-fog/screenshot2.png){:height="35%" width="35%"} 

![img](/img/volumetric-fog/screenshot3.png){:height="75%" width="75%"} 

代码简单到不想废话：

```
inline void FogApplySolid ( inout SOLightingData i_sold, in v2f i )
{
#ifdef SO_GD_BASE_PASS
#ifdef SO_SF_FOG_ON
#ifdef SO_GD_FOG_SOLID
    half    dist            = distance ( _WorldSpaceCameraPos.xyz , i.positionWorld );
    half    distFade        = 1.0 - clamp ( ( _FogEndDistance - dist ) / _FogSolidDistance, 0.0, 1.0 );
    i_sold.finalRGBA.rgb    = lerp ( i_sold.finalRGBA.rgb, _FogColorFade, distFade );
#endif
#endif
#endif
}
```

### FogApplyVolumetric

**FogApplyVolumetric** 是 **指数高度雾** 再混合之前的 **线性雾**。

![img](/img/volumetric-fog/screenshot4.png){:height="35%" width="35%"} 

![img](/img/volumetric-fog/screenshot5.png){:height="75%" width="75%"} 

这里除了之前的线性雾参数之外，又加了高度雾参数。可以看到雾有高低和远近的过渡，并且有2种雾色。

关于指数高度雾，这里的计算公式如下：

```
fogAmount = 1.0 - exp ( -dist * ( fogHeightIntensity * _FogDensity ) );
```

这里的公式和Unity的 **Exponential** 雾类似，只是多了一个 **fogHeightIntensity** 这个高度因子。

**fogHeightIntensity** 的计算公式类似线性雾，不过是算的是高度：

```
fogHeightIntensity = ( 1.0 - min ( ( length (  i.positionWorld.y - _FogHeight ) / _FogHeightSize ), 1.0 ) );
```

最后雾效和场景颜色的混合计算，是先算指数高度雾，后算线性雾，代码如下：

```
i_sold.finalRGBA.rgb = lerp ( i_sold.finalRGBA.rgb, fogColor, fogAmount );
i_sold.finalRGBA.rgb = lerp ( i_sold.finalRGBA.rgb, _FogColorFade, distFade );
```

代码还是比较清楚的，市面上大部分手游的雾效和这个类似，只是计算公式可能有一点差别。

### FogApplyVolumetric3D

最后来看一下 **FogApplyVolumetric3D**，也就是 **体积雾**。

**FogApplyVolumetric3D** 和 **FogApplyVolumetric** 的设置差不多，多了一些体积雾相关的参数，还有一个全局的射线数量需要设置：

![img](/img/volumetric-fog/screenshot6.png){:height="35%" width="35%"} 

![img](/img/volumetric-fog/screenshot7.png){:height="75%" width="75%"} 

![img](/img/volumetric-fog/screenshot8.png){:height="75%" width="75%"} 

两者的计算流程也非常相似，指数雾的公式如下：

```
fogAmount = 1.0 - exp ( -dist * fogAmount );
```

不过这里 **fogAmount** 的计算和 **FogApplyVolumetric** 相比就比较复杂了：为了平滑的勾勒出体积，这里需要依靠 **射线步进**，主要代码如下：

```
fogAmount       = 0;

for ( int i = 0; i < SO_GD_FOG_RAYCOUNT; i++ )
{
    rayPos      += viewDir * stepSize;

    fogValue    = tex3D ( _FogTexture3D, ( rayPos * _FogScale ) + fogUVOffset ).a;

    #ifdef SO_GD_FOG_ROUGHNESS_ON
        fogYLength = length ( rayPos.y - ( _FogHeight + ( ( fogValue - 0.5 ) * _FogVerticalRoughness ) ) );
    #else
        fogYLength  = length ( rayPos.y - _FogHeight );
    #endif

    heightIntensity  = 1.0 - min ( ( fogYLength / _FogHeightSize ), 1.0 );

    fogAmount   += fogValue * heightIntensity;
}

fogAmount = ( fogAmount / SO_GD_FOG_RAYCOUNT ) * _FogDensity;
```

上述代码的大致计算流程如下：

+ 从摄像机向场景发射 **SO_GD_FOG_RAYCOUNT** 条射线，这里的 **SO_GD_FOG_RAYCOUNT** 即前面的全局射线数，这个值越高效果越好，当然也越费。

+ 每一条射线从摄像机点沿着 **viewDir** 方向逐步向前探测 **stepSize** 段距离，并且从一个预定的3d纹理 **_FogTexture3D** 去采样雾的密度。

+ 最后把这么多根射线的采样结果加权平均，得出 **fogAmount**。

#### 关于_FogTexture3D

可以看到，雾的体积由 **_FogTexture3D** 这张3d纹理决定。

**_FogTexture3D** 是一个3维的噪声图，如果我们想要自己调整噪音参数，可以借由作者提供的一个工具：**Fog Texture Designer**。

![img](/img/volumetric-fog/screenshot9.png){:height="50%" width="50%"} 

这个工具依赖另一个Unity的插件：[FastNoise](https://assetstore.unity.com/packages/tools/particles-effects/fastnoise-70706?aid=1101l85Tr)，具体细节可以参考插件源码，这个不是本文的重点。

#### 关于stepSize的计算

前文提到 **射线步进** 的步长是 **stepSize**，**stepSize** 的计算方式如下：

```
dist            = length ( _WorldSpaceCameraPos.xyz - i.positionWorld ) - _FogOffset;
distFade        = 1.0 - clamp ( ( _FogEndDistance - dist ) / _FogSolidDistance, 0.0, 1.0 );
stepSize        = dist / (float)SO_GD_FOG_RAYCOUNT;
```

这里比较简单，把dist按照射线数等分即可。

#### 关于SO_GD_FOG_RAYCOUNT

**SO_GD_FOG_RAYCOUNT** 控制射线的数量，这个参数对体积雾的质量影响较大，下图是只发3条射线的计算结果：

![img](/img/volumetric-fog/screenshot10.png){:height="75%" width="75%"}

可以看到之前的平滑过渡出现了断裂。

#### 关于Fog Roughness

**Fog Roughness** 通过给高度增加扰动来增加雾的高低起伏，计算代码如下：

```
#ifdef SO_GD_FOG_ROUGHNESS_ON
    fogYLength = length ( rayPos.y - ( _FogHeight + ( ( fogValue - 0.5 ) * _FogVerticalRoughness ) ) );
#else
    fogYLength = length ( rayPos.y - _FogHeight );
#endif

heightIntensity  = 1.0 - min ( ( fogYLength / _FogHeightSize ), 1.0 );
```

下图是 **Fog Roughness** 设置为0和设置为16的效果对比：

![img](/img/volumetric-fog/screenshot11.png){:height="100%" width="100%"}

## 移动设备的帧率

最后，看看哥的 **小米MIX2** 是否可以一战。

24条射线，Roughness不开很高的情况下，稳稳60帧，效果也能接受，如下图：

![img](/img/volumetric-fog/screenshot12.jpg){:height="75%" width="75%"}

好了，拜拜。







