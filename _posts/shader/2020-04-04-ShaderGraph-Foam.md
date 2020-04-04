---
layout: post
title: "用ShaderGraph实现卡通的沙滩泡沫效果"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
  - Shader
---

## 一个卡通的水

本文用 **ShaderGraph** 来模仿一下 [The Illustrated Nature](https://assetstore.unity.com/packages/3d/vegetation/the-illustrated-nature-153939?aid=1101l85Tr) 插件中水的 **泡沫效果**，原效果如下：

![](/img/cartoon-foam/screenshot1.png)

下图是我用 **ShaderGraph** 模拟的效果：

![](/img/cartoon-foam/screenshot2.png)

代码我丢在了github上，地址：[https://github.com/fatdogsp/URP-Simple-Water](https://github.com/fatdogsp/URP-Simple-Water)。

## 泡沫的实现细节

[The Illustrated Nature](https://assetstore.unity.com/packages/3d/vegetation/the-illustrated-nature-153939?aid=1101l85Tr) 是风格化渲染，水的shader其实还是比较简单的，并没有常见的 **反射** 或者 **折射**，本文主要记录一下 **泡沫** 部分的实现细节。

这里并不需要任何 **泡沫贴图**，只提供了一个 **泡沫颜色**，而泡沫的形状主要由如下两个部分组成：

+ 规则的边缘线条
+ 不规则的噪声线条

在计算泡沫线条之前，我们首先需要计算出水的深度，因为泡沫只会出现在浅水区。

深度计算比较常规，代码如下:

![](/img/cartoon-foam/screenshot3.png)

这里计算出水和场景在摄像机空间下的 **深度差**，用于后续的泡沫计算。

#### 规则的边缘线条

我们可以定一个泡沫出现的最大深度，比如0.3，超出这个深度，泡沫就消失。

此外，在出现泡沫的深度区间，我们还可以利用 **正弦函数** 来模拟海浪的效果，代码如下：

![](/img/cartoon-foam/screenshot4.png)

动态效果图如下：

![](/img/cartoon-foam/screenshot5.gif)

这里区间转换节点 **Remap** 非常好用，谁用谁知道。

#### 不规则的噪声线条

在规则的泡沫线条上再叠加一层不规则的噪声线条，整个泡沫就比较完美了。

这里我参照 [The Illustrated Nature](https://assetstore.unity.com/packages/3d/vegetation/the-illustrated-nature-153939?aid=1101l85Tr) 的做法，用2层不同scale的 **Simple Noise** 相叠加，配合 **UV动画** 模拟泡沫的运动，代码如下：

![](/img/cartoon-foam/screenshot6.png)

#### 深度比较

在计算线条的时候，我用 **step** 去做深度比较，这个计算结果非0即1，所以泡沫和非泡沫的边界是很硬的，这样符合卡通渲染的特征。

当然，如果需要做边界过度，我们可以用 **smoothstep** 来替代 **step**。

#### 泡沫的颜色合成

泡沫的颜色合成就是把上面的两层泡沫相加，代码如下：

![](/img/cartoon-foam/screenshot7.png)

这个很简单，就不罗嗦了。

## 水体的其他部分

其他部分都是常规代码，水波纹很简化，有兴趣的自己改。

好了，拜拜。













































































































































































