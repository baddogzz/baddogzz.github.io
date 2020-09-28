---
layout: post
title: "关于RenderFeature重复计算的一个Bug"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
  - URP
---

## 径向模糊的掉帧问题

渲染迁移到 **URP** 的工作基本差不多了，不过昨天跑包的时候遇到一个问题，在播放 **径向模糊** 效果的时候，**帧率** 下降得很厉害，并且UI也跟着场景一起模糊了，如下图：

![img](/img/renderfeature-repeat-bug/screenshot1.png)

这个问题在原先的 **标准管线** 下并不存在。

## 原因

简单检查了一下原因，发现 **CameraStack** 上的所有相机都执行了径向模糊这个 **RenderFeature**，包括 **UI相机**，我们 **CameraStack** 的设置如下图：

![img](/img/renderfeature-repeat-bug/screenshot4.png)

看了一下 **URP** 的代码，发现无论是渲染 **Base相机** 相机还是渲染 **Overlay相机**，最终都会走到 **RenderSingleCamera** 这个函数，如下图：

![img](/img/renderfeature-repeat-bug/screenshot3.png)

**RenderSingleCamera** 的主要流程如下：

![img](/img/renderfeature-repeat-bug/screenshot5.png)

上图的 **Setup** 流程会执行到 **RenderFeature** 的 **AddRenderPasses** 函数，而 **Execute** 流程会执行到 **RenderPass** 的 **Execute** 函数。

由此可见，如果我们不对相机进行区分，所有相机都会执行一遍 **径向模糊** 这个 **RenderFeature**，这就导致了在播放这个效果时帧率显著下降，同时UI也跟着糊掉了...

## 修正

修正方式很简单，我们在 **RenderFeature** 的 **AddRenderPasses** 函数中对相机做一个过滤即可，代码如下：

![img](/img/renderfeature-repeat-bug/screenshot6.png)

当然，我们也可以对不同相机设置不同的 **Renderer**，不过这里我不想维护多个 **Renderer**，所以还是直接代码加个判断就好了。

修正后一切正常了：

![img](/img/renderfeature-repeat-bug/screenshot2.png)

好了，拜拜！































































































































































































































































