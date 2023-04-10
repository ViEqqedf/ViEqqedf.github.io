---
title: HttpClient的一种错误使用方式
date: 2023-04-10 18:34:18
tags:
- C#
categories:
- Tech
---

# 问题

在.Net相关的开发中经常需要使用到各种非托管或不受GC控制的资源，这类资源管理类大多实现了IDisposable接口。在使用完毕后，我们需要通过该接口及时清理它们。

[微软的手册](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/statements/using)中建议我们使用```using```关键字，该关键字结束后会隐式地调用Disposable进行清理，而不用我们手动调用```Dispose```方法。

HttpClient是.Net提供的一个用于分配TCP连接资源，处理Http请求和响应的类。它在初始化时将申请一个套接字资源并创建连接，所以很自然地，我们会这样子使用它：

<!--more-->

```c#
using(var client = new HttpClient()) {
    // 一些操作
}
```

接下来我们来模拟该操作执行多次的情景：

```c#
// 使用using创建一个HttpClient实例来Get一些数据
Console.WriteLine("Starting connections");
for(int i = 0; i<50; i++) {
    // 重复执行50次
    using(var client = new HttpClient()) {
    	var result = await client.GetAsync("一个url");
		Console.WriteLine(result.StatusCode);
	}
}
Console.WriteLine("Connections done");
```

我们创建了50个HttpClient实例，并且在每次using结束后都清理了其占用的所有资源。非常合理。

但如果我们在Shell中通过```netstat```指令查看本机的连接状态会发现有大量的重复连接信息，并且它们都处于```TIME_WAIT```状态（即该连接已经由某一方断开了连接）。

{% asset_img image-20230410211356163.png 测试结果 %}

# 原因

实际上，HttpClient类的设计目的是仅实例化一次的单例对象，并在程序的整个生命周期内重复使用。HttpClient在初始化时将创建一个该实例专属的连接池，在调用Dispose释放HttpClient实例时，它占用的TCP端口并不会在连接关闭后立刻释放，而是会遵循TCP协议保持TIME-WAIT状态一段时间，该时间取决于系统设置。

# 解决方法

在第一次创建出HttpClient实例后，在非必要的情况下不要Dispose，重用这个实例

```c#
private static HttpClient Client = new HttpClient();

public static async Task Main(string[] args) {
    Console.WriteLine("Starting connections");
    for (int i = 0; i < 10; i++) {
        var result = await Client.GetAsync("一个url");
        Console.WriteLine(result.StatusCode);
    }
    Console.WriteLine("Connections done");
    Console.ReadLine();
}
```

运行这个例子，可以看到保持着的连接只剩下一个了，并且处于```ESTABLISHED```激活状态

{% asset_img image-20230410213200891.png 测试结果 %}

# 引用

1. [HttpClient 类](https://learn.microsoft.com/zh-cn/dotnet/api/System.Net.Http.HttpClient?view=net-7.0)
2. [HttpClient 的使用准则](https://learn.microsoft.com/zh-cn/dotnet/api/System.Net.Http.HttpClient?view=net-7.0)
3. [You are using HttpClient wrong](https://www.aspnetmonsters.com/2016/08/2016-08-27-httpclientwrong/)