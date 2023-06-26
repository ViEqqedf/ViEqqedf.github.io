---
title: Unity在插入新组件时的生命周期执行顺序
date: 2022-02-27 15:05:15
tags:
- Unity
categories:
- Tech
---

众所周知，在Unity中挂载了```MonoBehaviour```脚本的物体将遵循Unity的生命周期执行Tick，常用的api和顺序如下：

```
Awake();
Start();
Update();
LateUpdate();
```

但是，如果在物体A的任意一个生命周期阶段中通过```AddComponent```方法添加了另一个组件B，组件B和组件A的执行顺序是怎么样的呢？

<!--more-->

写一个简单的测试脚本：

```c#
using UnityEngine;

public class TestComponent : MonoBehaviour {
    private bool hasAwake = false;
    private bool hasStarted = false;
    private bool hasUpdated = false;
    private bool hasLateUpdated = false;

    private void Awake() {
        if (!hasAwake) {
            Debug.Log($"{Time.frameCount}  Test Comp Awake");
        }
        hasAwake = true;
    }

    void Start() {
        if (!hasStarted) {
            Debug.Log($"{Time.frameCount}  Test Comp Start");
        }
        hasStarted = true;
    }

    void Update() {
        if (!hasUpdated) {
            Debug.Log($"{Time.frameCount}  Test Comp Update");
        }
        hasUpdated = true;
    }

    private void LateUpdate() {
        if (!hasLateUpdated) {
            Debug.Log($"{Time.frameCount}  Test Comp LateUpdate");
        }
        hasLateUpdated = true;
    }
}
```

同时创建一个逻辑完全相同的```TestComponet2```组件，并在```TestComponent```中各个生命周期阶段分别添加```gameObject.AddComponent<TestComponent2>();```。

在```Awake```中添加：

```c#
      0  Test Comp Awake
      0  Test Comp2 Awake
      1  Test Comp Start
      1  Test Comp2 Start
      1  Test Comp Update
      1  Test Comp2 Update
      1  Test Comp LateUpdate
      1  Test Comp2 LateUpdate
```

可以看到，两组件完全同步。

-------

在```Start```中添加：

```c#
      0  Test Comp Awake
      1  Test Comp Start
      1  Test Comp2 Awake
      1  Test Comp2 Start
      1  Test Comp Update
      1  Test Comp2 Update
      1  Test Comp LateUpdate
      1  Test Comp2 LateUpdate
```

可以看到，因为```Comp2```是在```Start```中添加上去的，所以 ```Comp2```的```Awake```也推迟了一帧，但后续Tick依然同步。

-------

在```Update```中添加：

```c#
      0  Test Comp Awake
      1  Test Comp Start
      1  Test Comp Update
      1  Test Comp2 Awake
      1  Test Comp2 Start
      1  Test Comp LateUpdate
      1  Test Comp2 LateUpdate
      2  Test Comp2 Update
```

发现```Comp2```生命周期中的```Update```被推迟到了下一帧！但是其```LateUpdate```依然同步。


-------

在```LateUpdate```中添加:

```c#
      0  Test Comp Awake
      1  Test Comp Start
      1  Test Comp Update
      1  Test Comp LateUpdate
      1  Test Comp2 Awake
      1  Test Comp2 Start
      2  Test Comp2 Update
      2  Test Comp2 LateUpdate
```

可以看到，```Comp2```在最后一个阶段（此处举例中四个的最后一个）才被添加，所以它的```Update```和```LateUpdate```都被推迟到了下一帧。

-------

上述四种情况中，除了```Update```的情况都相对符合直觉。为什么唯独它这么奇怪？

猜想：在引擎每一次执行```Update()```之前，整个世界需要```Update()```的组件数量就会确定，并在执行过程中不会改变这个集合。所以猜测```LateUpdate()```和```FixedUpdated()```等方法的结果应该也是相同的。

试了一下，确实是这样：

```
0  Test Comp Awake
1  Test Comp Start
1  Test Comp FixedUpdate
1  Test Comp2 Awake
1  Test Comp2 Start
1  Test Comp Update
1  Test Comp2 Update
1  Test Comp LateUpdate
1  Test Comp2 LateUpdate
2  Test Comp2 FixedUpdate
```

