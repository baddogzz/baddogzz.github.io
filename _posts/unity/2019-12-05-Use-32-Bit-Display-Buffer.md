---
layout: post
title: "关于 Use 32-bit Display Buffer"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
  - 打包
---

### 一个Bug：辉光没了

在 **Android** 设备上，之前工作良好的 **辉光** 在某一个版本突然没了。

看了一下 **svn log**，并没有什么代码改动，但 **主美哥** 用他的 **24K氪金狗眼** 跟我保证：辉光真的没了。

好吧，检查了一下发现，辉光的计算依然还在，但是计算结果却是一片黑...

下面是 **FrameDebugger** 的调试结果：

![img](/img/use-32bit-buffer/screenshot1.jpg)
<center>辉光计算依然存在</center>

![img](/img/use-32bit-buffer/screenshot2.jpg){:height="75%" width="75%"}
<center>_BloomTex一片漆黑</center>

**_BloomTex** 存储的是 **屏幕中的亮色区域经过下采样/上采样和模糊处理** 后的结果。

**Uber** Shader最终会上采样 **_BloomTex**，并且和屏幕颜色做叠加以产生辉光，由于 **_BloomTex** 一片黑，所以辉光基本就没了。

```
// HDR Bloom
#if BLOOM
{
  half3 bloom = UpsampleFilter(_BloomTex, uv, _BloomTex_TexelSize.xy, _Bloom_Settings.x) * _Bloom_Settings.y;
  color += bloom;
}
#endif
```

---

### 分析

那么，在没改动代码的情况下，之前工作良好的 **_BloomTex** 为什么会突然一片黑？ 

这个时候我就十分怀疑 **RenderTexture的格式** 是否正确了。

**FrameDebugger** 在调试移动设备的时候有点力不从心，所以这里换 [RenderDoc](https://renderdoc.org/) 出场。

下面就开始分析辉光每一步的计算结果。

#### 第一个Pass：Prefilter

辉光的第一步计算：从屏幕上选出亮的区域。 用 **RenderDoc** 查看一下这个Pass的输入和输出。

![img](/img/use-32bit-buffer/screenshot3.jpg)
<center>Inputs: R11G11B10_FLOAT</center>

![img](/img/use-32bit-buffer/screenshot4.jpg)
<center>Outputs: B5G5R5A1_UNORM</center>

由截图可见，由于我们开了 **HDR**，并且 **HDR Mode** 选择了 **R11G11B10**，作为输入的TempBuffer格式是吻合的。

但是输出的格式居然是 **B5G5R5A1**，由于我们选用的 **PostProcessing Stack V1** 版本在移动设备上是通过 **RGBM** 来编码 **HDR** 的，16位的RT显然无法满足编码要求，这就是一切错误的根源了。

```csharp
// Blur buffer format
// TODO: Extend the use of RGBM to the whole chain for mobile platforms
var useRGBM = Application.isMobilePlatform;
var rtFormat = useRGBM ? RenderTextureFormat.Default : RenderTextureFormat.DefaultHDR;
```

```
half4 EncodeHDR(float3 rgb)
{
#if USE_RGBM
    //rgb *= 1.0 / 8.0;
	rgb *= 0.125;
    float m = max(max(rgb.r, rgb.g), max(rgb.b, 1e-6));
    m = ceil(m * 255.0) / 255.0;
    return half4(rgb / m, m);
#else
    return half4(rgb, 0.0);
#endif
}
```

#### 为什么RT会变成16位的？

> *是我干的，嗯。*

定位到了原因后，我突然想起自己前几天随手改了一个打包设置 **Use 32-bit Dispaly Buffer**。

![img](/img/use-32bit-buffer/screenshot5.jpg)

Unity官方文档对其的说明如下：

> Enable this option to create a Display Buffer to hold 32-bit color values (16-bit by default). Use it if you see banding, or need alpha in your post-processed effects, because they create Render Textures in the same format as the Display Buffer.

文档说得挺清楚的了，我的 **post-processed effects** 确实需要 **alpha**，但是我却选择了16位的RT。

---

### 修正

修正很容易，把这个勾勾回去就好了。

不过 **PostProcessing Stack V1** 这里写的也不够严谨啊，对 **HDR** 的兼容性如果做的好一点，也不会有这个bug。

当然，现在 **PostProcessing Stack V1** 已经被淘汰了，只是我们还没升级到 [Post-processing Stack v2](https://github.com/Unity-Technologies/PostProcessing)。

好了，拜拜。









