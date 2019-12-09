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
  - 小甜甜
---

### Lux的风和WindTexture

玩 [Lux LWRP Essentials](https://assetstore.unity.com/packages/vfx/shaders/lux-lwrp-essentials-150355?aid=1101l85Tr) 的过程中发现了挺多好玩的东西，比如它的风和草的随风摆动，实现细节就挺有趣。

下图是前项目草的摆动效果：

![img](/img/lux-wind/screenshot2.gif){:height="75%" width="75%"}

这里其实没有风，草的摆动计算比较简单，就是周期性的正余弦摆动。 效果还不错，计算量也不算大。

如果用 **Lux的风** 来驱动草的摆动，效果如下：

![img](/img/lux-wind/screenshot1.gif){:height="75%" width="75%"}

摆动不再那么规则，并且风的方向可以调整，表现更加真实。

下面就来看一下它的实现细节。

---

### 实现细节

#### Wind Texture

**Lux Grass** 的摆动是被一张全局的 **Wind Texture** 驱动的，我们可以看一下这张 **Wind Texture** 长什么模样。

![img](/img/lux-wind/screenshot3.gif){:height="40%" width="40%"}

**Wind Texture** 看上去像一张有uv动画的 **噪声图**，它定义了风的强度变化。 

#### 草的摆动

草的摆动很简单：在顶点着色器采样 **Wind Texture**，结合风的方向计算出摆动的幅度xz，和顶点位置相加即可。

```
half4 wind = SAMPLE_TEXTURE2D_LOD(_LuxLWRPWindRT, sampler_LuxLWRPWindRT, positionWS.xz * _LuxLWRPWindDirSize.w + phase * _WindMultiplier.z, _WindMultiplier.w);                

half3 windDir = _LuxLWRPWindDirSize.xyz;
half windStrength = bendAmount * _LuxLWRPWindStrengthMultipliers.x * _WindMultiplier.x;

/* not a "real" normal as we want to keep the base direction */
wind.r = wind.r   *   (wind.g * 2.0h - 0.243h);
windStrength *= wind.r;
positionWS.xz += windDir.xz * windStrength;
```

#### Wind Texture 的生成

**Wind Texture** 是全局的，这样风的计算可以独立出来，不用每个顶点着色器都去各自计算。

要生成 **Wind Texture**，我们需要Unity的 **WindZone** 组件，同时添加 **LuxLWRP_Wind** 脚本，这个脚本会每帧去更新 **Wind Texture** 的信息。

![img](/img/lux-wind/screenshot4.jpg){:height="70%" width="70%"}

脚本的参数不多，比较重要的是它需要一张 **Wind Base Tex**：

![img](/img/lux-wind/screenshot5.jpg){:height="50%" width="50%"}

**Wind Base Tex** 的四个通道分别是 **不同scale的perlin noise**，**RGB** 通道主要用于计算 **Wind Strength**， **A** 通道不但影响 **Wind Strength**，更重要的是计算 **Wind Gusting**。

![img](/img/lux-wind/screenshot6.jpg){:height="50%" width="75%"}

注意这里的 **Wind Strength** / **Wind Gusting** 和 **WindZone** 的 **windMain** / **windTurbulence** 是对应的，一个表示 **主风**，一个表示 **急风**。

合成 **Wind Texture** 的代码比较简单，各个通道采样后做混合，生成最终结果。 **Wind Texture** 的R通道保存的是 **Wind Strength** 的计算结果，B通道保存的是 **Wind Gusting** 的计算结果。

```
half4 frag(VertexOutput i) : SV_Target 
{
  half4 n1 = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, i.uv + _LuxLWRPWindUVs);
  half4 n2 = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, i.uv + _LuxLWRPWindUVs1);
  half4 n3 = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, i.uv + _LuxLWRPWindUVs2); 
  half4 n4 = SAMPLE_TEXTURE2D(_MainTex, sampler_MainTex, i.uv * _LuxLWRPGust.x + _LuxLWRPWindUVs3);

  half4 sum = half4(n1.r, n1.g + n2.g, n1.b + n2.b + n3.b, n1.a + n2.a + n3.a + n4.a);
  const half4 weights = half4(0.5000, 0.2500 , 0.1250 , 0.0625 );
                
  half2 WindStrengthGustNoise;
  //  WindStrength
  WindStrengthGustNoise.x = dot(sum, weights);
  //  GrassGustNoise / _LuxLWRPGust.y comes in as 0.5 - 1.5                                 
  WindStrengthGustNoise.y = lerp(1.0h, (n4.a + dot(half3(n1.a, n2.a, n3.a), _GustMixLayer)) * 0.85h, _LuxLWRPGust.y - 0.5h);
  //  Sharpen WindStrengthGustNoise according to turbulence
  WindStrengthGustNoise = (WindStrengthGustNoise - half2(0.5h, 0.5h)) * _LuxLWRPGust.yy + half2(0.5h, 0.5h);

  return half4(WindStrengthGustNoise, (n3.a + abs(WindStrengthGustNoise.y)) * 0.5h + n2.a * 0.0h, 0);
}

```

**Lux Grass** 最终采样 **Wind Texture** 的时候，会把 **Wind Gusting** 应用到 **Wind Strength**，这里有点类似 **解码法线贴图**， 但是为了保证 **急风不要完全打乱主风的方向**，作者用了一个magic number： **0.243**。

```
 /* not a "real" normal as we want to keep the base direction */ 
 wind.r = wind.r  *  (wind.g * 2.0h - 0.243h );
```

#### 草的位置和 **Wind Texture** 的对应关系 

**LuxLWRP_Wind** 有一个参数 **Size In World Space**，用来决定 **Wind Texture** 覆盖的世界空间的xz范围。 

草的 **世界坐标xz** 除以 **Size In World Space** 即可换算成 **UV坐标**，用于 **Wind Texture** 的采样，从而得到当前位置风的强度信息。

```
half4 wind = SAMPLE_TEXTURE2D_LOD(_LuxLWRPWindRT, sampler_LuxLWRPWindRT, vertexInput.positionWS.xz * _LuxLWRPWindDirSize.w + phase * _WindMultiplier.z, _WindMultiplier.w);

```

上面代码中的 **_LuxLWRPWindDirSize** 向量，其xyz保存的是风的方向，w保存的是 **1 / Size In World Space** 的结果。 **vertexInput.positionWS.xz * _LuxLWRPWindDirSize.w** 即顶点的 **世界坐标** 换算成 **Wind Texture的UV坐标**。

**Wind Texture** 的 **wrapMode** 是 **Repeat**，**Lux Wind** 会按照 **Size In World Space** 定义的大小在场景内重复。

---


### 参考

更多细节可以直接参考 [Lux LWRP Essentials](https://assetstore.unity.com/packages/vfx/shaders/lux-lwrp-essentials-150355?aid=1101l85Tr) 的代码。

此外，关于 **Wind Texture的合成**， 作者在他的Shader中也给出了一个参考：[dynamic clouds](http://www.iquilezles.org/www/articles/dynclouds/dynclouds.htm)。

好了，拜拜。



