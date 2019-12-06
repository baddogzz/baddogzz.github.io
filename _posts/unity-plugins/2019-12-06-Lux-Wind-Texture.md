---
layout: post
title: "Lux的风和WindTexture"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
  - Lux
---

### Lux的风和WindTexture

玩 [Lux LWRP Essentials](https://assetstore.unity.com/packages/vfx/shaders/lux-lwrp-essentials-150355?aid=1101l85Tr) 的过程中发现了挺多好玩的东西，比如 它的风，实现细节就挺有趣。

下图是前项目草的摆动效果：

![img](/img/lux-wind/screenshot2.gif){:height="75%" width="75%"}

这里的顶点摆动计算比较简单，没有 **WindZone** 之类的设置，就是周期性的正余弦运动。 效果其实还不错，计算量也不算大。

如果用 **Lux的风** 来驱动草的摆动，效果如下：

![img](/img/lux-wind/screenshot1.gif){:height="75%" width="75%"}

摆动明显加入了 **噪音**，并且风的方向可以调整，表现更加真实。

下面就来看一下它的实现细节。

### 实现细节

下班了，回头再写。。。。。。