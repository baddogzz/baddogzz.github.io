---
layout: post
title: "关于AssetBundle的卸载"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
  - 资源管理
---

## 关于AssetBundle的卸载

又坐了一天月子，继续写文章找状态。

本文是关于 **卸载AssetBundle** 的一些知识点。

下图是一个最简单的从 **AssetBundle** 加载 **Asset** 并 **实例化** 的流程：

![img](/img/unload-resources/screenshot1.png)

这里的 **Bundle** 在加载完 **资源A** 后就没用了，我们可以通过 **AssetBundle.Unload(false)** 把它卸掉，只保留住 **资源A**。

如果 **资源A** 也没用了，我们可以通过 **Destroy** 接口或者 **Resources.UnloadAsset** 接口销毁它。

题外话，我们需要注意一下 **Resources.UnloadAsset** 的应用场合：

> This function can only be called on Assets that are stored on disk.

> The referenced asset (assetToUnload) will be unloaded from memory. The object will become invalid and can't be loaded back from disk. Any subsequently loaded Scenes or assets that reference the asset on disk will cause a new instance of the object to be loaded from disk. This new instance will not be connected to the previously unloaded object.

**UWA** 也有相关的回答：

> Resources.UnloadAsset仅能释放非GameObject和Component的资源，比如Texture、Mesh等真正的资源。对于由Prefab加载出来的Object或Component，则不能通过该函数来进行释放。

好了，回到上图。

上图描述的场景过于简单，实际项目中，**资源A** 可能依赖 **其他资源**，并且 **其他资源** 又被打进 **不同的Bundle** 中，如下图：

![img](/img/unload-resources/screenshot2.png)

这个时候，卸载 **AssetBundle** 就需要一定的 **策略** 了。

在介绍 **卸载策略** 之前，我们必须先了解清楚 **AssetBundle.Unload** 这个函数。 

## 关于AssetBundle.Unload

Unity官方文档对于 **AssetBundle.Unload** 的描述如下：

```csharp
public void Unload(bool unloadAllLoadedObjects);
```

> Unloads assets in the bundle.

> When unloadAllLoadedObjects is false, compressed file data for assets inside the bundle will be unloaded, but any actual objects already loaded from this bundle will be kept intact. Of course you won't be able to load any more objects from this bundle.

> When unloadAllLoadedObjects is true, all objects that were loaded from this bundle will be destroyed as well. If there are GameObjects in your Scene referencing those assets, the references to them will become missing.

+ **AssetBundle.Unload(false)** 会把 **Bundle** 卸载，但是已经从 **Bundle** 里加载出来的 **资源** 是不会被卸载的。

+ **AssetBundle.Unload(true)** 不但会卸载 **Bundle**，也会卸载已经从 **Bundle** 里加载出来的 **所有资源**，哪怕这些 **资源** 还被引用着。

对于用户来说，如果选择 **AssetBundle.Unload(true)**，用户必须确保 **Bundle** 中已经加载的 **资源** 是没有被引用的，否则就会发生 **资源丢失**。

如果选择 **AssetBundle.Unload(false)**，用户就要承担起卸载 **已加载资源** 的责任，如果处理不当，就可能造成 **资源重复**，如下图：

![img](/img/unload-resources/screenshot3.png)

最后，Unity提供了一个 **Resources.UnloadUnusedAssets** 接口帮助我们销毁没有任何引用的 **野资源**，不过这个函数会扫描全部对象，开销较大，一般只在 **切场景** 时调用。

## 卸载AssetBundle的策略

了解 **AssetBundle.Unload** 的行为后，再来看一下我们采用过的策略。

#### 暗黑血统的策略

最早做 **暗黑血统** 的时候，我们卸载 **AssetBundle** 的策略如下：

+ 用 **AssetBundle.Unload(false)** 来卸载 **Bundle**。

