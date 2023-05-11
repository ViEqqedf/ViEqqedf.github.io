---
title: Is "if (gameObject == null) {}" equals to "gameObject?." in Unity?
date: 2023-05-02 15:10:02
tags:
- Unity
- Scripting
categories:
- Tech
---



首先请原谅我在标题放了点洋屁，我只是觉得在一长串英文字母里塞几个中文字符不太整齐。

标题的意思是：“请问在Unity中```if (gameObject == null) {} ```和 ```gameObject?.``` 是等价的吗？”

<!--more-->

描述得详细些：现有一```UnityEngine.GameObject```类型对象```testGo```，分别对该对象做如下两个操作，产生的结果是否相同？

```c#
UnityEngine.Object.Destroy(testGo);
if(testGo != null) {
    testGo.SetActive(false);
}
```

```c#
UnityEngine.Object.Destroy(testGo);
testGo?.SetActive(false);
```

# 结论？

**不一样**。

Unity重载了```UnityEngine.Object```类的```==```运算符，在一个```GameObject```被Destroy后，若使用```gameObject == null```进行比较，将会返回```true```。换句话说，是Unity让你觉得这个对象已经不存在了。实际上这个对象还在内存中，若调用原生的```object.ReferenceEquals(gameObject, null)```方法进行引用检查，将会返回```false```。

然而，```?.```运算符是没有被重载的（实际上微软[不允许重载该运算符](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/operator-overloading#non-overloadable-operators)），它依然使用了C#原生的操作方式，所以对一个已被Destroy的```UnityEngine.Object```对象使用```?.```时，可能会抛出```MissingRefernceException```异常。

需要指出的是，所有继承了```UnityEngine.Object```的类都会有该现象，在Unity[对```UnityEngine.Object```类的说明文档](https://docs.unity3d.com/ScriptReference/Object.html)里提到了不支持空判断运算符：

> This class doesn't support the null-conditional operator (?.) and the null-coalescing operator (??).

由此可见，如果在开发过程中需要进行```UnityEngine.Object```对象的生命周期检查，就应该使用```if (object == null) {} ```的形式，否则应该使用```object.ReferenceEquals(gameObject, null)```或```?.```这样的语法糖。

本文结束。

# Why & How

不，还没完。

这样的运算符重载显然是一个违反直觉的行为，为什么Unity要这么做？

Unity官方给出了两个目的：

1. 当Unity初始化一个继承了```MonoBehaviour```的子类对象时，该子类对象中的字段将被赋予一个手动构造的空对象。通过该操作，Unity可以在这个手动构造的空对象中储存一些信息，以便于在该字段没有赋值的情况下被引用时获得更好的错误提示，包括发生错误的```GameObject```实例信息等，而不只是一个光秃秃的```NullReferenceException```和一些不明确的堆栈信息。Unity甚至可以借此在你双击错误信息的时候直接在Inspector中高亮对应的```GameObject```实例。
   1. 所以在这种情况下你会看到类似于```MissingRefernceException: looks like you are accessing a non initialised field in this MonoBehaviour over here```的错误提示，这是由Unity抛出的异常。
   2. 这一操作仅在Editor环境中进行，并且将产生大量的性能消耗，这也是在做Profiler时常常需要在built版本中进行的原因。
2. Unity是一个基于C/C++开发的引擎，C#是它的上层封装。然而```UnityEngine.Object```对象的实际信息都是保存在C++层的，C#层中的对象只是**原生对象**(Native Object)的**封装对象**(Wrapper Object)，其中仅包含了指向原生对象的指针。众所周知，C++有一整套显式管理内存的操作方式，当你调用```Object.Destroy(myObject)```或切换场景时，原生对象所使用的内存将被释放；而C#的对象生命周期独立地受GC系统控制，这使得一个C#层的封装对象指向一个已被释放C++层的原生对象变得可行。

基于以上两点，Unity在最初设计时重载了```UnityEngine.Object```的 ```==```。

实际上，Unity的工程师们也说明了这样做可能带来的问题：

> - 这是违反直觉的。
> - 比较两个UnityEngine.Objects对象或与null比较的速度比预期的要慢。
> - 重载==操作符不是线程安全的，因此不能在主线程之外比较对象。(这个我们可以修复)。
> - 它的行为与??操作符不一致，??也做空检查，但??做的是纯c#空检查，不能绕过它来调用我们重载的空检查。

同时直言，如果重新设计，Unity会选择取消```==```的重载，并提供类似于myObject.destroyed的属性来检查对象的生命周期。但是出于对“激进修复”和“保守维护”之间平衡[^1]的考量，以及这个现象**过强的**被忽视性[^2]，工程师们于14年向用户开放征求意见，其结果和决定在2023年的现在显而易见——这个特性被保留了下来。

# 结论。

对于我来说，之所以以接近翻译或整理的形式写下这篇Blog，一方面是软件工程中的细小设计产生的影响引发了我的兴趣，另一方面，我认为我能从中得出一个小小的结论：当你试图修改/创造一个众所周知的特性时，最好仔细考虑它在认知统一上的影响力。

结束。

# 引用

- [Custom == operator, should we keep it?](https://blog.unity.com/technology/custom-operator-should-we-keep-it)

- [Possible unintended bypass of lifetime check of underlying Unity engine object](https://github.com/JetBrains/resharper-unity/wiki/Possible-unintended-bypass-of-lifetime-check-of-underlying-Unity-engine-object)

-------

一点题外话，如果你在开发中使用Rider，那么在你对一个继承自```UnityEngine.Object```的对象使用```?.```时将会收到如下警告：

{% asset_img image-20230510220636162.png 警告内容 %}

看！多好用的IDE！快来用Rider！（笑



[^1]: 原文为&quot;fix and cleanup old things&quot; and &quot;do not break old projects&quot;
[^2]: 工程师在意见征求的Blog中提到了一件趣事：Unity的两名工程师在通过缓存优化GetComponent的性能时，因为忽视了==重载带来的性能消耗，以至于始终见不到缓存带来的优化效果。工程师们开始想：如果连我们都忽视了它，那么有多少用户会注意到呢？于是有了这次意见征求。