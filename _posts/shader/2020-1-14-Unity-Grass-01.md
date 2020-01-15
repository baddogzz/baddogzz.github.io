---
layout: post
title: "移动端草海的渲染方案（一）"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
  - Shader
  - 小甜甜
---

## 塞尔达草海的模仿

**塞尔达的草海** 让人印象深刻，忍不住又要背诵台词：

> 我想起那天下午夕阳下的奔跑，那是我逝去的青春。

![img](/img/unity-grass/screenshot1.png)

好了，现在回来。

如果尝试用unity内置的地形草来还原上图效果，我们会发现有点力不从心：首先，内置草的shader不支持 **高光**。 

下图是前项目我们用传统的 **Blinn-Phong** 光照模型添加的高光：

![img](/img/unity-grass/screenshot2.png)

这里在计算光照时让 **草的法线向上**，大致可以模拟出塞尔达草海的高光形状。

不过，塞尔达的草可远不止于此：随风摆动，碰撞弯曲，可破坏，可点燃......

下面的视频是我们用unity对上述效果的高仿：

<iframe frameborder="0" width="720" height="480" src="https://v.qq.com/txp/iframe/player.html?vid=h3051zdbrxd" allowFullScreen="true"></iframe>

是不是有点帅，：）

本文以及后续的几篇文章陆续会介绍一下我们的实现方式，以及一些改进方案。

## Unity地形草的局限

我们的目标是要能在 **中高端移动设备** 跑得起 **至少60米** 视野范围的草海。 

如果用Unity提供的 **Terrain** 来刷草的话，下面的2个参数我们必须非常注意：

![img](/img/unity-grass/screenshot3.png)

为了记录刷草的信息，Unity会把我们的 **Terrain** 栅格化，**Detail Resolution** 指定了格子的 **划分粒度**：这个值越大精度越高，同时需要的内存也越高。

比如我们把 **Detail Resolution** 设置成 **512**，那么地表会被划分成 **512 x 512** 个格子，每个格子是刷草的最基本单位，草的 **密度** 决定了每个格子草的数量。

Unity的地形给我们提供了一个接口用于获取刷草信息：

> public int[,] GetDetailLayer(int xBase, int yBase, int width, int height, int layer);

这里 **GetDetailLayer** 返回的是一个二维数组，数组长宽的最大值和 **Detail Resolution** 是对应的。

考虑到大面积草的渲染，如果以每个格子里的单株草为单位，那 **drawcall** 会非常高。

对此，Unity做了它的优化：把一定数量的格子合并成一个 **Patch**，以 **Patch** 为单位来渲染，这样 **drawcall** 就能大幅度降低，**Detail Resolution Per Patch** 决定了每个Patch包含的格子数量。

比如我们把 **Detail Resolution Per Patch** 设为 **16**，那么每个 **Patch** 就包含了 16 * 16 = 256 个格子，Unity会把这 **256** 格里的草合并成一个大的Mesh，用于最终的渲染。

这个时候，你可能和我有一样的疑问，**GPU Instancing** 跑哪去了？

按照Unity的实现方案，每个 **Patch** 生成的Mesh是 **独立且各不相同** 的，**GPU Instancing** 的条件并不满足......

没有 **GPU Instancing** 就算了，你很快会发现另一个问题，新的 **Patch** 进入视野时，有严重的 **CPU性能开销**，看一下下图的峰值：

![img](/img/unity-grass/screenshot4.png)

![img](/img/unity-grass/screenshot5.png)

**Patch** 在运行时 **合并Mesh** 的开销很大。

这个时候，你可能已经心灰意冷，尝试着寻找 **Terrain刷草** 的替代方案了。

## 几个插件

事实上，国内用 **Terrain** 做地表的手游似乎也不多，更别说用 **Terrain刷草** 了。

不过了解 **Terrain** 的做法还是必要的，我们至少知道了Unity内置刷草的大致实现方式和存在的问题，以此为基础，再去理解一些第三方插件就容易得多了。

我们刚才的痛点主要有2个：

1. 不支持 GPU Instancing。

2. **Patch** 进入视野后合并模型的CPU开销很大。

这里介绍3款插件，都号称解决了我们的痛点：

1. [uNature](https://assetstore.unity.com/packages/vfx/shaders/unature-gpu-grass-and-interactable-trees-43129?aid=1101l85Tr)

2. [Advanced Terrain Grass](https://assetstore.unity.com/packages/tools/terrain/advanced-terrain-grass-100014?aid=1101l85Tr)

3. [Nature Renderer](https://assetstore.unity.com/packages/tools/terrain/nature-renderer-153552?aid=1101l85Tr)

当然这几个插件各有各的限制，我们是需要做二次开发的。

下篇文章会介绍一下这几个插件的实现原理，以及我们最终采用的方案。

拜拜！



















