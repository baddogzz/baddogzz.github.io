---
layout: post
title: "关于自定义shader的自发光烘培"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
  - Shader
---

## 一个bug

最近正在做交接工作，被问到了一个问题，如下图：

![img](/img/emission-bake/screenshot1.png)

左边这个球用的是Unity内置的Standard shader，开启了红色的自发光。可以看到，烘培后，自发光照亮了地面。

![img](/img/emission-bake/screenshot2.png)

右边这个球用的是我们自定义的shader，在 **Meta Pass** 我们同样设置了 **Emission**，并且为了调试，我们把 **Albedo** 设置成了纯黑色，并且简化了自发光的计算，但是烘培的结果，自发光并未对地面产生影响。

![img](/img/emission-bake/screenshot3.png)

```
half4 FragMeta (v2f i) : SV_Target 
{
    float2 mainUV = i.uvTex0;
    half4 mainColor = tex2D(_MainTex, mainUV);
    half3 emissionColor = mainColor.rgb * _EmissionColor.rgb * _EmissionFactor * mainColor.a;
    half3 specularColor = mainColor.a * _SpecularFactor * _SpecularColor;

    UnityMetaInput metaIN;
    UNITY_INITIALIZE_OUTPUT(UnityMetaInput, metaIN);
    metaIN.Albedo = mainColor * 0;
    metaIN.Emission = _EmissionColor.rgb * _EmissionFactor;
    metaIN.SpecularColor = specularColor;
    return UnityMetaFragment(metaIN);
}
``` 

## 原因

搜了一下，发现也有人遇到了同样的问题，链接如下：

[My Emissive Material/Shader Does Not Appear In The Lightmap.](https://support.unity3d.com/hc/en-us/articles/214718843-My-Emissive-material-shader-does-not-appear-in-the-Lightmap-)

文章给出了解决方案：如果是自定义的shader，我们需要特别设置一下材质的GI属性，代码如下：

```
using UnityEngine;
using UnityEditor;

public class LightMapMenu : MonoBehaviour
{
    // The menu item.
    [MenuItem ("MyCustomBake/Bake")]
    
    static void Bake ()
    {
        // Find all objects with the tag <Emissive_to_baked>
        // We have to set the tag “Emissive_to_baked” on each GO to be baked.
        GameObject[] _emissiveObjs = GameObject.FindGameObjectsWithTag("Emissive_to_baked");

        // Then, by each object, set the globalIllumiationFlags to BakedEmissive.
        foreach (GameObject tmpObj in _emissiveObjs)
        {
            Material tmpMaterial = tmpObj.GetComponent<Renderer> ().sharedMaterial;
            tmpMaterial.globalIlluminationFlags = MaterialGlobalIlluminationFlags.BakedEmissive;
        }

        // Bake the lightmap.
        Lightmapping.Bake ();
    }
}
```

这里的GI属性，我们选择 **MaterialGlobalIlluminationFlags.BakedEmissive** 即可。

关于 **MaterialGlobalIlluminationFlags.BakedEmissive**，官方文档说明如下：

> The emissive lighting affects baked Global Illumination. It emits lighting into baked lightmaps and baked lightprobes.

设置了GI属性后，我们可以看到，左右2个球的材质都出现在了 **Light Explorer** 的 **Static Emissives** 面板中：

![img](/img/emission-bake/screenshot4.png)

并且右边的自发光也参与了烘培：

![img](/img/emission-bake/screenshot5.png)

## 其他

对比修改前后的材质文件，我们发现主要就是 **m_LightmapFlags** 的设置发生了改变：

![img](/img/emission-bake/screenshot6.png)

如果不想写代码，我们也可以直接修改材质文本中的 **m_LightmapFlags** 的设值，这里的设置和 **MaterialGlobalIlluminationFlags** 枚举值是一一对应的：

![img](/img/emission-bake/screenshot7.png)

好了，拜拜。


























































































































































































































































