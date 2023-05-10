---
title: Is "if (gameObject == null) {}" Equals to "gameObject?." in Unity?
date: 2023-05-10 21:10:02
tags:
- Unity
categories:
- Tech
---



首先请原谅我在标题放了个洋屁，我只是觉得在一长串英文字母里塞几个中文字符有点怪。

标题的意思是：“请问在Unity中```if (gameObject == null) {} ```和 ```gameObject?.``` 是等价的吗？”

<!--more-->

-------

# 问题



# 结论(?)

**不一样**。

Unity重写了```UnityEngine.Object```的```==```运算符，在一个```GameObject```被Destroy后，如果使用```gameObject == null```将会返回```true```，换句话说，是Unity让你觉得这个对象已经不存在了。
但实际上这个对象还在内存中，如果调用原生的```object.ReferenceEquals(gameObject, null)```进行引用检查，就会返回```false```。

但是```?.```运算符是没有重写的（实际上官方[不允许重写该运算符](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/operator-overloading#non-overloadable-operators)），它依然使用了C#原生的方式，所以会抛出```MissingRefernceException```的异常。在Unity[对Object类的说明文档](https://docs.unity3d.com/ScriptReference/Object.html)里仅提到了不支持空判断运算符：

> This class doesn't support the null-conditional operator (?.) and the null-coalescing operator (??).

# How?

但是，这期间发生了什么？Unity是如何创造一个对象不存在于内存的假象的？

# Why?

为什么Unity要重写```==```？





-------



一点题外话，如果你在Unity开发中使用的IDE是Rider，在你对一个继承自```UnityEngine.Object```

的对象使用```?.```时将会受到如下警告：![image-20230510220636162](D:\Project\ViEqqedf.github.io\source\_posts\image-20230510220636162.png)

看！多好用的IDE！快来用Rider！（笑
