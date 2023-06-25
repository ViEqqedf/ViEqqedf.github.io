---
title: 场景管理器说明书
date: 2021-10-17 17:24:22
tags:
- C#
- Frame_Design
categories:
- Tech
---



这篇Blog是以往工作中编写的一个场景管理器的说明文档，它的主要功能是管理游戏内场景跳转、视图状态和UI的生命周期。虽然管理器本身已经停止使用和维护，但其中的一些思考成果依然对我有效，故在此记录并归档。

话说回来，现在看来这套东西的启动效率其实有点低啊（笑

<!--more-->

# 快速开始

如何快速使用场景管理器把业务跑起来？

## 视图状态(GameViewStatus)

### 第一步

在**GameViewStatusEnum**中添加一个状态枚举，名字自定义，编号开头按游戏模块递增，编号末尾以Page的00打头，后续递增。

```c#
public enum GameViewStatusEnum
{
    //...
    
	#region 英雄.4

    HeroPage = 4000,  //Page状态必须存在，且一定是x000
    HeroMainView = 4001,  //递增
    HeroDetailView = 4002,
    HeroIncreaseStar = 4003,
    HeroMilitaryView = 4004,
    HeroQualification = 4005,
    HeroReset = 4006,

    #endregion
        
    //...
}
```

> 什么是游戏模块？这个由你决定。通常而言，需要新开一个Page的系统一定是一个游戏模块。但也有例外，例如仅在主场景展示的公告也被定义为一个游戏模块。没有很严格的约束。

### 第二步

到**GameViewDefine**的**GameViewStatusInfoMap**中使用刚才写的枚举做状态配置，你会看到其中大量使用了一个叫*ViewStatusInfo*的类。

```c#
public static Map<GameViewStatusEnum, ViewStatusInfo> GameViewStatusInfoMap = 
    new HashMap<GameViewStatusEnum, ViewStatusInfo>()
{
    //...
	[GameViewStatusEnum.HeroPage] = new ViewStatusInfo(
		new object[] { PageKeys.HeroPage },
		new GameViewStatusEnum[]
		{
			GameViewStatusEnum.App, GameViewStatusEnum.MainScene,
			GameViewStatusEnum.HeroPage,
		}, GameViewStatusEnum.App, true),
	[GameViewStatusEnum.HeroMainView] = new ViewStatusInfo(
		new object[] { UIKeys.HeroMainPanel },
		new GameViewStatusEnum[]
		{
			GameViewStatusEnum.App, GameViewStatusEnum.MainScene,
			GameViewStatusEnum.HeroPage, GameViewStatusEnum.HeroMainView,
		}),
    //...
}
```

ViewStatusInfo的构造函数接收以下四个参数，此处介绍前两个

```C#
public ViewStatusInfo(
    object[] contentKeys, 
    GameViewStatusEnum[] path, 
    GameViewStatusEnum boundaryStatus = GameViewStatusEnum.App, 
    bool ignoreOnBack = false)
{
	ViewKeys = contentKeys;
	Path = path;
	IgnoreOnBack = ignoreOnBack;
	BoundaryStatus = boundaryStatus;
}
```

1. **ViewKeys**
	本视图状态要打开的视图(SceneKey、PageKey、UIKey、MapKey)
   同一个视图状态内不允许同时包含不同层的视图Key.例如```{SceneKey, PageKey}```或```{PageKey, UIKey}```等。而同层的```{UIKey, MapKey}```是可以的。
   
2. **Path**

   抵达<u>该视图状态</u>需要经过的完整路径，即必须从起点App状态开始，以<u>该视图状态</u>结束。

   在实际跳转状态时，场景管理器会根据路径配置一个一个状态逐步进入。参考上方的视图状态```GameViewStatusEnum.HeroMainView```（接下来省略前缀），假定玩家当前正在```MainScene```，此时跳转到```HeroMainView```的话，就会先进入```HeroPage```，再进入```HeroMainView```。

