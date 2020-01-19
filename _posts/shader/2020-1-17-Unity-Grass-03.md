---
layout: post
title: "移动端草海的渲染方案（三）"
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

前文介绍了Unity内置的 **地形草（Terrain Detail）** 存在的问题以及一些优秀插件的优化方案。

这些插件做法各有差别，但是主要的优化方式可以总结为以下几点：

1. 利用 **GPU Instancing** 提速渲染。

2. 消除 **Terrain Patch** 运行时 **合并Mesh** 造成的 **CPU峰值**。

3. 利用 **多线程设计** 分担 **主线程** 的负担。

4. 提升摆动表现，提升光照表现。

前文主要介绍的是 **GPU Instancing**，这是一个大的优化方向。

有了 **GPU Instancing**，上面第二点提到的 **CPU峰值** 也自然而然的消除了，因为修改Mesh的操作已经从 **CPU移动到了GPU**，当然 **GPU** 的负担会加重一些。

至于 **多线程设计**，就是把一些计算从Unity的主线程抽离，比如 [uNature](https://assetstore.unity.com/packages/vfx/shaders/unature-gpu-grass-and-interactable-trees-43129?aid=1101l85Tr) 自己实现了一个 **多线程任务管理器**，把 **建立渲染管线** 的计算移动到了 **子线程**，典型的代码如下：

```csharp
public static void CreateRenderingQueue(RenderingQueueReceiver receiver, bool threaded = true)
{
    Vector3 originalCameraPosition = receiver.camera.transform.position + UNStandaloneUtility.GetStreamingAdjuster();
    originalCameraPosition.x = Mathf.Floor(originalCameraPosition.x);
    originalCameraPosition.z = Mathf.Floor(originalCameraPosition.z);

    FoliageCore_MainManager.WarmUpGrassMaps(receiver.neighbors, true);

    ThreadedRenderingQueueData threadData = new ThreadedRenderingQueueData(receiver, originalCameraPosition);

    var task = new ThreadTask<ThreadedRenderingQueueData>((ThreadedRenderingQueueData data) =>
    {
    	CreateRenderingQueue_Threaded(data.queueReceiver, data.targetedManagerInstances, data.originalCameraPosition);
    }, threadData);

    if (threaded)
    {
    	UNThreadManager.instance.RunOnThread(task);
    }
    else
    {
    	task.Invoke();
    }
}
```

上面的 **UNThreadManager** 就是 [uNature](https://assetstore.unity.com/packages/vfx/shaders/unature-gpu-grass-and-interactable-trees-43129?aid=1101l85Tr) 基于 **ThreadPool** 实现的一个多线程任务管理器，不过 **多线程** 的东西不是本文的重点，大家有兴趣可以去看源码。

在介绍 **提升摆动表现** 和 **提升光照表现** 之前，还有一些性能相关的问题需要注意。

## 面片草 vs 模型草

**塞尔达** 的草是 **模型草**，即便从顶视图看下来，也不会有面片感。

我们的美术对 **模型草** 念念不忘，但是程序往往会担心 **顶点数超标** 引发的性能问题。

其实，就性能方面来说，**面片草** 和 **模型草** 各有优劣，这里做一个简单的总结：

+ 面片草顶点数少，但是依赖 **AlphaTest**，这一步在很多移动设备上非常昂贵。

+ 模型草顶点数高，但因为是 **实体渲染**，性能表现远远优于 **AlphaTest**。

两害相权取其轻，我们最终的选择是 **模型草**。

选择模型草，除了性能优势之外，还有美术表现的优势，比如更好的光照表现，更好的碰撞表现。

此外，顶视图下的 **模型草** 也不会穿帮了，如下图：

![img](/img/unity-grass3/screenshot6.png)

当然，程序和美术都必须尽可能的压低模型草的顶点数。

从美术制作的角度来说，草的制作方式基本仿照 **塞尔达**：每根草叶要么是 **3个顶点的三角形**，要么是 **4个顶点的菱形**，不会再多了。

此外，**LOD** 对降顶点数非常重要，我们游戏内做了 **3级LOD**。

下图是我们 **LOD 0** 的草模型，大约 **100** 个顶点：

![img](/img/unity-grass3/screenshot1.png)

下图是我们 **LOD 1** 的草模型，大约 **50** 个顶点：

![img](/img/unity-grass3/screenshot2.png)

基本上每一级模型的顶点数都会减一半。

好了，美术已经做了他能做的。

现在，除了要求美术在刷草的时候尽量 **克制** 以外，剩下的就得靠程序了。

## 控制模型草的总顶点数

下图是美术大哥刷的草原氛围，很显然，大哥并不克制，这个草的密集程度已经很高了：

![img](/img/unity-grass3/screenshot3.png)

我们看到这里的三角形总数达到了 **234K**，如果把草屏蔽掉，三角形总数降低到了 **185K**，如下图：

![img](/img/unity-grass3/screenshot4.png)

**侑虎** 关于手游性能指标之 **三角形数上限** 的建议是 **200K**，由于我们的游戏是 **大世界超远视距**，外加地表开了 **深度图写入**，场景和角色都开了 **实时阴影**，**200K** 的三角形上限是 **Hold不住的**，经过反复的测试，我们最终把三角形上限设定为 **300K**，这里还有富余。

控制草的总顶点数，程序需要处理好以下几点：

+ 选择合适的 **Fade Distance**。

+ 选择合适的 **LOD** 策略。

+ 选择合适的 **Patch Size**。

+ 尽可能的剔除。

**Fade Distance** 以及 **LOD** 这个需要和美术讨价还价的，适合每个项目的需求就好，我们项目的 **Fade Distance** 是 **60米**，美术认为足够了。

此外，摄像机的 **视锥体剔除** 非常重要，剔除的精度和 **Patch Size** 相关：**Patch Size** 越小，剔除越精准，如下图：

![img](/img/unity-grass3/screenshot5.png)

需要注意的是，**Patch Size 并非越小越好**，主要原因有2个：

+ **Patch Size** 越小，总格子数越多，但是 **DrawMeshInstanced** 有最大数量 **1023** 的限制。

+ 总格子数越多，遍历或者管理的开销就越高，需要的内存也越多。

事实上，如果格子数很多，[uNature](https://assetstore.unity.com/packages/vfx/shaders/unature-gpu-grass-and-interactable-trees-43129?aid=1101l85Tr) 在更新视野的时候还有明显的 **GC负担**。

这里我们需要对 **uNature** 的代码做优化，同时权衡好 **Patch** 的尺寸。

## 自阴影

开启 **自阴影** 的草海会更加立体和真实，我们可以对比一下下面两张图：

*关闭自阴影*

![img](/img/unity-grass3/screenshot7.png)

*开启自阴影*

![img](/img/unity-grass3/screenshot8.png)

显然开 **自阴影** 的效果更好，代价是我们的草海会被 **绘制两遍**，第一遍写 **ShadowMap**，第二遍才是 **场景绘制**。

比如模型草的总顶点是 **50K**，那么绘制两遍就是 **100K**，**drawcall** 也会翻倍。

这个代价有点高，所以我们最终舍弃了 **自阴影**。

## 结尾

好了，就到这里。

下文会介绍一下如何提升草的 **摆动表现** 和 **光照表现**。

拜拜！





































































