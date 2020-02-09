---
layout: post
title: "关于Unity的Object.name的内存分配"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
  - 内存
---

## 战场玩法的一个热点

年前跑 **仙盟战场** 玩法的时候，发现一处意料之外的 **内存分配**，如下图：

![img](/img/object-name-memory/screenshot1.jpg)

这里 **Object:get_name** 的 **调用频率** 很高，每次调用都有 **内存分配**，虽然是 **临时内存**，但是也会增加 **GC的负担**。

经过定位，发现这个问题是我们游戏的 **换装系统** 引发的。

我们游戏的换装是基于 **共享骨骼** 的实现，每个部位的换装都有一个 **骨骼匹配** 的操作，从而引发 **Transform.name** 的比较操作。

我把上图的 **CopySubPartBones** 函数简化一下，去掉不相关的的代码，大致的流程如下：

```csharp
private Transform FindChildRecursion(Transform root, string name)
{
    foreach (Transform child in root)
    {
    	if (child.name == name)
    	{
    	    return child;
    	}
    	else
    	{
            Transform ret = FindChildRecursion(child, name);

            if (ret != null)
            {
            	return ret;
            }
        }
    }

    return null;
}

private void CopySubPartBones(SkinnedMeshRenderer currentSkin, SkinnedMeshRenderer targetSkin)
{
    Transform[] newBones = new Transform[targetSkin.bones.Length];

    Dictionary<Transform, string> missingBones = new Dictionary<Transform, string>();

    for (int i = 0; i < targetSkin.bones.Length; i++)
    {
    	GameObject bone = targetSkin.bones[i].gameObject;
    	newBones[i] = FindChildRecursion(rootBone.transform, bone.name);
    }

    currentSkin.bones = newBones;
}
```

上述代码每一次 **xxx.name** 的操作，就是一次 **内存分配**。

这里的内存分配是我意料之外的，也就是说，Unity并不会在脚本层缓存 **name**，而是每次都从 **Native** 代码去获取，这就牵扯到 **托管内存/非托管内存** 和 **Marshalling** 的问题，如下图：

![img](/img/object-name-memory/screenshot2.png)

## 题外话：服务器的类似逻辑

说这个内存分配意外，其实也不意外。

比如我们服务器也有类似的逻辑：**Lua** 脚本向 **C++** 请求 **字符串** 数据。

以下面这个 **获取IP地址** 的函数为例：

这是 **C++** 代码：

```c++
// C++
int32_t GameSvr::c_get_server_ip( lua_State* _L )
{
    lcheck_argc( _L, 0 );

    char ip[APP_CFG_NAME_MAX] = {0};
    get_remote_addr()->GetStringIP(ip);
    lua_pushstring( _L, ip);  
    return 1;
}
```

Lua调用如下：

```lua
// Lua
local game_ip = g_gamesvr:c_get_server_ip()
```

假设返回的IP地址是 **127.0.0.1**，如果 **Lua虚拟器** 里没有这个字符串，它就会创建一个新的，这个字符串可能在这次函数调用后就没用了，等待着被 **GC** 的命运。

## 优化

回到第一节换装的 **临时内存分配** 问题。

我们游戏对于模型是有 **缓存池** 的，但是对于换装这个操作，我们却并不缓存。

比如，玩家的出生发型是 **A(20根骨骼)**，进入游戏后换了一套发型 **B(15根骨骼)**，当玩家离开并再次进入游戏时，我们不能直接读取 **B**，而是需要再做一次 **A换B** 的操作。

这样做的原因是，只有出生模型 **A** 才包含了头发的 **全部骨骼**，**B** 只包含了自己 **用到的骨骼**。 

如果我们直接读取 **B**，并以 **B** 为起点再去换 **C(18根骨骼)**，换装就会出问题。

要解决这个问题，我们当然可以要求美术输出的时候，每个部件都输出 **全部骨骼**，不过从资源的角度来说，这个做法稍显浪费。

此外，**Object:get_name** 调用频率很高，除了 **视野加载** 触发以外，还和玩法本身相关，比如玩法要求相同阵营的人统一服装等等。

权衡了一下，我最后的处理方式还是缓存大法，缓存 **骨骼匹配** 的结果：

+ 首先，在我们不得不 **xxx.name** 的时候，确保只取一次，不要对同一根骨骼重复获取。

+ 其次，每个玩家 **缓存** 住最近 **4** 套时装的骨骼数据，不用每次换装都去做骨骼匹配。

这样做不但可以大幅降低 **临时内存分配**，对于 **CPU** 也有帮助，毕竟 **骨骼匹配** 的计算也不少。

最后，我们不用太担心 **缓存** 带来的 **常驻内存增长**，因为对于每个玩家来说，下图红框标记的骨骼还是 **只有一份**，我们缓存的其实是 **骨骼的顺序**，这部分内存不大。

![img](/img/object-name-memory/screenshot3.png)

好了，拜拜！
































































































