剩余两个参数目前保留默认值即可，下方再介绍。**但是它们很重要，请记得去下面看。**

### 第三步

如果你的视图状态需要在**进入之前**通过请求获取一些数据，那么需要配置**入场请求**。不需要则跳过这步。

1. 在业务服务中注入```GameViewAppService```
2. 创建一个方法，在其中调用接口```GameViewAppService.SendEnterRequest```创建一个入场请求
3. 去```GameViewModule```中查找```GetEnterRequest```方法，在其中添加状态及其对应的入场请求

结束。

### 第四步（容易遗漏！）

如果你创建了一个新的Scene或Page，请去对应的Builder中添加对应的Controller，没有创建则跳过这步。

```c#
controllerModule.AddController<GameViewSceneContentController>();  //Scene需要
controllerModule.AddController<GameViewPageContentController>();  //Page需要
```

添加的位置要求在自动生成的Controller之前，例如

```c#
public override void OnBuildControllers(IControllerModule controllerModule)
{
	//要在LegionPageController前面
	controllerModule.AddController<GameViewPageContentController>();
            
	controllerModule.AddController<LegionPageController>();
	controllerModule.AddController<ChatController>();
}
```

如果有特殊情况需要有别的顺序，联系我。

------------------------------

以上，一个新的视图状态创建完成。接下来就可以用```GameViewAppService.GoToNextStatus```跳转状态（该接口有多个重载形式）、```GameViewAppService.BackPreviousStatus```回退状态。

```c#
public interface GameViewAppService
{        
	#region 状态跳转

    /// <summary>
    /// 进入视图状态（不需要上下文）
    /// </summary>
    /// <param name="targetStatus">目标状态</param>
    /// <returns></returns>
    bool GoToNextViewStatus(GameViewStatusEnum targetStatus);

    /// <summary>
    /// 进入视图状态（需要入场检查和一个可null的上下文）
    /// </summary>
    /// <param name="targetStatus"></param>
    /// <param name="enterCheck"></param>
    /// <param name="targetContext"></param>
    /// <returns></returns>
    bool GoToNextViewStatus(
        GameViewStatusEnum targetStatus, 
        Func<bool> enterCheck,
        Action checkCallback = null, 
        object targetContext = null);
        
    /// <summary>
    /// 进入视图状态
    /// </summary>
    /// <param name="targetStatus">目标状态</param>
    /// <param name="targetContext">目标状态需要的上下文</param>
    bool GoToNextViewStatus(GameViewStatusEnum targetStatus, object targetContext);
    
    /// <summary>
    /// 进入视图状态
    /// </summary>
    /// <param name="targetStatus">目标状态</param>
    /// <param name="statusContextMap">上下文字典</param>
    bool GoToNextViewStatus(
        GameViewStatusEnum targetStatus, 
        Map<GameViewStatusEnum, object> statusContextMap);
    
    /// <summary>
    /// 回到上一视图状态
    /// </summary>
    /// <param name="needReRequest"></param>
    void BackPreviousViewStatus();
        
    #endregion
}
```



## 视图状态附属界面(Additional Panel)

如果你只是想要显示一个弹窗，或者是一个对话框，或者是一个与游戏主体逻辑流程关系不大的界面，那么就不需要将其视为一个视图状态，可以当做附属界面(AdditionalPanel)简单地打开和关闭。**注意，所有附属界面会在跳转到下一个状态之前自动关闭。**

可调用的接口是

