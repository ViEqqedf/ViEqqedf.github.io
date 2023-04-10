---
title: Unity Job System介绍(1)——概览
date: 2023-04-04 11:01:22
tags:
- Unity
- ECS
categories:
- Tech
---

# Job System 概览

Unity的Job System使用户可以编写多线程代码，让程序可以使用所有的可用CPU核心来执行你的代码。多线程执行将提供更好的性能表现，这是因为程序将更有效地使用正在运行其上的CPU核心容量，而不是只在一个CPU核心上运行所有代码。

你可以单独使用Job System，但为了更好地提升性能，你也应该一起使用Burst Compiler，Burst是为Job System的编译过程专门设计的。Burst Compiler改进了代码生成，从而提高了性能并减少了移动设备的电量消耗。

你也可以同时使用Job System和Unity的ECS来编写高性能的面向数据(data-oriented)的代码。

<!--more-->

## 多线程

Unity使用自己的原生Job System来通过多个工作线程来执行自己的原生代码(native code)
