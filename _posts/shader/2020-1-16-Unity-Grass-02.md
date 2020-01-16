---
layout: post
title: "移动端草海的渲染方案（二）"
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

前文介绍了Unity内置 **Terrain** 刷草的一些缺陷，并且介绍了3款插件：

1. [uNature](https://assetstore.unity.com/packages/vfx/shaders/unature-gpu-grass-and-interactable-trees-43129?aid=1101l85Tr)

2. [Advanced Terrain Grass](https://assetstore.unity.com/packages/tools/terrain/advanced-terrain-grass-100014?aid=1101l85Tr)

3. [Nature Renderer](https://assetstore.unity.com/packages/tools/terrain/nature-renderer-153552?aid=1101l85Tr)

下面就简单介绍一下这几款插件的做法，以及我们的选择。

## 如何刷草

Unity内置的刷草工具还是很好用的，[Advanced Terrain Grass](https://assetstore.unity.com/packages/tools/terrain/advanced-terrain-grass-100014?aid=1101l85Tr) 和 [Nature Renderer](https://assetstore.unity.com/packages/tools/terrain/nature-renderer-153552?aid=1101l85Tr) 沿用 **Terrain** 的刷草，只是接管了渲染。

参考一下 **TerrainData** 的API，我们是可以通过脚本获取刷草信息：

> public int[,] GetDetailLayer(int xBase, int yBase, int width, int height, int layer);

> public float GetHeight(int x, int y);

沿用 **Terrain** 的刷草方式有兼容性上的好处，但是这里就强迫你必须选择 **Terrain** 来做地表了。

[uNature](https://assetstore.unity.com/packages/vfx/shaders/unature-gpu-grass-and-interactable-trees-43129?aid=1101l85Tr) 和上面两个插件不太一样，他自己写了刷草工具，刷草的对象不局限于 **Terrain**，也可以是 **普通模型**。

比如下图，我不但在地表刷了草，也在Cube上刷了草。

![img](/img/unity-grass2/screenshot2.png)

## GPU Instancing

渲染大面积草，**GPU Instancing** 是非常合适的。

然而，Unity的渲染方案是把地表分成一个一个的 **Patch**，每个 **Patch** 的草合并成一个大的Mesh，以此来降低 **Drawcall**，但是 **多个Patch** 的渲染是无法通过 **GPU Instancing** 提速的。

我们看一下 **GPU Instancing** 需要满足的条件：

> Use GPU Instancing to draw (or render) multiple copies of the same Mesh
 at once, using a small number of draw calls. It is useful for drawing objects such as buildings, trees and grass, or other things that appear repeatedly in a Scene.

这里每个 **Patch** 生成的Mesh显然是不同的......

当然，我们可以突破这个限制。

既然要求 **相同Mesh**，那我们可以把造成 **Mesh差异** 的因素 ( 比如 **Noise** 和 **高度** ) 编码到纹理，然后在 **顶点着色器** 采样纹理再把这些差异应用到顶点。

这样我们就可以用相同的Mesh来渲染，即满足 **GPU Instancing** 的开启条件，又可以满足表现上的多样性，顺带把前文提到的 **运行时生成Mesh造成的CPU峰值** 也优化掉了。 

以 **uNature** 为例，场景依然会被栅格化，如下图：

![img](/img/unity-grass2/screenshot1.png)

这里的 **蓝色格子** 类似 **Terrain** 的 **Patch**，处于同一个 **紫色格子** 内的蓝色格子是可以通过 **GPU Instancing** 来渲染提速的。

如果不考虑 **LOD** 和 **密度** 的差异，每个 **蓝色格子** 的Mesh是一样的，最终表现上的差异被编码到了 **顶点uv** 以及 **GrassMap** 和 **HeightMap** 这2张纹理中去了。

**HeightMap** 一览：

![img](/img/unity-grass2/screenshot3.png)

具体的编码方式我就不细说了，大家可以参考源码。

事实上，Unity在 **2018.3** 及以后的版本，对 **Terrain** 的渲染也加了 **GPU Instancing** 的支持，原理和我上面说的差不多：

> When enabled, Unity transforms all of the heavy terrain data, like height maps and splat maps, into textures on the GPU. Instead of constructing a custom mesh for each terrain patch on the CPU, we can use GPU instancing to replicate a single mesh and sample the height map texture to produce the correct geometry. This reduces the terrain CPU workload by orders of magnitude, as a few instanced draw calls replace potentially thousands of custom mesh draws.

不过，一直到我目前在用的版本 **2019.3**，Unity关于 **Terrain刷草** 的渲染方式还是老样子......

## GPU Instancing 的 API

关于 **GPU Instancing**，如果通过脚本来操作，Unity提供了如下2个接口：

+ Graphics.DrawMeshInstanced

+ Graphics.DrawMeshInstancedIndirect

考虑到移动设备的兼容性，我们一般会选择 **Graphics.DrawMeshInstanced** 这个接口，不过 **Graphics.DrawMeshInstanced** 有一个最大数量 **1023** 的限制：

> Note: You can only draw a maximum of 1023 instances at once.

如果我们以每一株草为单位来渲染，很容易就会突破这个限制。

[Advanced Terrain Grass](https://assetstore.unity.com/packages/tools/terrain/advanced-terrain-grass-100014?aid=1101l85Tr) 就是这么做的，所以最后他用了 **Graphics.DrawMeshInstancedIndirect** 接口。

[uNature](https://assetstore.unity.com/packages/vfx/shaders/unature-gpu-grass-and-interactable-trees-43129?aid=1101l85Tr) 则是对草先做一定程度的 **Mesh合并**，回想一下这张图的 **蓝色格子**，我们可以通过控制格子的粒度，从而把每个 **紫色格子** 内的 **蓝色格子** 数控制在 **1023** 以内，然后就可以通过 **Graphics.DrawMeshInstanced** 这个接口一次完成渲染。

![img](/img/unity-grass2/screenshot1.png)

[Nature Renderer](https://assetstore.unity.com/packages/tools/terrain/nature-renderer-153552?aid=1101l85Tr) 的作者并没提供源码，不过从反编译的结果来看，他也是用了 **Graphics.DrawMeshInstanced** 这个接口，只是对 **GPU Instancing** 的 **Drawcall** 做了更细致的管理，如下图：

![img](/img/unity-grass2/screenshot4.png)

每个相同颜色的格子属于同一个 **Drawcall**，和 **uNature** 的 **9宫格** 管理方式并不相同。

## 我们的选择

好了，插件就介绍到这里。

最后，说一下我们的选择：基于 [uNature](https://assetstore.unity.com/packages/vfx/shaders/unature-gpu-grass-and-interactable-trees-43129?aid=1101l85Tr) 做改进。

+ 不选择 [Advanced Terrain Grass](https://assetstore.unity.com/packages/tools/terrain/advanced-terrain-grass-100014?aid=1101l85Tr)，主要因为它是基于 **Graphics.DrawMeshInstancedIndirect** 的实现。此外，如果你想实现类似塞尔达的割草功能，整个 **ComputeBuffer** 的数据都要重建，这个开销在运行时难以承受。

+ 不选择 [Nature Renderer](https://assetstore.unity.com/packages/tools/terrain/nature-renderer-153552?aid=1101l85Tr) 的原因则更简单，作者并不提供源码。

不过 [uNature](https://assetstore.unity.com/packages/vfx/shaders/unature-gpu-grass-and-interactable-trees-43129?aid=1101l85Tr) 本身的问题也不少，如果大家要用这个插件，你得有心里准备：

+ 作者已经很久没有更新了。
+ 代码有不少bug。
+ 针对移动端还要做很多优化。

无论如何，二次开发是必不可少的。

不过，有了 **GPU Instancing**，大面积的草海已经变得可行了。下面会继续介绍草的其他渲染技巧以及高仿塞尔达的一些好玩的效果。

好了，拜拜！





























