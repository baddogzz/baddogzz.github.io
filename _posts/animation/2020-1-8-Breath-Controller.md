---
layout: post
title: "一个日本人写的插件：Breath Controller"
subtitle: ""
author: "恶毒的狗"
header-mask: 0.2
catalog: true
tags:
  - Unity
  - Animation
  - 小甜甜
---

## Breath Controller

今天无意发现一个日本人写的 **呼吸控制器**，挺好玩的，可以从他的 [主页](http://mebiustos.hatenablog.com/entry/2015/08/31/201902) 下载源代码。

![img](/img/breath-controller/screenshot1.png){:height="70%" width="70%"}

这个插件目前只支持 **人形动画**，不过只需要简单的几行修改就可以支持 **Generic动画** 了，文章的最后会给出代码。

好了，二话不说，先套到我们的 **小甜甜** 身上看看效果：

*听轻音乐*

![img](/img/breath-controller/screenshot2.gif)

*听摇滚*

![img](/img/breath-controller/screenshot3.gif)

## 实现原理

这里是程序控制的呼吸动画，作者区分了 **吸气**，**呼气**，**休息** 三个状态，我们可以调整这3个状态的持续时长：

![img](/img/breath-controller/screenshot6.png)

代码就是不断地循环这3个状态以模拟 **呼吸动画**：

```csharp
void OnInhaling() 
{
    if (this.RotateBone()) 
    {
        this.phase = Phase.Exhaling;
        this.SetEase();
    }
}

void OnExhaling() 
{
    if (this.RotateBone()) 
    {
        this.phase = Phase.Rest;
        this.restEndTime = Time.time + (this.restDuration * this.durationRate);
    }
}

void OnRest() 
{
    this.RotateBone();
    if (this.restEndTime <= Time.time) 
    {
        this.phase = Phase.Inhaling;
        this.SetEase();
    }
}
```

**呼吸动画** 主要涉及 **脊椎**，**胸**，**颈**，**头** 这4根骨骼的旋转计算，如下图：

![img](/img/breath-controller/screenshot4.png)

这里额外标注出了 **左肩** 和 **右肩**，这是因为根据骨骼的父子关系，**脊椎** 或者 **胸** 的运动也会带动 **肩膀** 的运动，作者不希望 **肩膀** 收到呼吸的影响，所以这里在计算完呼吸的运动后会对 **肩膀** 做一个复位操作，伪代码大致如下：

```csharp
void RotateBone() 
 {
    // Backup Shoulder(or UpperArm) rotation.
    var originLeftShoulderRotation = this.LeftShoulder.rotation;
    var originRightShoulderRotation = this.RightShoulder.rotation;

    // Rotate Spine, Cheast, Neck, Head
    // TODO: 旋转脊椎，胸，颈，头        

    // Rotate Shoulder or UpperArm
    this.LeftShoulder.rotation = originLeftShoulderRotation;
    this.RightShoulder.rotation = originRightShoulderRotation;
}
```

好了，下面看一下 **旋转骨骼** 的实现细节。

作者给出的旋转参数不多，最主要的参数是每根骨骼 **吸气** 和 **呼气** 的最大旋转角度，如下图：

![img](/img/breath-controller/screenshot5.png)

这里提到的旋转，作者用了 **Transform.Rotate** 这个函数:

> public void Rotate(Vector3 eulers, Space relativeTo = Space.Self);

> Applies a rotation of eulerAngles.z degrees around the z-axis, eulerAngles.x degrees around the x-axis, and eulerAngles.y degrees around the y-axis (in that order).

旋转主要围绕 **Transform.right** 进行，也就是下图的 **红色轴**：

![img](/img/breath-controller/screenshot7.gif)

旋转的核心代码如下：

```
// Rotate Spine, Cheast, Neck, Head
int finishCnt = 0;
for (int i = 0; i < this.Segments.Length; i++) 
{
    var seg = this.Segments[i];

    if (this.hasController) 
    {
        seg.transform.Rotate(new Vector3(
            seg.x.UpdateEase(this.phase, this.InhalingMethod, this.durationRate),
            seg.y.UpdateEase(this.phase, this.InhalingMethod, this.durationRate),
            seg.z.UpdateEase(this.phase, this.InhalingMethod, this.durationRate)));
    } 
    else 
    {
        var lastEaseValueX = seg.x.lastEaseValue;
        var lastEaseValueY = seg.y.lastEaseValue;
        var lastEaseValueZ = seg.z.lastEaseValue;
        seg.transform.Rotate(new Vector3(
            seg.x.UpdateEase(this.phase, this.InhalingMethod, this.durationRate) - lastEaseValueX,
            seg.y.UpdateEase(this.phase, this.InhalingMethod, this.durationRate) - lastEaseValueY,
            seg.z.UpdateEase(this.phase, this.InhalingMethod, this.durationRate) - lastEaseValueZ));
    }

    if (seg.x.IsFinishEase(this.durationRate) &&
            seg.y.IsFinishEase(this.durationRate) &&
            seg.z.IsFinishEase(this.durationRate)) 
    {
        finishCnt++;
    }
}
```

这里的代码比较简单，唯一要注意的是这里区分了是否有 **Animator**，如果有，**Animator** 的 **Update** 会复位动作，所以这时旋转角度不用减去 **lastEaseValue**。

最后，对于吸气和呼气旋转角度的插值，作者给出了不同的插值算法，并且开放了吸气的插值算法给我们选择：

*正弦插值*

![img](/img/breath-controller/screenshot8.png)

```
float easeOutSine(float t, float b, float c, float d) 
{
    if (t >= d) return c + b;
    return c * Mathf.Sin(t / d * (Mathf.PI / 2)) + b;
}
```

*分段二次插值*

![img](/img/breath-controller/screenshot9.png)

```csharp
float easeInOutQuad(float t, float b, float c, float d) 
{
    if (t >= d) return c + b;
    t /= d / 2;
    if (t < 1) return c / 2 * t * t + b;
    t--;
    return -c / 2 * (t * (t - 2) - 1) + b;
}
```

想象一下 **深吸一口气**，是不是下图的 **正弦插值** 更加合适呢，：）

![img](/img/breath-controller/screenshot10.gif)

## 和DynamicBone一起工作

**Breath Controller** 和 [DynamicBone](https://assetstore.unity.com/packages/tools/animation/dynamic-bone-16743?aid=1101l85Tr) 一样，都是在 **LateUpdate** 里去更新骨骼，如果两者一起工作的时候，我们必须保证 **Breath Controller** 先更新，**DynamicBone** 后更新，不然 **DynamicBone** 就不会对呼吸生效了。

这里我们人为的指定一下脚本执行顺序即可：

![img](/img/breath-controller/screenshot11.png){:height="70%" width="70%"}

## 非人形动画的支持

**Breath Controller** 目前的版本只支持 **人形动画**，如果需要支持 **Generic动画**，我们可以手动指定呼吸计算所需要的骨骼。

这里偷个懒，我在所有 **Animator.GetBoneTransform** 逻辑的后面都加一个判断，如果取不到就用手动指定的骨骼来计算。

最后，代码如下：

```
using UnityEngine;
using System.Collections;

/**
BreathController

Copyright (c) 2015 Toshiaki Aizawa (https://twitter.com/xflutexx)

This software is released under the MIT License.
 http://opensource.org/licenses/mit-license.php …
*/
namespace Mebiustos.BreathController {
    public class BreathController : MonoBehaviour {
        public const float InitialDurationInhale = 1.2f; // 1.3
        public const float InitialDurationExhale = 2.4f; // 2.7
        public const float InitialDurationRest = 0.2f;
        public const float InitialAngleSpineInhale = 2f;
        public const float InitialAngleSpineExhale = -2f;
        public const float InitialAngleChestInhale = -3f;
        public const float InitialAngleChestExhale = 3f;
        public const float InitialAngleNeckInhale = 0.5f;
        public const float InitialAngleNeckExhale = -0.5f;
        public const float InitialAngleHeadInhale = 0.5f;
        public const float InitialAngleHeadExhale = -0.5f;
        public const HalingMethod InitialMethodInhale = HalingMethod.EaseOutSine;

        [System.Serializable]
        public class Segment {
            public HumanBodyBones Bone;

            public Angle x = new Angle();
            public Angle y = new Angle();
            public Angle z = new Angle();

            [System.NonSerialized]
            public Transform transform;
        }

        [System.Serializable]
        public class Angle {
            public float max;
            public float min;
            public float maxDuration;
            public float minDuration;

            float startTime;
            float startValue;
            float changeInValue;
            float durationTime;

            public float lastEaseValue;

            public void SetEase(float startValue, float changeInValue, float durationTime) {
                this.startTime = Time.time;
                this.startValue = startValue;
                this.changeInValue = changeInValue;
                this.durationTime = durationTime;
            }

            public float UpdateEase(Phase status, HalingMethod inhalingMethod, float durationRate) {
                if (status == Phase.Inhaling) {
                    if (inhalingMethod == HalingMethod.EaseOutSine)
                        this.lastEaseValue = easeOutSine(Time.time - this.startTime, this.startValue, this.changeInValue, this.durationTime * durationRate);
                    else
                        this.lastEaseValue = easeInOutQuad(Time.time - this.startTime, this.startValue, this.changeInValue, this.durationTime * durationRate);
                    return this.lastEaseValue;
                } else {
                    this.lastEaseValue = easeInOutQuad(Time.time - this.startTime, this.startValue, this.changeInValue, this.durationTime * durationRate);
                    return this.lastEaseValue;
                }
            }

            public bool IsFinishEase(float durationRate) {
                if (this.durationTime == 0) return true;
                return Time.time - this.startTime >= this.durationTime * durationRate;
            }

            /// <summary>
            /// </summary>
            /// <param name="t">current time</param>
            /// <param name="b">start value</param>
            /// <param name="c">change in value</param>
            /// <param name="d">duration</param>
            /// <returns></returns>
            float easeInOutQuad(float t, float b, float c, float d) {
                if (t >= d) return c + b;
                t /= d / 2;
                if (t < 1) return c / 2 * t * t + b;
                t--;
                return -c / 2 * (t * (t - 2) - 1) + b;
            }
            float easeOutCubic(float t, float b, float c, float d) {
                if (t >= d) return c + b;
                t /= d;
                t--;
                return c * (t * t * t + 1) + b;
            }
            float easeOutQuart(float t, float b, float c, float d) {
                if (t >= d) return c + b;
                t /= d;
                t--;
                return -c * (t * t * t * t - 1) + b;
            }
            float easeInOutQuart(float t, float b, float c, float d) {
                if (t >= d) return c + b;
                t /= d / 2;
                if (t < 1) return c / 2 * t * t * t * t + b;
                t -= 2;
                return -c / 2 * (t * t * t * t - 2) + b;
            }
            float easeOutSine(float t, float b, float c, float d) {
                if (t >= d) return c + b;
                return c * Mathf.Sin(t / d * (Mathf.PI / 2)) + b;
            }
            float easeInOutSine(float t, float b, float c, float d) {
                if (t >= d) return c + b;
                return -c / 2 * (Mathf.Cos(Mathf.PI * t / d) - 1) + b;
            }
            float easeOutExpo(float t, float b, float c, float d) {
                if (t >= d) return c + b;
                return c * (-Mathf.Pow(2, -10 * t / d) + 1) + b;
            }
        }

        [Header("Basic Config")]
        public float durationRate = 1;
        public float effectRate = 1;

        [Header("Generic Bones")]
        public Transform genericLeftShoulder;
        public Transform genericRightShoulder;
        public Transform genericHead;
        public Transform genericNeck;
        public Transform genericSpine;
        public Transform genericChest;

        Segment[] Segments;
        Transform LeftShoulder;
        Transform RightShoulder;
        public enum Phase {
            Inhaling,
            Exhaling,
            Rest
        }
        Phase phase;
        float restEndTime;
        bool hasController;

        void OnEnable() {
            var anim = GetComponent<Animator>();
            this.hasController = anim.runtimeAnimatorController != null;
            if (!this.hasController)
                Debug.LogWarning("Not found 'Animator Controller' : " + this.gameObject.name);

            this.phase = Phase.Inhaling;

            this.InitializeSegments(anim);
            this.InitializeSoulders(anim);

            this.SetEase();
        }

        void LateUpdate() {
            if (this.hasController)
                switch (phase) {
                    case Phase.Inhaling: OnInhaling(); break;
                    case Phase.Exhaling: OnExhaling(); break;
                    case Phase.Rest: OnRest(); break;
                }
        }

        void OnInhaling() {
            if (this.RotateBone()) {
                this.phase = Phase.Exhaling;
                this.SetEase();
            }
        }

        void OnExhaling() {
            if (this.RotateBone()) {
                this.phase = Phase.Rest;
                this.restEndTime = Time.time + (this.restDuration * this.durationRate);
            }
        }

        void OnRest() {
            this.RotateBone();
            if (this.restEndTime <= Time.time) {
                this.phase = Phase.Inhaling;
                this.SetEase();
            }
        }

        /// <summary>
        /// Bone Rotate
        /// </summary>
        /// <returns>IsReadyToNextPhase</returns>
        bool RotateBone() {
            // Backup Shoulder(or UpperArm) rotation.
            var originLeftShoulderRotation = this.LeftShoulder.rotation;
            var originRightShoulderRotation = this.RightShoulder.rotation;

            // Rotate Spine, Cheast, Neck, Head
            int finishCnt = 0;
            for (int i = 0; i < this.Segments.Length; i++) {
                var seg = this.Segments[i];

                if (this.hasController) {
                    seg.transform.Rotate(new Vector3(
                        seg.x.UpdateEase(this.phase, this.InhalingMethod, this.durationRate),
                        seg.y.UpdateEase(this.phase, this.InhalingMethod, this.durationRate),
                        seg.z.UpdateEase(this.phase, this.InhalingMethod, this.durationRate)
                        ));
                } else {
                    var lastEaseValueX = seg.x.lastEaseValue;
                    var lastEaseValueY = seg.y.lastEaseValue;
                    var lastEaseValueZ = seg.z.lastEaseValue;
                    seg.transform.Rotate(new Vector3(
                        seg.x.UpdateEase(this.phase, this.InhalingMethod, this.durationRate) - lastEaseValueX,
                        seg.y.UpdateEase(this.phase, this.InhalingMethod, this.durationRate) - lastEaseValueY,
                        seg.z.UpdateEase(this.phase, this.InhalingMethod, this.durationRate) - lastEaseValueZ)
                        );
                }

                if (seg.x.IsFinishEase(this.durationRate) &&
                    seg.y.IsFinishEase(this.durationRate) &&
                    seg.z.IsFinishEase(this.durationRate)) {
                    finishCnt++;
                }
            }

            // Rotate Shoulder or UpperArm
            this.LeftShoulder.rotation = originLeftShoulderRotation;
            this.RightShoulder.rotation = originRightShoulderRotation;

            // return IsReadyToNextPhase
            return finishCnt >= Segments.Length;
        }

        void SetEase() {
            for (int i = 0; i < this.Segments.Length; i++) {
                var seg = this.Segments[i];
                if (this.phase == Phase.Inhaling) {
                    seg.x.SetEase(seg.x.lastEaseValue, (seg.x.max * this.effectRate) - seg.x.lastEaseValue, seg.x.maxDuration);
                    seg.y.SetEase(seg.y.lastEaseValue, (seg.y.max * this.effectRate) - seg.y.lastEaseValue, seg.y.maxDuration);
                    seg.z.SetEase(seg.z.lastEaseValue, (seg.z.max * this.effectRate) - seg.z.lastEaseValue, seg.z.maxDuration);
                    //Debug.Log("duration:" + seg.z.maxDuration);
                } else {
                    seg.x.SetEase(seg.x.lastEaseValue, (seg.x.min * this.effectRate) - seg.x.lastEaseValue, seg.x.minDuration);
                    seg.y.SetEase(seg.y.lastEaseValue, (seg.y.min * this.effectRate) - seg.y.lastEaseValue, seg.y.minDuration);
                    seg.z.SetEase(seg.z.lastEaseValue, (seg.z.min * this.effectRate) - seg.z.lastEaseValue, seg.z.minDuration);
                    //Debug.Log("duration:" + seg.z.minDuration);
                }
            }
        }

        [Header("Advanced Config")]
        public float maxDuration = BreathController.InitialDurationInhale;
        public float minDuration = BreathController.InitialDurationExhale;
        public float restDuration = BreathController.InitialDurationRest;

        public float SpineInhaleAngle = BreathController.InitialAngleSpineInhale;
        public float SpineExhaleAngle = BreathController.InitialAngleSpineExhale;
        public float ChestInhaleAngle = BreathController.InitialAngleChestInhale;
        public float ChestExhaleAngle = BreathController.InitialAngleChestExhale;
        public float NeckInhaleAngle = BreathController.InitialAngleNeckInhale;
        public float NeckExhaleAngle = BreathController.InitialAngleNeckExhale;
        public float HeadInhaleAngle = BreathController.InitialAngleHeadInhale;
        public float HeadExhaleAngle = BreathController.InitialAngleHeadExhale;
        public enum HalingMethod {
            EaseOutSine,
            EaseInOutQuad
        }
        public HalingMethod InhalingMethod = BreathController.InitialMethodInhale;

        void InitializeSegments(Animator anim) {
            this.Segments = new BreathController.Segment[4];
            BreathController.Segment seg;

            // spine
            seg = new BreathController.Segment();
            seg.Bone = HumanBodyBones.Spine;
            seg.transform = anim.GetBoneTransform(seg.Bone);
            if(seg.transform == null)
                seg.transform = genericSpine;
            this.Segments[0] = seg;

            // chest
            seg = new BreathController.Segment();
            seg.Bone = HumanBodyBones.Chest;
            seg.transform = anim.GetBoneTransform(seg.Bone);
            if (seg.transform == null)
                seg.transform = genericChest;
            this.Segments[1] = seg;

            // neck
            seg = new BreathController.Segment();
            seg.Bone = HumanBodyBones.Neck;
            seg.transform = anim.GetBoneTransform(seg.Bone);
            if (seg.transform == null)
                seg.transform = genericNeck;
            this.Segments[2] = seg;

            // head
            seg = new BreathController.Segment();
            seg.Bone = HumanBodyBones.Head;
            seg.transform = anim.GetBoneTransform(seg.Bone);
            if (seg.transform == null)
                seg.transform = genericHead;
            this.Segments[3] = seg;

            var originRotation = this.transform.rotation;
            this.transform.rotation = Quaternion.identity;

            InitAngleConfig(anim, this.Segments[0], this.SpineInhaleAngle, this.SpineExhaleAngle);
            InitAngleConfig(anim, this.Segments[1], this.ChestInhaleAngle, this.ChestExhaleAngle);
            InitAngleConfig(anim, this.Segments[2], this.NeckInhaleAngle, this.NeckExhaleAngle);
            InitAngleConfig(anim, this.Segments[3], this.HeadInhaleAngle, this.HeadExhaleAngle);

            this.transform.rotation = originRotation;
        }

        enum vect {forward, right, up};
        void InitAngleConfig(Animator anim, Segment segment, float inhaleAngle, float exhaleAngle) {
            var btra = anim.GetBoneTransform(segment.Bone);

            if(btra == null)
            {
                if(segment.Bone == HumanBodyBones.Chest)
                {
                    btra = genericChest;
                }
                else if(segment.Bone == HumanBodyBones.Neck)
                {
                    btra = genericNeck;
                }
                else if(segment.Bone == HumanBodyBones.Head)
                {
                    btra = genericHead;
                }
                else if(segment.Bone == HumanBodyBones.Spine)
                {
                    btra = genericSpine;
                }
            }

            var forwardDot = Vector3.Dot(transform.right, transform.InverseTransformDirection(btra.forward));
            var rightDot = Vector3.Dot(transform.right, transform.InverseTransformDirection(btra.right));
            var upDot = Vector3.Dot(transform.right, transform.InverseTransformDirection(btra.up));

            //Debug.Log("---- " + this.gameObject.name + " (" + btra.gameObject.name + ")");
            //Debug.Log("Forward Dot:" + forwardDot);
            //Debug.Log("Right   Dot:" + rightDot);
            //Debug.Log("Up      Dot:" + upDot);

            float min = 1;
            vect bestvec = 0;
            float machv;

            machv = 1 - Mathf.Abs(forwardDot);
            if (machv < min) {
                bestvec = vect.forward;
                min = machv;
            }

            machv = 1 - Mathf.Abs(rightDot);
            if (machv < min) {
                bestvec = vect.right;
                min = machv;
            }

            machv = 1 - Mathf.Abs(upDot);
            if (machv < min) {
                bestvec = vect.up;
                min = machv;
            }

            switch (bestvec) {
                case vect.forward:
                    segment.z.max = inhaleAngle * Mathf.Sign(forwardDot);
                    segment.z.min = exhaleAngle * Mathf.Sign(forwardDot);
                    segment.z.maxDuration = this.maxDuration;
                    segment.z.minDuration = this.minDuration;
                    break;
                case vect.right:
                    segment.x.max = inhaleAngle * Mathf.Sign(rightDot);
                    segment.x.min = exhaleAngle * Mathf.Sign(rightDot);
                    segment.x.maxDuration = this.maxDuration;
                    segment.x.minDuration = this.minDuration;
                    break;
                case vect.up:
                    segment.y.max = inhaleAngle * Mathf.Sign(upDot);
                    segment.y.min = exhaleAngle * Mathf.Sign(upDot);
                    segment.y.maxDuration = this.maxDuration;
                    segment.y.minDuration = this.minDuration;
                    break;
            }
        }

        private void InitializeSoulders(Animator anim) {
            this.LeftShoulder = anim.GetBoneTransform(HumanBodyBones.LeftShoulder);
            this.RightShoulder = anim.GetBoneTransform(HumanBodyBones.RightShoulder);

            if (LeftShoulder == null)
                this.LeftShoulder = anim.GetBoneTransform(HumanBodyBones.LeftUpperArm);

            if (LeftShoulder == null)
                this.LeftShoulder = genericLeftShoulder;
            
            if (RightShoulder == null)
                this.RightShoulder = anim.GetBoneTransform(HumanBodyBones.RightUpperArm);

            if (RightShoulder == null)
                this.RightShoulder = genericRightShoulder;
        }
    }
}
```

好了，拜拜！