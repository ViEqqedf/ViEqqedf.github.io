---
title: Unity Entities SubScene 烘焙主要流程
date: 2024-03-28 16:59:56
tags:
- Unity
- ECS
categories:
- Tech
---



Unity Entities 有 SubScene 的概念，它允许场景之间存在嵌套关系，并通过 Bake 的方式预先烘焙数据到储存设备上，本篇说明记录 Unity Baking 的主要流程，为多平台 / 环境的烘焙区分做准备。

<!--more-->

Unity Baking Scene 的主要流程如下：

1. Bake 流程专用的 World (_ConvertedWorld / _IncrementalConversionDebug.World / EditorScenesBakingWorld) 被创建

2. BakingSystem 被创建。该系统承担主要的 Baker Invoke 任务，该 System 执行结束后 authoring 的组件和 GameObject 将被转化为 ECS 概念中的实体和组件

   1. BakerDataUtility 被初始化
      1. 添加继承了 GameObjectBaker 和 Baker<> 的 Baker 类型到容器 _IndexToBakerInstances 中，并添加该 Baker 的程序集信息到 _BakersByAssembly 容器中
      2. 对每个类型的 Baker 根据使用该 Baker 的类型数量进行降序排序

3. 开始 PreBaking 阶段

   1. 获得所有 [WorldSystemFilter(WorldSystemFilterFlags.BakingSystem)] 的系统

   2. 依序

      创建各个 BakingSystemGroup

      1. BakingSystemGroup
      2. PostBakingSystemGroup
      3. PreBakingSystemGroup
      4. TransformBakingSystemGroup

   3. 将 WorldSystemFilterFlags.BakingSystem 中找到的系统创建到其 UpdateInGroup 特性对应的 group 内，并排序

   4. 执行 PreBakingSystemGroup.Update

4. 执行 bakingSystem.Bake

   1. 为烘焙流程做准备 
      - 对于全量 Bake
        1. 清理 Bake 缓存
        2. 为预制体创建 Entity
        3. 收集场景中**所有** GameObject (_BakeInstructionsCache.CreatedGameObjects / _BakeInstructionsCache.BakeGameObjects) 及其上需要 Bake 的组件 (_BakeInstructionsCache.BakeComponents)
      - 对于增量 Bake
        1. 更新发生变化了的预制体 tracker 信息
        2. 创建增量烘焙 batch (IncrementalBakingBatch)
        3. 收集场景中**发生变化的** GameObject 及其上**发生变化的**需要 Bake 的组件 (_BakeInstructionsCache.BakeComponents / _BakeInstructionsCache.RevertComponents)
   2. 执行所有 Baker 的 baker.InvokeBake，并写入一个 EntityCommandBuffer 中。其中部分需要还原的 Entity(?) 状态会被写入一个单独的 EntityCommandBuffer 中，作用尚且不明

5. 开始 PostBaking 阶段

   1. 执行 TransformBakingSystemGroup.Update
   2. 执行 BakingSystemGroup.Update
   3. 执行 LinkedEntityGroupBaking.Update
      1. 以 Chunk 为单位为每个转化完毕的 Entity 添加 LinkedEntityGroup
      2. 从该 Entity 的 LinkedEntityGroupBakingData 中剔除包含了 BakeOnlyEntity 特性的实体（即不会输出到二进制文件），剩余的实体加入 LinkedEntityGroup Buffer
   4. 执行 PostBakingSystemGroup.Update

6. 结束
