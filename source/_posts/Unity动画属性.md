---
title: Unity动画属性
date: 2019-08-31 20:19:34
tags:
- Unity
categories:
- Tech
---

关于Unity Anim中的各项属性解释

<!--more-->

Has Exit Time        √

▽Settings

​        Exit Time        1

​        Fixed Duration        □

​        Transition Duration        1

​        Transition Offset        0.5

​        Interruption Source        None



就当我手动画了个图吧（笑

假定该过渡是从A动画过渡到B动画

- **Has Exit Time**: A→B是否有过渡时间，其下的Exit Time则表示发生过渡的时刻。如果没有其它过渡条件(Conditions)，则动画抵达这个时间后就会发起过渡，否则需要满足所有过渡条件才会发起。Conditions需要在动画抵达Exit Time之前满足，否则动画将根据本身的设置停止或循环。

- **Transition Offset**: 该属性表示A→B的过渡过程中，B的播放起点时间百分比。该属性的取值范围在0-1之间。举例，该值为0.5f时，A过渡到B以后B将从其50%的时刻开始播放。

以上两个属性结合可以解决很多过渡的冲突问题。例如，行走动画A过渡到奔跑动画B，两者的起步脚都是左脚，此时A→B直接转移就会产生左脚抽搐/闪回的问题。如果将Exit Time设置为**A动画的左脚迈出的结束时间点**，将Transition Offset设为**B动画的右脚起步时间点**，就可以做到流畅的过渡。

- **Transition Duration**: 过渡时间，表示A→B的完整过渡时间。如果同时勾选了**Fixed Duration**，该选项的值单位将为“秒”，否则将是A的总时长百分比。该值设为0会使两个动画之间的切换完全没有过渡
- **Interruption Source**: 直译为“打断来源”，实际上表示打断优先级。在动画播放过程中如果有某个状态(State)被触发，所有Interruption Source内值为该状态的动画都会被打断并立刻过渡到状态目标的动画。其中，"Any State"表示任意状态都可以被打断并转移到该状态来。
