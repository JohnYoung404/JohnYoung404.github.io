---
layout:     post
title:      从Unity源码理解Monobehaviour的生命周期
subtitle:   Update一定比LateUpdate先执行吗?
date:       2018-12-06
author:     John Young
header-img: img/tag-bg-o.jpg
catalog: true
tags:
    - Unity
    - MonoBehaviour
---

## 前言

Unity的Monobehaviour每个Unity程序员都不可能陌生，update, fixedUpdate, lateUpdate等等built-in函数也是信手拈来。Unity间程序员流传的一张MonoBehaviour的图更是被奉为圭臬。
然而这篇文章的结论可能会超过你的想象：一些看似self-explanatory的接口，可能并不按你想象的顺序执行。

而这一切，都是从Unity的源码中得到的答案。

### 一般认为的MonoBehaviour的生命周期

这张图可能入门Unity的时候大家都见过，当你对MonoBehaviour的执行顺序有疑问时，也往往会搜到这张图：
![](https://johnyoung404.github.io/img/monoBehaviour/lifeCycle.png)
其中比较重要的几个信息：

* 当MonoBehaviour被实例化时，会依次执行Awake、OnEnable、Start(如果没被调用过)
* 接着会依次执行FixedUpdate(1至n次，取决于fixed time step), Update和LateUpdate
* 其中一些协程的yield result也会在如图位置被调用

如果你不想深入了解这背后的机制，那这张图已经足够了，因为它描述了99%的事实。
但是如果你手头有Unity源码，并且研究过相关的实现，你可能会有不一样的理解。

### 一个小问题：这个脚本挂在物体上，会在console输出什么结果？

```C#
using System.Collections.Generic;
using UnityEngine;

public class GO : MonoBehaviour {

    bool loaded = false;
    // Use this for initialization
    void Start () {
        loaded = true;
    }
    void Update () {
        loaded = false;
    }
    void LateUpdate()
    {
        if(loaded)
        {
            Debug.Log("Eureka!");
        }
    }
}
```
这个问题其实有点tricky，因为在不同的条件下，答案不同。
### 从PlayerLoop说起

如果你想开始阅读Unity的代码，那么你一定离不开PlayerLoop这个函数。这里Player指的播放器，可以是Editor Player，Android Player，或者Windows Player等等，它最后会被编译为跨平台的代码。在Player的一次Loop里，做的就是一次绘制，或者用另一种说法，一次loop就是一帧。在一次PlayerLoop的循环里，包括脚本、物理、音频、图形、垂直同步(vSync)等等几乎所有模块的更新。

这个函数大概长这样：
![](https://johnyoung404.github.io/img/monoBehaviour/PlayerLoop.png)

对于这篇文章的主题，我们要关注其中几个函数的执行顺序。在后面的DelayedCallManager和BehaviourManager中我们再具体讨论这几个函数。

### 协程的实现和延迟调用：DelayedCallManager

实现协程最重要的机制就是延迟调用，而MonoBehaviour的Start函数，也和延迟调用有关。

在Unity中这个功能是由一个叫DelayedCallManager的类来管理的。

DelayedCallManager的能力简单来说可以概括为：
* 提供计时能力，可以选择某个时间之后调用回调
* 提供一定的顺序控制能力，在PlayerLoop的某些阶段或者下一帧才能执行某种回调
* 提供取消回调的能力

对于第二点，DelayedCallManager提供如下几种模式：
``` C++
enum DelayedCallMode
{
    kRunFixedFrameRate     = 1 << 0,
    kRunDynamicFrameRate   = 1 << 1,
    kRunStartupFrame       = 1 << 2,
    kWaitForNextFrame      = 1 << 3,
    kAfterLoadingCompleted = 1 << 4,
    kEndOfFrame            = 1 << 5,
    kRunOnClearAll         = 1 << 6,     
};
```
让我们回过头看看PlayerLoop，其中和DelayedCallManager有关的回调如下：
```C++
...
CALL_UPDATE_MODULAR(EarlyUpdate, ScriptRunDelayedStartupFrame);         //kRunStartupFrame
...
while (GetTimeManager().StepFixedTime())
    {
        ...
        CALL_UPDATE_MODULAR(FixedUpdate, ScriptRunDelayedFixedFrameRate);   //kRunFixedFrameRate
        ...
    }
...
CALL_UPDATE_MODULAR(Update, ScriptRunDelayedDynamicFrameRate);          //kRunDynamicFrameRate
...
CALL_UPDATE_MODULAR(PostLateUpdate, ScriptRunDelayedDynamicFrameRate);  //kRunDynamicFrameRate
...
CALL_UPDATE_MODULAR(PostLateUpdate, PlayerSendFrameComplete);           //kEndOfFrame
```
注意这里的kRunDynamicFrameRate在Update调用之后，在PostLateUpdate之前又调用了一次，在原来的代码里，在这里有一行注释，表达了Code Reviewer的不解：
```C++
 REGISTER_PLAYERLOOP_CALL(PostLateUpdate, ScriptRunDelayedDynamicFrameRate,
    {
        // second time, why?
        GetDelayedCallManager().Update(DelayedCallManager::kRunDynamicFrameRate);
    });
```
根据我的理解，可能是因为Update和LateUpdate中会引入一些延迟调用，这些调用必须在第一帧被调用到（虽然不知道具体的场景是什么，但是应该是这种考虑）。

介绍延迟调用，主要是因为MonoBehaviour的Start是通过延迟调用的。

### MonoBehaviour更新顺序的管理：BehaviourManager

MonoBehaviour的Update, FixedUpdate, LateUpdate的更新分别是由三个从BaseBehaviourManager继承的子类来管理的。他们在功能上完全一致，就是类型不同，因此我们只需要看看BaseBehaviourManager::CommonUpdate干了些什么就好。

我们先在PlayerLoop中看看相关函数顺序：
```C++
...
while (GetTimeManager().StepFixedTime())
    {
        ...
        CALL_UPDATE_MODULAR(FixedUpdate, ScriptRunBehaviourFixedUpdate)
        ...
    }
...
CALL_UPDATE_MODULAR(Update, ScriptRunBehaviourUpdate);
...
CALL_UPDATE_MODULAR(PreLateUpdate, ScriptRunBehaviourLateUpdate);
```
在对应的update的时候，会调用BaseBehaviourManager::CommonUpdate的对应版本。
我们来看看这个函数：
```C++
template<typename T>
void BaseBehaviourManager::CommonUpdate()
{
    IntegrateLists();

    for (Lists::iterator i = m_Lists.begin(); i != m_Lists.end(); i++)
    {
        Lists::mapped_type& listPair = (*i).second;

        SafeIterator<BehaviourList> iterator(*listPair.first);
        while (iterator.Next())
        {
            Behaviour& behaviour = **iterator;
            T::UpdateBehaviour(behaviour);
        }
    }
}

void BaseBehaviourManager::IntegrateLists()
{
    for (Lists::iterator i = m_Lists.begin(); i != m_Lists.end(); i++)
    {
        Lists::mapped_type& listPair = (*i).second;

        listPair.first->append(*listPair.second);
    }
}

void BaseBehaviourManager::AddBehaviour(BehaviourListNode& p, int queueIndex)
{
    Lists::mapped_type& listPair = m_Lists[queueIndex];
    if (listPair.first == NULL)
    {
        listPair.first = new BehaviourList();
        listPair.second = new BehaviourList();
    }

    listPair.second->push_back(p);
}
```
实际上BehaviourManager内部维护了两个链表，在MonoBehaviour实例化的时候，会把Start加入DelayedCallManager,同时会把所有不为Null的回调通过BaseBehaviourManager::AddBehaviour加入到对应的BehaviourManager的第二个链表，代码如下：
```
if (!m_Methods[MonoScriptCache::kCoroutineStart].IsNull() || !m_Methods[MonoScriptCache::kCoroutineMain].IsNull())
        CallDelayed(DelayedStartCall, this, -10, NULL, 0.0F, NULL, DelayedCallManager::kRunDynamicFrameRate | DelayedCallManager::kRunFixedFrameRate | DelayedCallManager::kRunStartupFrame);

AddBehaviourCallbacksToManagers();
```
在BehaviourManager更新的时候，会合并第一个和第二个链表，再依次执行所有的回调。在执行这些回调的时候，如果没有Start过，就先执行Start(有个变量m_started，保证只start一次)。

为什么采用两个链表的设计呢？因为如果我们在Update里面有实例化MonoBehaviour的话，我们不能在现在这个链表里处理(否则可能会有死循环)，我们只能先加到第二个链表，等下次合并之后再执行。

### 回到之前那个脚本，你有答案了吗？

不知道有没有解释清楚，可能对照源码可能更清楚一点。对于之前提出的那个问题，你有答案了嘛？

假设一个MonoBehaviour是在Update中被实例化的，它的Fixed、UpdateUpdate和LateUpdater会被加到对应的BehaviourManager的第二个链表中，其中FixedUpdate、Update无论如何都是下一帧执行，因为合并链表的操作在Update开始时进行过一次，而FixedUpdate在这一帧早已经进行完了；但这一帧的LateUpdate还未执行合并链表的操作，因此新初始化的LateUpdate可以在这一帧执行。

所以对于挂在了那个脚本的物体，如果是挂在场景中，那么不会有任何输出；如果是在Update中初始化的，则会输出一次“Eureka!”。