```c#
public interface GameViewAppService
{
	#region 附属界面相关操作
        
	/// <summary>
	/// 打开一个附属Panel，可传入参数
	/// </summary>
	/// <param name="uiKey"></param>
	/// <param name="context"></param>
	/// <param name="dontDestroyOnLeave">该参数无效，请无视</param>
	void OpenAdditionalPanelWithContext(
	    UIKey uiKey, 
	    object context, 
	    bool dontDestroyOnLeave = true);
    
	/// <summary>
	/// 打开一个附属Panel（不需要上下文）
	/// </summary>
	/// <param name="uiKey"></param>
	/// <param name="dontDestroyOnLoad">该参数无效，请无视</param>       
	void OpenAdditionalPanel(UIKey uiKey, bool dontDestroyOnLoad = true);
    
	/// <summary>
	/// 手动关闭附属Panel
	/// </summary>
	/// <param name="uiKey"></param>
	void CloseAdditionalPanel(UIKey uiKey);
        
	#endregion
}
```



-------------------------------------------



# 补充介绍

> Tip:在快速开始中介绍过的概念大多不会在此重复说明。

## 操作栈和视图链

场景管理器是以**视图状态(GameViewStatus)**为基本单位做视图跳转的一个模块。其中维护实时游戏场景实时状态的数据结构是**操作栈**和**视图链**。

### 操作栈

这是一个栈结构，它用来记录场景管理器的实时操作记录。当调用```GameViewAppService.GoToNextStatus```跳转状态时，该<u>操作记录</u>会被压入操作栈中；当调用```GameViewAppService.BackPreviousStatus```后，栈顶的<u>操作记录</u>会被弹出操作栈。操作记录是以```GameViewStatusEnum```标记的，可以在Log中看到实时的变化。

### 视图链

这是一个List，它用来记录实时的游戏场景状态，即当前游戏内的各个视图状态是以什么样的顺序或层级展示的。视图链只和实际展示出来的视图状态有关。越靠后的视图状态，表现上越在上层。

链尾的视图状态就是当前视图状态，也是```GameViewAppService.GetCurStatus```接口获得的结果。

视图链一定会和当前视图状态在配置中的路径一致。

## ViewStatusInfo

ViewStatusInfo是场景管理器的核心数据模型，它描述了视图状态的所有基本要素

```c#
public struct ViewStatusInfo
{
	// 本状态要打开的视图（填入目标视图的Key）
	public object[] ViewKeys;

	// 完整到达路径（参考样例，必须从App状态开始完整描述）
	public GameViewStatusEnum[] Path;

	// 返回（即调用BackPreviousStatus方法）时是否跳过该状态
	public bool IgnoreOnBack;

	// 状态作用域（作用域状态必须已存在于当前实时链上才允许进入该状态）
	public GameViewStatusEnum BoundaryStatus;
	
	//...
}
```
1. **ViewKeys**
   本视图状态要打开的视图(SceneKey、PageKey、UIKey、MapKey)
   同一个视图状态内不允许同时包含不同层的视图Key.例如```{SceneKey, PageKey}```或```{PageKey, UIKey}```等。而同层的```{UIKey, MapKey}```是可以的。

   视图打开顺序和写在数组内的顺序一致，但是如果某些视图（如Map）有一些异步加载策略的话则无法保证实际顺序。
   
2. **Path**

   抵达<u>该视图状态</u>需要经过的完整路径，即必须从起点App状态开始，以<u>该视图状态</u>结束。

   在实际跳转状态时，场景管理器会根据路径配置一个一个状态逐步进入。参考上方的视图状态```GameViewStatusEnum.HeroMainView```（接下来省略前缀），假定玩家当前正在```MainScene```，此时跳转到```HeroMainView```的话，就会先进入```HeroPage```，再进入```HeroMainView```。

3. **IgnoreOnBack**

   一般情况下，调用GameViewAppService.BackPreviousStatus()时，操作栈会做一次出栈，然后视图上回到新的操作栈栈顶的位置。但是如果该参数的值为true，则会**再出栈一次**。简单来说就是回退两次操作。

