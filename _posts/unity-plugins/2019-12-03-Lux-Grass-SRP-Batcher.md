---
layout: post
title: "Lux的Grass Shader和SRP Batcher"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
tags:
  - Lux
  - SRP
---

### Lux的Grass Shader和SRP Batcher

**Lux** 系列的shader一直以高质量著称，最近在玩 [Lux LWRP Essentials](https://assetstore.unity.com/packages/vfx/shaders/lux-lwrp-essentials-150355?aid=1101l85Tr)，这是 **Lux** 的轻量管线版本。

[Lux LWRP Essentials](https://assetstore.unity.com/packages/vfx/shaders/lux-lwrp-essentials-150355?aid=1101l85Tr) 的所有shader都支持 [SRP Batcher](https://blogs.unity3d.com/2019/02/28/srp-batcher-speed-up-your-rendering/)，以 **Grass** 为例，他的帮助文档有一段很有意思的说明：

>Using the SRP batcher combined with ​layer based culling →​ allows us to render even a vast  amount of grass patches / foliage instances quite efficiently on the CPU. Of course 1 mio  grass patches in a 1x1 km level are out of scope. But running 3K grass patches in a 300 x 300  meters levels isn’t a problem at all.  

意思是说，开启了 **SRP Batcher** 之后，渲染高数量级的草毫无压力。

![img](/img/lux-grass/screenshot-01.jpg)
<center>测试场景Environment Demo</center>

---

### 关于SRP Batcher

这里我其实有一点疑惑，按照 [SRP Batcher官方文档](https://blogs.unity3d.com/2019/02/28/srp-batcher-speed-up-your-rendering/) 所介绍，如果一个场景有很多不同的材质，但是shader很少，那么 **SRP Batcher** 可以发挥最大功效，因为材质内容会在 **GPU内存** 中持久保存，那么 **CPU** 设置材质属性的开销就可以大大减少。

>We aimed to speed up the general case where a Scene uses a lot of different Materials, but very few Shader variants.

> All Materials have persistent CBUFFERs located in the GPU memory, which are ready to use. To sum up, the speedup comes from two different things:
> 
> Each material content is now persistent in GPU memory
> 
> A dedicated code is managing a large “per object” GPU CBUFFER

但是回到上图的测试场景，每一株草的材质都是相同的，并且草的模型也是相同的，**GPU Instancing** 可以良好工作，在这种情况下 **SRP Batcher** 真的会快一点么？ 

还是先做一个测试。

---

### 关于SRPBatcherProfiler

官方提供了 [SRPBatcherBenchmark](https://github.com/Unity-Technologies/SRPBatcherBenchmark) 这个工具让我们去调试性能。

在编辑器下开关 **SRP Batcher** ，Profiler结果如下：

![img](/img/lux-grass/screenshot-02.jpg)
<center>SRP Batcher On</center>

![img](/img/lux-grass/screenshot-03.jpg)
<center>SRP Batcher Off</center>

对比结果如下：

类型 | SRP Batcher On | SRP Batcher Off
---|:--:|---:
SetPass Calls | 32 | 32 
Batches | 399 | 60
渲染实体  | 0.28ms | 0.36ms
渲染影子  | 0.38ms | 0.41ms

关闭 **SRP Batcher**，标准的渲染流程，**GPU Instancing** 正常工作，Batches只有 **60**。 打开 **SRP Batcher**，Batches一下飙到了 **399**，但是总的 **CPU耗时** 确实有降低。

继续测一下手机，这里关闭 **多线程渲染**，打一个包到我的 **小米 Mix 2** 手机上看一下，结果如下：

类型 | SRP Batcher On | SRP Batcher Off
---|:--:|---:
SetPass Calls | 27 | 27 
Batches | 406 | 59
渲染实体  | 2.26ms | 1.79ms
渲染影子  | 2.63ms | 2.48ms

在我手机上的测试结果，**SRP Batcher Off** 版本的cpu耗时反而更少。

有点不信邪，打开 **多线程渲染** 再测一次，结果一样：**SRP Batcher Off** 的版本cpu耗时更少。

---

### 测试环境和结论

**小米 Mix 2** 的处理器 **骁龙835** 还是比较强悍的，虽然测试场景的地表是PBR的，草也开了实时阴影，但是Profiler并没有发现GPU跟不上的情况。

我使用的Unity版本还是比较新的，**2019.2.12f1**，不过 **SRP Batcher** 依然还是标记为 **实验性的(Experimental)**。

我看到 **Unity 2019.3** 下，**LWRP** 已经升级为 **URP**，并且 **SRP Batcher** 的 **实验性** 标记已经去掉。考虑到 **Unity 2019.3** 依然处于Beta测试阶段，所以暂且就不升级了。

由于移动设备的测试结果不稳定，目前写 **LWRP版本Shader** 的策略如下：

+ Shader还是需要兼容 **SRP Batcher**。
+ 至于最后到底开不开，可以充分测试再决定。