+ 加载完资源后立即卸载 **叶子节点的Bunlde**，这里 **叶子节点** 表示 **没有被其他Bundle所依赖的Bundle**。

+ 对于 **非叶子节点的Bundle**，卸载逻辑完全依靠 **引用计数**。

以下图为例：

![img](/img/unload-resources/screenshot4.png)

红框标注的 **Bundle** 是可以在加载完 **资源** 后立刻卸载的。

我们看一下包含 **资源A和B的Bundle**，如果我们只加载了 **A**，然后就把 **Bundle** 卸载了，然后我们再加载 **B**，这个时候 **Bundle** 又要被重新加载，如果我再从这个 **Bundle** 加载 **A**，这个时候不是就有 **2个A** 了？

事实上，因为 **第一个A** 依然还被我们的 **AssetManager** 管理着，上层逻辑不会直接从 **Bundle** 中去加载 **A**，而是从缓存中去拿，所以上述情况并不会出现。

我们再看一下包含 **资源C的Bundle**，如果我们加载完 **C** 后就把 **Bundle** 卸载了，然后我们再去加载 **G**，由于 **G依赖C**，**C所在的Bundle** 会再次被加载，同时加载出一个 **新的C**，这就是真正的 **资源重复** 了。

此外，因为我们的 **AssetManager** 只会管理 **直接加载的资源**，假设我们先加载了 **G**，**C** 做为 **G** 的依赖被 **间接加载**，此时我们再去直接加载 **C** 就无法命中 **AssetManager** 的缓存了。对于这种情况，**AssetManager** 必须做一些额外记录，稍微有点蛋疼。

最后，考虑一下 **引用计数**，假设我们已经销毁了 **G**，那么 **Ｇ** 依赖的所有Bunlde引用计数会-1，假设 **包含D的Bundle** 以及 **包含H的Bundle** 引用计数都为 **０** 了，选择 **AssetBundle.Unload(false)** 卸载的结果是 **被间接加载的D和H** 在 **Bundle卸载** 后依然存活着，我们的 **AssetManager** 并没有管理到它们，它们变成了 **野资源**。

这一类 **野资源** 最终得依靠 **Resources.UnloadUnusedAssets** 来卸载，一般我们在 **场景切换** 时做这个操作。

#### 当前项目的策略

暗黑血统的策略在线上工作良好，不过因为 **AssetManager** 只会管理 **直接加载** 的资源，**AssetBundle.Unload(false)** 必须配合 **Resources.UnloadUnusedAssets** 才能完成一次彻底的资源清除。

当前项目，**AssetManager** 依然只管理 **直接加载** 的资源，不过我选择了 **AssetBundle.Unload(true)** 的策略，并且不再区分 **Bundle是否是叶子节点**，一切卸载的依据都是 **引用计数**。

正所谓 **Bundle在，资源在，Bundle亡，资源亡**，：）

以下图为例，最左边一列会标记出每个 **Bundle** 的引用计数：

![img](/img/unload-resources/screenshot5.png)

当我们销毁 **资源A**，引用计数变化如下：

![img](/img/unload-resources/screenshot6.png)

当我们再销毁 **资源B**，部分 **Bundle** 的引用计数变为0，调用 **AssetBundle.Unload(true)** 就能把资源及时清理干净了，如下图：

![img](/img/unload-resources/screenshot7.png)

只要引用计数没问题，理论上 **Resources.UnloadUnusedAssets** 也就不用了。

最后，在实际项目中，并非 **AssetBundle** 卸载得越及时就越好，比如反复打开关闭一个UI，我们不需要一关闭UI就销毁 **资源** 进而卸载 **AssetBundle**，而是需要在 **资源** 上层做一些缓存策略。

## 结尾

本文就到这里，今年有机会也可以试试Unity推荐的 [Addressable Assets](https://docs.unity3d.com/Packages/com.unity.addressables@1.6/manual/index.html)。

好了，拜拜。





























































































