4. **BoundaryStatus**

   该属性表示，在试图调用跳转接口跳转到该状态时，玩家当前必须该属性的视图状态内，默认值为顶级的App状态，即随意跳转（App是所有状态路径的起点）。跳转状态时，场景管理器会在当前的视图链中搜索与该属性匹配的状态，搜索成功则允许跳转，搜索失败则丢弃这次操作。
   
## 入场请求(EnterRequest)

在配置中的所有入场网络请求会在视图开启之前全部完成，即客户端收到了所有入场请求的Response。

例如，用户希望跳转的视图状态A的路径上需要经过5个其它视图状态，而这5个视图状态都配置了入场请求，那么只有在这5个请求都完成以后，路径上的第一个视图才会开始打开。这是为了保证视图所需数据的完整性。

## 视图状态附属界面(Additional Panel)

附属界面是隶属于打开时所处的视图状态的，它的**生命周期与隶属的视图状态一致**。例如，如果我当前在HeroList英雄列表状态，我在这里打开了一个附属弹窗，然后我跳转到MainScene主场景。则HeroList状态和它的附属弹窗会一起被关闭。

## 视图状态与游戏模块编号

目前，项目内的游戏模块编号和视图状态```GameViewStatusEnum```的映射数值是一致的。

所以如果策划来问某个模块的入口编号是什么，就要到这里来看对应的视图状态枚举。

```c#
/// <summary>
/// 视图状态枚举
/// 编号格式为("游戏模块编号" * 1000)起始的1000个编号
/// </summary>
public enum GameViewStatusEnum
{
	...

    #region 英雄.4

    HeroPage = 4000,
    HeroMainView = 4001,
    HeroDetailView = 4002,
    HeroIncreaseStar = 4003,
    HeroMilitaryView = 4004,
    HeroQualification = 4005,
    HeroReset = 4006,

    #endregion
    
    ...
}
```

## 当你调用GoToNextStatus的时候会发生什么？

对于用户来说只要知道一件事，就是**视图链**和**实际的视图展示(Panel,Map等)**一定会和你在视图状态中配置的**路径**完全一致。在跳转之前的，与目标状态路径中有差异的状态节点会被关闭销毁，还没打开的视图会按照路径配置顺序依次打开。

```c#
public static Map<GameViewStatusEnum, ViewStatusInfo> GameViewStatusInfoMap =
	new HashMap<GameViewStatusEnum, ViewStatusInfo>()
{
    //英雄升星
	[GameViewStatusEnum.HeroIncreaseStar] = new ViewStatusInfo(
	    new object[] { UIKeys.HeroIncreaseStarPanel },
	    new GameViewStatusEnum[]
	    {
	        GameViewStatusEnum.App, GameViewStatusEnum.MainScene,
	        GameViewStatusEnum.HeroPage, GameViewStatusEnum.HeroIncreaseStar
	    }, GameViewStatusEnum.HeroPage),
	//英雄军衔
    [GameViewStatusEnum.HeroMilitaryView] = new ViewStatusInfo(
	    new object[] { UIKeys.HeroIncreaseMilitaryPanel },
	    new GameViewStatusEnum[]
	    {
	        GameViewStatusEnum.App, GameViewStatusEnum.MainScene,
	        GameViewStatusEnum.HeroPage, GameViewStatusEnum.HeroMilitaryView,
	    }, GameViewStatusEnum.HeroPage)
}
```

例如以上配置，英雄升星和英雄军衔路径的差异只在最后一个节点。那么当用户从升星状态跳转到军衔状态时，就会把升星的视图```HeroIncreaseStarPanel```关闭，把军衔的视图```HeroIncreaseMilitaryPanel```打开。

----------------------



# 其它

1. 只有```GameViewAppService```中的接口是开放给用户的，**不允许**访问和调用其它和GameView相关的领域数据和接口。
2. 不要在退出模块之前发一个**非静默**的网络请求！如果玩家手速快的话可能会阻塞下一个模块的入场请求。
3. 框架内Page的切换需要一定时间，如果多次快速切换可能会被框架阻塞。
