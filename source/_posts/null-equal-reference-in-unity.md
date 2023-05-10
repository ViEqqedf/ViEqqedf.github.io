---
title: Is "if (gameObject == null) {}" equals to "gameObject?." in Unity?
date: 2023-05-10 21:10:02
tags:
- Unity
- Scripting
categories:
- Tech
---



首先请原谅我在标题放了个洋屁，我只是觉得在一长串英文字母里塞几个中文字符不太整齐。

标题的意思是：“请问在Unity中```if (gameObject == null) {} ```和 ```gameObject?.``` 是等价的吗？”

<!--more-->

-------

# 问题



# 结论(?)

**不一样**。

Unity重写了```UnityEngine.Object```类的```==```运算符，在一个```GameObject```被Destroy后，若使用```gameObject == null```进行比较，将会返回```true```。换句话说，是Unity让你觉得该引用已经不指向任何内存地址。但实际上这个C#对象还在内存中，若调用C#原生的```object.ReferenceEquals(gameObject, null)```方法进行引用检查，将会返回```false```。

然而，```?.```运算符是没有被重写的（实际上微软[不允许重写该运算符](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/operators/operator-overloading#non-overloadable-operators)），它依然使用了C#原生的操作方式，所以对一个已被Destroy的UnityEngine.Object对象使用```?.```时，可能会抛出```MissingRefernceException```异常。

需要指出的是，所有继承了```UnityEngine.Object```的类都会有该现象，在Unity[对```UnityEngine.Object```类的说明文档](https://docs.unity3d.com/ScriptReference/Object.html)里提到了这点：

> This class doesn't support the null-conditional operator (?.) and the null-coalescing operator (??).

# How?

但是Unity是如何创造一个对象不存在于内存的“假象”的？

# Why?

为什么Unity要重写```==```？





-------



一点题外话，如果你在开发中使用Rider，那么在你对一个继承自```UnityEngine.Object```的对象使用```?.```时将会收到如下警告：

{% asset_img image-20230510220636162.png 警告内容 %}

看！多好用的IDE！快来用Rider！（笑
