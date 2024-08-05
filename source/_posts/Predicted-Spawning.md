---
title: Predicted-Spawning
date: 2023-08-13 13:22:33
tags:
- Unity
- network
categories:
- Tech
---

# 概述

Predicted Spawning 的主要功能是：在服务端发送新生成的 Ghost 信息到客户端之前，允许客户端在本地自行生成相同 GhostType 的 Ghost，并在服务端消息抵达时同步校正 Ghost 的信息。

<!--more-->

# 实现方法

1. 对于需要预测生成的 Ghost，其 GhostMode 需要设置为 ```GhostMode.Predicted``` 或 ```GhostMode.OwnerPredicted```。或是将其 Entity 上的 Ghost Authoring Component 组件中的 Supported Ghost Modes 设置为 Predicted
2. 由于客户端需要提前进行预测生成，所以需要创建一个在 PredictedSimulationSystemGroup 中 Update 的 System，并在其中实例化需要 Predicted Spawning 的 Entity
   1. 该 System 需要同时在客户端和服务端 Update
   2. 由于大多数时候对一个实体只需要进行一次 Predicted Spawning，所以在该System中实例化 Entity之前可先判断```NetworkTime.IsFirstTimeFullyPredictingTick```，若为 ```false``` 则跳过多余的预测
3. 接收到来自服务端的 Ghost 信息后，客户端将由 DefaultGhostSpawnClassificationSystem 分拣出与之匹配的该客户端预测生成的 Ghost，默认的匹配方式是相同的 GhostType 和 5 次以内的 Spawning Tick 差值。但是根据需要，该分拣过程是可以被重写的。例如，当前有 10 个已经生成的投射物，当服务端返回信息时，每个 Ghost 都要正确地匹配上客户端对应预测生成的投射物，此时就需要重写 ClassificationSystem 的行为。具体的实现方法可参考 [Spawning Ghost Entities](https://docs.unity3d.com/Packages/com.unity.netcode@1.0/manual/ghost-spawning.html)
   1. 核心目的是将客户端中匹配的 ```PredictedGhostSpawn.entity``` 赋给接收到的新 Ghost 的 ```PredictedSpawnEntity``` 字段，并将原本保存在 Buffer 中的 PredictedGhostSpawn 移除：
         ```
         newGhostSpawn.PredictedSpawnEntity = predictedSpawnList[j].entity;
         predictedSpawnList[j] = predictedSpawnList[predictedSpawnList.Length - 1];
         predictedSpawnList.RemoveAt(predictedSpawnList.Length - 1);
         ```
   
   2. 手册的示例中创建的 Job 有两个参数：```DynamicBuffer<GhostSpawnBuffer> ghosts``` 和 ```DynamicBuffer<SnapshotDataBuffer> data```，由于需要遍历当前客户端收到的所有 Ghost 信息，所以 ```ghosts``` 参数是必需的。而如果需要通过查找快照历史来做一些更详细的匹配（例如上方例子的需求），就可以从 ```data``` 参数中获得相关信息

# 示例

## Unity 中的预生成特性示例

### 内容

客户端中的玩家点击一次发射按钮，同时发射两个投射物：

1. 投射物 A 使用 Predicted Spawing 特性在客户端立刻生成，并在客户端上与其它碰撞体发生交互。直到服务端发送了包含该投射物信息的快照后，相应信息将赋给对应的预测生成的投射物进行校正。
2. 投射物 B 不进行预测生成，将发射指令上发到服务端后，等待服务端返回投射物信息后才生成投射物

对于投射物 A，在来自服务端的投射物信息抵达时，将客户端预测生成的投射物信息与服务端下发的进行对比并输出

### 预期

通过调整当前的 NetworkTickRate 模拟延迟，预期在 NetworkTickRate 较低时，投射物 A 的生成表现依然平滑，投射物 B 会出现明显的延迟生成

## 预测失败示例

### 内容

客户端中的玩家点击一次发射按钮，发射一个投射物。该投射物在客户端进行预测生成，但在服务端手动拦截它的生成

### 预期

由于服务端没有及时下发实际的 Ghost 信息，客户端预测生成的 Ghost 触发过时销毁机制，对应的 Entity 也将被销毁

# 内部流程

1. 对于一个希望被预测生成的 Ghost，其 Ghost Entity 会被自动添加 PredictedGhostSpawnRequest 组件。
   1. 除了在实现方法中的两种方式以外，还可以使用 ```GhostPrefabCreation.ConverToGhostPrefab``` 方法，并将 ```GhostPrefabCreation.Config.SupportedGhostModes``` 设置为 ```GhostModeMask.Predicted``` 来为其添加 PredictedGhostSpawnRequest 
2. 在 PredictedGhostSpawnSystem 中，InitGhostJob 会先遍历所有的 PredictedGhostSpawnRequest 并生成和初始化 Ghost，该过程结束后对应的 PredictedGhostSpawnRequest 会被移除，然后 CleanupPredictedSpawn 会遍历已经预测生成的 Ghost，销毁掉其中过时的 Ghost
   1. InitGhostJob 通过 EntityTypeHandle 以 Chunk 为单位遍历所有 Entity
      1. 初始化 Ghost 时，会填入一个无效的 Ghost Id，填入的 GhostType 是 GhostCollectionPrefab 的索引，其值通过遍历 ```Dynamic<GhostCollectionPrefab>``` 并与 Chunk 中实体的 GhostType 进行比较获取，填入的 SpawnTick 是上一次客户端实例化了一个包含 PredictedGhostSpawnRequest 组件时的 ServerTick。
      2. 初始化完成的 Ghost 将被添加到  ```DynamicBuffer<PredictedGhostSpawn>``` 中
      3. InitGhostJob 不支持可开关的组件类型(enableable component types)，尚不理解其含义
   2. CleanupPredictedSpawn 的销毁依据是：Ghost 的 SpawnTick 比当前 Interpolation Tick 还要老
3. 在客户端接收到来自服务端的 Ghost 信息后（也就是 GhostReceiveSystem Update 结束后），客户端将由 GhostSpawnClassificationSystem 遍历 ```DynamicBuffer<PredictedGhostSpawn>```，从中筛选出 GhostOwner 为本地客户端且类型为 Predicted 的 Ghost，将这些 Ghost 的 SpawnType 设置为 ```GhostSpawnBuffer.Type.Predicted```。然后由 DefaultGhostSpawnClassificationSystem 或自定义的 Classification System 来做最终的分拣
   1. 新接收到的 Ghost 在尚未被分拣时会先被分配一个临时的 SpawnType，其值由 ```GhostPrefabBlobMetaData.DefaultMode``` 决定，在 ConvertToGhostPrefab 时会将 Ghost 的 SupportedGhostModes 写入 DefaultMode 中