---
layout: post
title: "修正半透明头发的渲染异常"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
  - Shader
  - 小甜甜
---

### 修正半透明头发的渲染异常

这两天在整理 **小甜甜** 的捏脸系统，顺带把前项目 **半透明头发渲染异常的bug** 修正了，效果如下：

![img](/img/fix-hair-transparent/screenshot1.png){:height="70%" width="70%"}

其实，前项目的头发只是有一点 **半透明渲染顺序** 的小瑕疵，但是正如前文 [一个自阴影的Bug](https://baddogzz.github.io/2019/12/04/Self-Shadow-Bug/) 所述，迁移shader的时候因为偷懒，我把头发改成了 **Surface Shader**，从而带来了一些新的问题。

---

### 问题一：关于Surface Shader的alpha指令

首先，我犯了第一个错误，改成 **Surface Shader** 的过程中，我漏掉了 **alpha** 指令的设置，shader变成了 **实体渲染**，表现如下：

![img](/img/fix-hair-transparent/screenshot2.png){:height="70%" width="70%"}

关于 **Surface Shader的alpha指令**，可以参考Unity帮助文档：

> alpha or alpha:auto - Will pick fade-transparency (same as alpha:fade) for simple lighting functions, and premultiplied transparency (same as alpha:premul) for physically based lighting functions.

> alpha:blend - Enable alpha blending.

> alpha:fade - Enable traditional fade-transparency.

> alpha:premul - Enable premultiplied alpha transparency.

虽然我指定了 **Queue = Transparent** 和 **Blend SrcAlpha OneMinusSrcAlpha**，但因为没设置 **alpha**，Unity最终生成的代码给我加了这么一句：

```
UNITY_OPAQUE_ALPHA(c.a);
```

UNITY_OPAQUE_ALPHA定义如下：

```
#define UNITY_OPAQUE_ALPHA(outputAlpha) outputAlpha = 1.0
```

这里认为是 **实体渲染**，强制的把alpha设置为 **1** 了。


修正这个问题很简单，加上 **alpha** 指令的设置即可：

```
#pragma surface surf BGHair fullforwardshadows addshadow vertex:vert alpha
```

--- 

### 问题二：半透明的渲染顺序

加上 **alpha** 后，头发通透了好多，但是新的问题来了，当 **头发的半透明部分相互交错** 时，渲染表现错误：

![img](/img/fix-hair-transparent/screenshot3.png){:height="70%" width="70%"}

因为是半透明渲染，**Surface Shader**生成的代码开启了 **深度测试**，关闭了 **深度缓冲写入**，当 **半透部分相互交错** 时，前后关系难以保证。

这是一个半透材质常遇到的问题，我们可以把头发模型拆成 **实体** 和 **半透** 两部分，先渲染 **实体部分**，再渲染 **半透部分**，在实体的基础上做混合，从而尽可能避免半透交错的机会。

不过当初做这个头发的时候，并没有拆分模型，做法也很简单：开启 **深度缓冲写入**，让头发的半透明部分仅局限于 **发尾**，由于半透区域很小，一般情况下很难穿帮，不过某些角度下细看还是会有问题。

比如下图红圈标注的部位，后面的头发因为 **深度测试** 没通过，导致前面的头发直接和背景混合，引发了表现错误。

![img](/img/fix-hair-transparent/screenshot4.png){:height="70%" width="70%"}

此外，因为迁移到了 **Surface Shader**，我发现如果打开了 **alpha** 指令，无论我是否指定 **ZWrite On**，Unity给我生成的代码始终是 **ZWrite Off** 的。 

---

### 解决方式一：用AlphaTest代替AlphaBlend

如果这里我们用 **AlphaTest** 来取代 **AlphaBlend** ，效果会如何呢？

作为测试，简单的加一句 **clip**，并且关闭 **alpha** 指令：

```
clip(mainColor.a - 0.6);
```

效果如下：

![img](/img/fix-hair-transparent/screenshot5.png){:height="70%" width="70%"}

渲染正确了，只是发尾太硬，用美术的话说：**不够透气**。

---

### 解决方式二：加一个Pass

**方式一** 不够完美，不过我们可用它做一个 **额外的Pass**，类似拆分模型后的 **实体部分渲染**，渲染完实体后再做之前的 **半透明渲染**。

代码如下：

```
Pass
{
    CGPROGRAM
    #pragma vertex vert
    #pragma fragment frag
    #include "UnityCG.cginc"

    sampler2D _MainTex;
    float4 _MainTex_ST;

    half4 _Color;
    half4 _Color2;

    half _Color2Offset;

    struct appdata            
    {
        float3 pos : POSITION;    
        float3 uv0 : TEXCOORD0;   
        UNITY_VERTEX_INPUT_INSTANCE_ID
    };
 
    struct v2f 
    {
        UNITY_POSITION(pos);
        float4 uv0 : TEXCOORD0;
        UNITY_VERTEX_OUTPUT_STEREO
    };
 
    v2f vert (appdata IN)
    {
        v2f o;
        UNITY_SETUP_INSTANCE_ID(IN);
        UNITY_INITIALIZE_VERTEX_OUTPUT_STEREO(o);

        o.pos = UnityObjectToClipPos(IN.pos);
        o.uv0.xy = TRANSFORM_TEX(IN.uv0, _MainTex);

        return o;
    }
 
    half4 frag(v2f IN) : SV_Target
    {
        half4 mainColor = tex2D( _MainTex, IN.uv0.xy );

        clip(mainColor.a - 0.999);

        half3 hairColor = _Color.rgb * mainColor.r;
        half mixStrength = saturate(mainColor.g + _Color2Offset);
        hairColor = lerp(hairColor, _Color2.rgb, mixStrength);

        return half4(hairColor, 1);
    }
    ENDCG          
}
```

代码很简单，最主要的就是下面这句：

```
clip(mainColor.a - 0.999);
```

这里把原来的半透部分尽可能的裁剪掉，只保留实体的轮廓，如下：

![img](/img/fix-hair-transparent/screenshot6.png){:height="70%" width="70%"}

在这个基础之上，再做一次半透明渲染即可。

---

### 局限

采用 **方式二** 后，原先发尾重叠的瑕疵也可以更大程度的缓解了：

![img](/img/fix-hair-transparent/screenshot7.png){:height="70%" width="70%"}

不过这里能良好工作，还是因为我们的半透部分仅局限于 **发尾**，不过这对于做头发表现已经足够了。

最后换个脸妆再来一张：

![img](/img/fix-hair-transparent/screenshot8.png){:height="70%" width="70%"}

拜拜!



