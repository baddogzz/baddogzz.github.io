---
layout: post
title: "一个自阴影的Bug"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Shader
  - 小甜甜
---

### 一个自阴影的Bug

最近刚按 **Github** + **Jekyll** 的方式搭建好个人主页，主页模板用的是 [Hux Blog](https://github.com/Huxpro/huxpro.github.io)。

今天把这个模板的Home页面背景图改成 **小甜甜** 了，问了一下 **主美哥** 意见，他看了一眼就立马指出问题：**角色没有自阴影，看起来有点飘。**

![img](/img/self-shadow-bug/screenshot1.jpg){:height="75%" width="75%"}
<center>没有自阴影的小甜甜</center>

神马？哥还会犯这种低级错误？

不过这里还是要佩服一下美术的眼睛，关注点和程序确实不一样。

---

### 原因

原因其实很简单：这个Shader是从前项目搬的，前项目没有用Unity的内置阴影，而是自己写的 **ShadowMap**。 

把这个Shader搬到新工程的时候，因为只是做一个封面，我就偷了个懒，阴影改回Unity内置的，并且Shader改成了**Surface Shader**。

我们知道，Unity内置的阴影需要 **ShadowCaster** 这个Pass去写 **ShadowMap**。 想要 **Surface Shader** 生成这个Pass，我们需要添加 **addshadow** 参数，或者 **FallBack** 指向的Shader包含这个Pass。 遗憾的是我漏了...
 
> addshadow - Generate a shadow caster pass. Commonly used with custom vertex modification, so that shadow casting also gets any procedural vertex animation. Often shaders don’t need any special shadows handling, as they can just use shadow caster pass from their fallback.

---

### 修正

修正很容易，因为无需特别定制 **ShadowCaster** 这个Pass，所以 **Surface Shader** 加上 **addshadow** 即可。

```
  #pragma surface surf BGStandard fullforwardshadows addshadow
```

![img](/img/self-shadow-bug/screenshot2.jpg){:height="75%" width="75%"}
<center>有自阴影的小甜甜</center>

附：一个最简单的 **ShadowCaster** Pass示例，摘自 **Legacy Shaders/VertexLit**

```
Pass 
{
	Name "ShadowCaster"
	Tags { "LightMode" = "ShadowCaster" }

	CGPROGRAM
	#pragma vertex vert
	#pragma fragment frag
	#pragma target 2.0
	#pragma multi_compile_shadowcaster
	#pragma multi_compile_instancing
	#include "UnityCG.cginc"

	struct v2f 
	{
		V2F_SHADOW_CASTER;
		UNITY_VERTEX_OUTPUT_STEREO
	};

	v2f vert( appdata_base v )
	{
		v2f o;
		UNITY_SETUP_INSTANCE_ID(v);
		UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO(o);
		TRANSFER_SHADOW_CASTER_NORMALOFFSET(o)
		return o;
	}

	float4 frag( v2f i ) : SV_Target
	{
		SHADOW_CASTER_FRAGMENT(i)
	}		
	ENDCG
}
```


















