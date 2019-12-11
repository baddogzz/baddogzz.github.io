---
layout: post
title: "关于pow编译后的代码"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
  - Shader
---

### 关于pow编译后的代码

**侑虎** 的深度测试报告里，有一个关于地表shader的报告：

![img](/img/shader-pow/screenshot1.jpg){:height="90%" width="90%"}

报告提到：

> 这里的 log2 在旧式Android机器上开销较高，是否可以优化掉？

从汇编代码来看，我们先 **log2** 得到一个结果，对其做了一个 **乘法** 运算，再 **exp2** 回去。 

现在问题来了，从第一感觉来说，我们似乎没有这样的代码。 

很是好奇，于是自己对着shader源码逐行分析了一下，最后发现这段汇编对应的代码如下：

```
inline half3 LinearToGammaSpace (half3 linRGB)
{
    linRGB = max(linRGB, half3(0.h, 0.h, 0.h));
    return max(1.055h * pow(linRGB, 0.416666667h) - 0.055h, 0.h);
}
``` 

没错了，**LinearToGammaSpace** 是Unity内置函数，把颜色从 **线性空间** 转回 **Gamma空间**。

这里配对出现的 **log2** 和 **exp2** 实际上是 **pow** 的计算。

> pow(x, y) = exp2(y * log2(x))

---

### 不同平台的编译结果对比

出于好奇，我对比了一下 **LinearToGammaSpace** 函数的 **Win版** 反汇编，结果类似，也是 **log + exp** 的方式。

#### Windows/D3D11

```
max r0.xyz, r0.xyzx, l(0.000000, 0.000000, 0.000000, 0.000000)
log r0.xyz, r0.xyzx
mul r0.xyz, r0.xyzx, l(0.416667, 0.416667, 0.416667, 0.000000)
exp r0.xyz, r0.xyzx
mad r0.xyz, r0.xyzx, l(1.055000, 1.055000, 1.055000, 0.000000), l(-0.055000, -0.055000, -0.055000, 0.000000)
max r0.xyz, r0.xyzx, l(0.000000, 0.000000, 0.000000, 0.000000)
```

#### Android/GLES3.0

```
u_xlat16_6.xyz = max(u_xlat16_6.xyz, vec3(0.0, 0.0, 0.0));
u_xlat16_3.xyz = log2(u_xlat16_6.xyz);
u_xlat16_3.xyz = u_xlat16_3.xyz * vec3(0.416666657, 0.416666657, 0.416666657);
u_xlat16_3.xyz = exp2(u_xlat16_3.xyz);
u_xlat16_3.xyz = u_xlat16_3.xyz * vec3(1.05499995, 1.05499995, 1.05499995) + vec3(-0.0549999997, -0.0549999997, -0.0549999997);
u_xlat16_3.xyz = max(u_xlat16_3.xyz, vec3(0.0, 0.0, 0.0));
```

此外，很多时候，编译器会帮我们做适当的优化。 

考虑一下 **pow(x, 2)** 这种情况，参考这篇帖子：[powx2的优化](https://www.gamedev.net/forums/topic/632528-are-gpu-drivers-optimizing-powx2/)

编译器最终会帮我们做如下优化：

> pow(x, 2) = x * x

多分析分析汇编代码，好处还是很多。

好了，拜拜。


