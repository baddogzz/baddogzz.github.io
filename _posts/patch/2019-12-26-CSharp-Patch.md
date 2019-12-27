---
layout: post
title: "C#热更新方案的选择"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
  - 热更新
---

### 前项目C#热更方案的选择

#### 小甜甜的C#热更方案

前段时间 **noodle** 说他把 **小甜甜** 项目中他做的 [C#热更方案](https://github.com/noodle1983/UnityAndroidIl2cppPatchDemo/) 开源了。

这个方案是一个 **骚操作**，不过是针对 **il2cpp** 的，核心思想是更新 **libil2cpp.so**，具体细节可以参考 [github主页](https://github.com/noodle1983/UnityAndroidIl2cppPatchDemo/)。

#### 暗黑血统的C#热更方案

再早一点的项目 **暗黑血统**，那还是 **Unity4** 的时代，我们的C#热更方案也是一个 **骚操作**，这里列一下要点：

1. **Assembly-CSharp-firstpass.dll** 和 **Assembly-CSharp.dll** 是Unity预定的2个程序集。

2. **Assembly-CSharp-firstpass.dll** 包括了加载热更代码的代码，不可被热更新，**Assembly-CSharp.dll** 包括主要的游戏逻辑代码，期望可以被热更新。

3. 打包的时候，把 **Assembly-CSharp.dll** 中的代码移动到我们自定的 **GameLogic.dll** 中，并把 **Assembly-CSharp.dll** 的代码清空(namespace颠倒)。

4. 运行的时候，**Assembly-CSharp-firstpass.dll** 中的代码通过 **Assembly.Load** 的方式去加载 **GameLogic.dll**，**GameLogic.dll** 可以从服务器下载获取，以此达到热更新的目的。

这样做看起来OK，但是有一个很大的限制： **预设不能挂载非firstpass目录的脚本**，原因可以参考[这篇帖子](https://blog.csdn.net/gz_huangzl/article/details/52486509)。 当然，我们可以在运行时通过 **AddComponent** 的方式去挂载脚本，但是这样做限制较大。

**骚操作** 之所以被称为 **骚操作**，就是我们可以打破这个限制：即把 **Assembly-CSharp.dll** 换成了 **GameLogic.dll** 后，也要保证预设能够找得到原先引用的脚本。

**暗黑血统** 的做法是：**在生成GameLogic.dll后，改cs文件对应的meta文件，把dll重新定向到GameLogic.dll，重启编辑器再打包**。

![img](/img/code-patch/screenshot1.png){:height="70%" width="70%"}

在打包的时刻，预设已经认定了 **GameLogic.dll**，所以加载时就不会丢失脚本了。

当然，这个方案依然也有局限：

1. 必须严格保证 **Assembly-CSharp-firstpass.dll** 的稳定，一旦出现了问题，只能换包。

2. 热更后，如果 **GameLogic.dll** 新增了一个上架包中并不存在的脚本，那么挂载这个脚本的预设在加载时依然还是会出现脚本丢失。 

总体来说，这套方案没什么大问题，在线上运行良好。出现上面的问题2时，我们就 **AddComponent** 绕一下。 

随着Unity的升级换代，metadata的格式也在变化，这套依赖 **改meta文件** 的方案在版本兼容性上出现了很大问题，最终因为难以维护被抛弃了。

---

### 目前的C#热更新方案

时至今日，如果再做方案选择，我倾向于集成 [xLua](https://github.com/Tencent/xLua)。

对于Android平台的C#热更，我倾向于目前公司所采用的方案：**自己编译libmono.so**。

我们可以在github上找到各个Unity版本对应的 [mono源码](https://github.com/Unity-Technologies/mono)，打开 **image.c** 文件，找到 **mono_image_open_from_data_with_name** 函数，截住加载 **Assembly-CSharp-firstpass.dll** 的逻辑，做我们自己的操作。

主要代码流程如下：

```
MonoImage *
mono_image_open_from_data_with_name (char *data, guint32 data_len, gboolean need_copy, MonoImageOpenStatus *status, gboolean refonly, const char *name)
{   
    int datasize = 0; 

    if(name != NULL && strstr(name,"Assembly-CSharp-firstpass.dll"))
    {
    	// 从我们的Patch目录读取我们自己的dll文件，覆盖传入的data  
    }

    // 解密data

    // 走原先的do_mono_image_load流程
}
```

这里有2个细节要注意：

1. **Assembly-CSharp-firstpass.dll** 和 **Assembly-CSharp.dll** 我们都做了加密，所以这里有一个步骤是解密。

2. **mono_image_open_from_data_with_name** 函数只处理了 **Assembly-CSharp-firstpass.dll**。至于 **Assembly-CSharp.dll**，我们打包的时候直接把他删了。和 **暗黑血统** 的做法类似，我们还是通过 **Assembly-CSharp-firstpass.dll** 中的代码来加载它。

改完源码，重新编译生成新的 **libmono.so**，就大功告成了。

这个方案也比较成熟，网上的文章一搜一大把。我之所以倾向于这个方案，主要有以下几点考虑：

1. 这个方案对整个项目的 **侵入性很小**，只需要替换掉 **libmono.so** 即可。

2. 加载热更代码的代码从 **Assembly-CSharp-firstpass.dll** 转移到了 **libmono.so**，因此 **Assembly-CSharp-firstpass.dll** 也可以被热更新了。

3. 这个方案对团队人员的要求没那么高，维护成本相对较低。

当然，这个方案也有以下一些限制： 

1. 如果更新 **Assembly-CSharp-firstpass.dll**，需要重启一次进程。

2. **Standard Assets** 或者 **Plugins** 目录下的代码可以被挂载，但是 **非firstpass目录** 下的代码不行，因为这里并没有 **暗黑血统改meta** 的那一步骚操作。

重启进程对用户体验有一点伤害，特别是 **进程不能被快速拉起** 时，可能会影响留存。

不过后来我们在sdk里加了一个 **秒启** 的函数，现在重启的代价可以忽略不计了，代码如下：

```
public void doRestartApp()
{
    new Thread()
    {
        public void run()
        {
            Intent localIntent = mContext.getPackageManager().getLaunchIntentForPackage(mContext.getPackageName());
            localIntent.addFlags(67108864);
            mContext.startActivity(localIntent);
            android.os.Process.killProcess(android.os.Process.myPid());
        }
    }.start();

    finish();
}
```

对于 **非firstpass目录** 下代码无法挂载的问题，我们会把需要挂载的代码统一移动到 **Plugins** 目录下，因为 **Assembly-CSharp-firstpass.dll** 已经可以被热更新了。

好了，拜拜。
































