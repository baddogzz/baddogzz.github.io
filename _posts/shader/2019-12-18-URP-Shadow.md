---
layout: post
title: "关于SHADOWS_SCREEN"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
  - Shader
  - URP
---

### 关于SHADOWS_SCREEN

在读Unity内置shader源码的过程中，我们经常看到这样的代码：

```
#if SHADOWS_SCREEN       
    // xxx
#else
    // xxx
#endif
```

那么，上面代码中的 **SHADOWS_SCREEN** 代表的是什么呢？ 

通过仔细阅读代码发现，**SHADOWS_SCREEN** 代表的是 **屏幕空间阴影**。

以 **URP** 管线 **主灯的实时阴影** 为例，我们看一下相关代码：

#### 主灯影衰减的入口

```
Light GetMainLight(float4 shadowCoord)
{
    Light light = GetMainLight();
    light.shadowAttenuation = MainLightRealtimeShadow(shadowCoord);
    return light;
}
```

#### 主灯实时阴影的主要函数

```
half MainLightRealtimeShadow(float4 shadowCoord)
{
#if !defined(_MAIN_LIGHT_SHADOWS) || defined(_RECEIVE_SHADOWS_OFF)
    return 1.0h;
#endif

#if SHADOWS_SCREEN
    return SampleScreenSpaceShadowmap(shadowCoord);
#else
    ShadowSamplingData shadowSamplingData = GetMainLightShadowSamplingData();
    half4 shadowParams = GetMainLightShadowParams();
    return SampleShadowmap(TEXTURE2D_ARGS(_MainLightShadowmapTexture, sampler_MainLightShadowmapTexture), shadowCoord, shadowSamplingData, shadowParams, false);
#endif
}
```

这里我们看到，如果定义了 **SHADOWS_SCREEN**，那么会走 **屏幕空间阴影** 的采样流程，否则，走 **ShadowMap** 的采样流程。

#### SHADOWS_SCREEN的定义

那么 **SHADOWS_SCREEN** 在什么情况下会被定义呢？我们继续看代码：

```
#ifndef SHADOWS_SCREEN
    #if defined(_MAIN_LIGHT_SHADOWS) && defined(_MAIN_LIGHT_SHADOWS_CASCADE) && !defined(SHADER_API_GLES)
        #define SHADOWS_SCREEN 1
    #else
        #define SHADOWS_SCREEN 0
    #endif
#endif
```

这里是否采用 **屏幕空间阴影**，主要取决于是否开启了 **阴影级联**。

对于大部分使用 **SHADER_API_GLES** 的 **Android** 设备来说，也会关闭 **屏幕空间阴影**。

### 开启屏幕空间阴影的渲染流程

Unity针对 **GLES** 设备关闭 **屏幕空间阴影**，说明它的渲染开销较高。

**屏幕空间阴影** 多用于 **延迟渲染**，不过 **URP** 还是一个 **前向渲染** 的流程，那么我们就来看一下开启 **屏幕空间阴影** 后 **URP** 的渲染流程。

以Unity自带的 **SampleScene** 为例，并且只考虑 **一盏主灯** 的情况，如下图：

![img](/img/urp-shadow/screenshot1.png)

这里开了 **2级阴影级联**，**ShadowMap** 的分辨率是 **1024**，**Shadow Distance** 是 **64**。

#### 第一步，绘制ShadowMap

因为开了 **2级级联**，实际创建的 **ShadowMap** 的分辨率是 **1024 x 512**，左边的 **512 x 512** 绘制 **0级阴影**，右边的 **512 x 512** 绘制 **1级阴影**，如下：

![img](/img/urp-shadow/screenshot2.png){:height="75%" width="75%"}

要写入 **ShadowMap**，shader必须有 **ShadowCaster** 这个Pass。

#### 第二步，绘制CameraDepthTexture

要生成 **屏幕空间阴影** 贴图，我们需要通过 **屏幕坐标** 和 **当前像素的深度** 还原出屏幕上一个点的 **世界坐标**。

而要获取 **当前像素的深度**，我们就必须写深度贴图 **CameraDepthTexture**，如下：

![img](/img/urp-shadow/screenshot3.png){:height="75%" width="75%"}

要写入 **CameraDepthTexture**，shader必须有 **DepthOnly** 这个Pass。

对于 **移动设备** 来说，如果场景比较复杂，**drawcall** 会大幅增加，这一步会比较昂贵。

#### 第三步，生成屏幕空间阴影贴图

有了 **ShadowMap** 和 **CameraDepthTexture**，就可以生成名为 **_ScreenSpaceShadowmapTexture** 的 **屏幕空间阴影贴图** 了：

