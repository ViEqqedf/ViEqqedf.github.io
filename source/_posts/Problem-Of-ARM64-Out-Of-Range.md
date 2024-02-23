---
title: Unity iOS 打包过程中出现的 "ARM64 branch out of range" 问题记录
date: 2023-12-13 11:11:51
tags:
- Unity
- iOS
- Build
categories:
- Tech
---

# 问题表现

在打 iOS 平台的 develop 包时，发现 xcode 工程构建能够成功，但是 ipa 构建稳定失败。

问题发生在链接段的跳转指令生成过程，会大量重复输出 "final section layout" 信息，并且最终会报错 "ARM64 branch out of range ..."。

另外会发现，如果构建相同代码的 release 包，ipa 的构建过程会成功。

<!--more-->

其构建日志如下：

{% asset_img problem.png 构建日志 %}



# 解决方法

在 xcode 工程构建完成后，在工程根目录下找到如下 il2cpp 配置代码：

Xcode工程根目录\iOS\项目名\工程名\Il2CppOutputProject\IL2CPP\libil2cpp\os\c-api\il2cpp-config-platforms.h

将该行中 IL2CPP_LARGE_EXECUTABLE_ARM_WORKAROUND 的值从 0 修改为 1。

{% asset_img solution.png 解决方法 %}

根据官方描述，该方法会产生一个副作用：在部分情况下，托管堆栈的追踪展示可能会不正确。



# 原因

iOS 的编译过程中，链接器 (linker) 会在可执行文件的程序段 (sections) 之间插入跳转点 (branch islands)。

但是跳转点只能在同一个程序段内工作，而不能跨程序段，此时如果一个程序段的体积过大，超过了 CPU 架构的限制（例如 ARM64 的限制是 128MB），生成的跳转点就会失效，于是链接器找不到后续的段，开始报错。



# 其他可能的解决方法

1. 减小代码体积
2. 删除不需要的库
3. 平摊代码文件大小
4. 使用 unity 的代码剥离功能 (Strip Level)



# 参考

1. https://forum.unity.com/threads/xcode-can-not-build-unitywebrequest-o.1096375/
2. https://zhuanlan.zhihu.com/p/132717069
3. https://stackoverflow.com/questions/74684093/how-to-measure-arm64-branch-size
4. https://blog.csdn.net/fcsfcsfcs/article/details/116407152
5. https://juejin.cn/post/7087857814059614222
6. https://jingxuanwang.github.io/2014/12/15/reason-of-ld-unable-to-insert-branch-island
