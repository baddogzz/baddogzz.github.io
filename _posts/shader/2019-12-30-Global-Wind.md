---
layout: post
title: "面对疾风吧"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
  - Shader
  - 小甜甜
---

### 面对疾风吧

趁着年前有空，把 **小甜甜** 项目当初没做好的暴风雨效果迭代了一遍，录了个视频，如下：

<iframe src="https://player.bilibili.com/player.html?aid=81329772&cid=139188512&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" width="640" height="430" allowfullscreen="true"> </iframe>

是不是很有 **塞尔达** 的感觉呢，：）

做法也不难，主要操作如下：

+ 添加全局风
+ 修改草的摆动，使其受全局风影响
+ 打开雨雾粒子的受力选项，使其受全局风的影响
+ 扩展DynamicBone的摆动，使其受全局风影响

---

### 添加全局风

要让风的表现统一，这里还是借用了Unity的 **WindZone**。

不过针对 **草** 以及 **人物头发和裙摆** 的摆动，这里只是取 **WindZone** 的朝向以及 **Main/Turbulence** 两个参数，摆动算法自己搞。

+ Main：控制主风的强度
+ Turbulence：控制疾风的强度

和 [Lux LWRP Essentials](https://assetstore.unity.com/packages/vfx/shaders/lux-lwrp-essentials-150355?aid=1101l85Tr) 的风做法一致，我们也会生成全局的 **Wind Texture** 用于控制风的强弱变化，具体细节可以参考前文：[Lux的风和WindTexture](https://baddogzz.github.io/2019/12/06/Lux-Wind-Texture/)。

---

### 草随风摆动

草的摆动也很简单：在顶点着色器采样 **Wind Texture**，得到当前位置风的强度，根据风的强度计算出xz偏移，把偏移和顶点位置叠加即可完成摆动。具体细节可以参考前文：[Lux的风和WindTexture](https://baddogzz.github.io/2019/12/06/Lux-Wind-Texture/)。

---

### 雨雾受风

雨雾是粒子，Unity的粒子系统本身就支持 **受力**，勾上 **External Forces** 即可。

![img](/img/global-wind/screenshot1.jpg){:height="90%" width="90%"} 

PS，注意这里的性能开销。

---

### DynamicBone随风摆动

本文主要介绍一下 [DynamicBone](https://assetstore.unity.com/packages/tools/animation/dynamic-bone-16743?aid=1101l85Tr) 的随风摆动。

[DynamicBone](https://assetstore.unity.com/packages/tools/animation/dynamic-bone-16743?aid=1101l85Tr) 是一款很出名的Unity插件，我们可以把需要应用 **物理** 的骨骼挂上 **DynamicBone** 组件，调整物理参数来满足我们的摆动需求。下图是 **小马哥** 新做的 **丑男2号** 添加 **DynamicBone** 后的效果：
 
![img](/img/global-wind/screenshot2.gif){:height="60%" width="60%"} 

查看 **DynamicBone** 的代码，我们可以发现，作者预留了 **受力** 的选项，只是外力的初始值都是 **0**。

```csharp
#if UNITY_5
    [Tooltip("The force apply to bones.")]
#endif
    public Vector3 m_Force = Vector3.zero;
```

我们要做的很简单：根据风力，调整 **m_Force** 的值，剩下的交给 **DynamicBone** 就可以了。

这里简单模仿一下 [Lux LWRP Essentials](https://assetstore.unity.com/packages/vfx/shaders/lux-lwrp-essentials-150355?aid=1101l85Tr) 关于 **Wind Texture** 的生成算法：利用多层噪音叠加，合成风的强度变化。

关于噪音的合成，这里做了简化处理：用2层噪音合成主风，再外加1层噪音给疾风。合成的方式也做了简化处理，效果差不多够用，代码如下：

```csharp
using UnityEngine;

[RequireComponent(typeof(WindZone))]
public class DynamicBoneWind : MonoBehaviour
{
    public float mainWindScale1;
    public float mainWindScale2;
    public float gustyWindScale;

    private WindZone m_WindZone;
    private DynamicBone[] m_DynamicBones;

    private void OnEnable()
    {
        m_WindZone = GetComponent<WindZone>();

        UpdateDynamicBones();
    }

    public void Update()
    {
        float mainWind = m_WindZone.windMain;
        float turbulence = m_WindZone.windTurbulence;

        float mainWindNoise1 = Mathf.PerlinNoise(Time.time * mainWindScale1, 0);
        float mainWindNoise2 = Mathf.PerlinNoise(Time.time * mainWindScale2, 0);
        float mainWindNoise = mainWindNoise1 * 0.7f + mainWindNoise2 * 0.3f;

        float gustNoise = Mathf.PerlinNoise(Time.time * gustyWindScale, 0);
        gustNoise = Mathf.Lerp(1.0f, gustNoise, turbulence);
        gustNoise = (gustNoise - 0.5f) * (turbulence + 0.5f) + 0.5f;
        gustNoise = Mathf.Clamp(gustNoise, 0, 1.0f);

        float finalWindForce = mainWind * mainWindNoise * (gustNoise * 2 - 0.243f);

        for (int i = 0; i < m_DynamicBones.Length; i++)
        {
            m_DynamicBones[i].m_Force = transform.forward * finalWindForce;
        }
    }

    public void UpdateDynamicBones()
    {
        m_DynamicBones = FindObjectsOfType<DynamicBone>();
    }
}
```

这里依然从 **WindZone** 获取参数，以保证人物的摆动和场景的摆动一致。

最后，那些说 **小甜甜** 的裙子为什么没被吹起的同学，哥满足你：

![img](/img/global-wind/screenshot3.gif){:height="60%" width="60%"} 

拜拜。