![img](/img/urp-shadow/screenshot4.png){:height="75%" width="75%"}

生成的代码见 **Shaders/Utils/ScreenSpaceShadows.shader**：

```
half4 Fragment(Varyings input) : SV_Target
{
    UNITY_SETUP_STEREO_EYE_INDEX_POST_VERTEX(input);

    float deviceDepth = SAMPLE_TEXTURE2D_X(_CameraDepthTexture, sampler_CameraDepthTexture, input.uv.xy).r;

    #if UNITY_REVERSED_Z
            deviceDepth = 1 - deviceDepth;
    #endif
    deviceDepth = 2 * deviceDepth - 1; //NOTE: Currently must massage depth before computing CS position.

    float3 vpos = ComputeViewSpacePosition(input.uv.zw, deviceDepth, unity_CameraInvProjection);
    float3 wpos = mul(unity_CameraToWorld, float4(vpos, 1)).xyz;

    //Fetch shadow coordinates for cascade.
    float4 coords = TransformWorldToShadowCoord(wpos);

    // Screenspace shadowmap is only used for directional lights which use orthogonal projection.
    ShadowSamplingData shadowSamplingData = GetMainLightShadowSamplingData();
    half4 shadowParams = GetMainLightShadowParams();
    return SampleShadowmap(TEXTURE2D_ARGS(_MainLightShadowmapTexture, sampler_MainLightShadowmapTexture), coords, shadowSamplingData, shadowParams, false);
}
```

#### 第四步，渲染场景

有了 **屏幕空间阴影贴图**，我们就可以根据 **屏幕坐标** 采样贴图，得到光的 **影衰减**，最终用于光照计算。

主灯实时阴影的相关代码如下：

**顶点着色器：**

```
output.ShadowCoords = GetShadowCoord(vertexInput); 
```

```
float4 GetShadowCoord(VertexPositionInputs vertexInput)
{
#if SHADOWS_SCREEN
    return ComputeScreenPos(vertexInput.positionCS);
#else
    return TransformWorldToShadowCoord(vertexInput.positionWS);              
#endif
}
```

```
float4 ComputeScreenPos(float4 positionCS) 
{               
    float4 o = positionCS * 0.5f;
    o.xy = float2(o.x, o.y * _ProjectionParams.x) + o.w;
    o.zw = positionCS.zw;
    return o;
}
```

**像素着色器：**

```
half MainLightRealtimeShadow(float4 shadowCoord)
{
#if !defined(_MAIN_LIGHT_SHADOWS) || defined(_RECEIVE_SHADOWS_OFF)
    return 1.0h;
#endif

#if SHADOWS_SCREEN
    return SampleScreenSpaceShadowmap(shadowCoord);
#else
    ShadowSamplingData shadowSamplingData = GetMainLightShadowSamplingData();
    half4 shadowParams = GetMainLightShadowParams();
    return SampleShadowmap(TEXTURE2D_ARGS(_MainLightShadowmapTexture, sampler_MainLightShadowmapTexture), shadowCoord, shadowSamplingData, shadowParams, false);
#endif
}
```

```
half SampleScreenSpaceShadowmap(float4 shadowCoord)
{
    shadowCoord.xy /= shadowCoord.w;

    // The stereo transform has to happen after the manual perspective divide 
    shadowCoord.xy = UnityStereoTransformScreenSpaceTex(shadowCoord.xy);

#if defined(UNITY_STEREO_INSTANCING_ENABLED) || defined(UNITY_STEREO_MULTIVIEW_ENABLED)
    half attenuation = SAMPLE_TEXTURE2D_ARRAY(_ScreenSpaceShadowmapTexture, sampler_ScreenSpaceShadowmapTexture, shadowCoord.xy, unity_StereoEyeIndex).x;
#else
    half attenuation = SAMPLE_TEXTURE2D(_ScreenSpaceShadowmapTexture, sampler_ScreenSpaceShadowmapTexture, shadowCoord.xy).x;
#endif

    return attenuation;
}
```

### 结尾

很多游戏的实时阴影都不是Unity内置的，如今在 **URP** 开放源码的情况下，我正在考虑把我们游戏的实时阴影切回Unity的内置版本。

去年为了抄 **塞尔达**，我们在移动设备搞过 **全场景的实时阴影**： 

+ 2级级联
+ 70米实时影
+ PCF2X2采样

当时基于一个叫 **Sunshine** 的Unity插件做了大量优化，在我的 **小米MIX2** 上还是非常流畅的，怀念一下：

![img](/img/urp-shadow/screenshot5.jpg){:height="100%" width="100%"}






















