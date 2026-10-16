# EM自定义异步加载

> [$openspec-explore](F://Pan1//UE_git//.agents//skills//openspec-explore//SKILL.md)
> 分析当前引擎异步加载功能模块

<details><summary>7 previous messages</summary>

> 我会先按你点名的 `openspec-explore` 来进入探索模式，同时读取会影响本轮工作的技能说明；然后再去引擎里定位异步加载相关模块，尽量把结构、入口、线程模型和关键风险讲清楚。
>
> 我已读取 `openspec-explore`：这轮只做分析和梳理，不改代码。先快速看一下当前 OpenSpec 上下文，再把 UE4 里的 Async Loading 代码入口和调用链拉出来。
>
> `openspec list --json` 被 PowerShell 执行策略拦住了，我会绕到 `openspec.cmd` 读取上下文，不需要改系统策略。代码定位已经确认：核心集中在 `CoreUObject/Serialization`，外层调用散在 `Engine` 的关卡流送、StreamableManager、World 等位置。
>
> 当前没有 active OpenSpec change（`{"changes":[]}`），所以这次就是纯架构探索。接下来我会读关键实现片段：一个是统一入口 `AsyncPackageLoader`，一个是旧加载器 `AsyncLoading`，一个是新加载器 `AsyncLoading2`。
>
> 旧加载器这边已经能看出两套路径叠在一起：普通 `TickAsyncPackage` 分阶段推进，以及 EDL 事件队列推进；并且你们代码里有中文注释提到“同步资源加载结束后先暂停其他异步加载，优先处理同步 PostLoad”，这是当前分支值得单独标出来的行为。
>
> 新加载器 `AsyncLoading2` 是 IoStore/PackageStore 路线：启用时会强制 EDL，包名先转成 `FPackageId`，再通过 `FPackageStore` 找 store entry；事件粒度也从旧版的 import/export 节点，变成 package summary 和 export bundle 节点。
>
> 我再快速看了一眼配置和调用方：默认配置里包流送线程是关的、EDL 是开的；所以“异步加载”在默认情况下更多是主线程分帧推进，只有 cooked/命令行/项目配置满足条件时才真的跑独立 ALT 线程。
</details>

已按 `openspec-explore` 做了只读分析，没有改代码。当前没有 active OpenSpec change。

**总体结论**

当前引擎的异步加载核心在 `CoreUObject/Serialization`，外部由关卡流送、软引用加载、无缝旅行、网络 GUID 等系统发起 `LoadPackageAsync`。它不是单一实现，而是统一接口 `IAsyncPackageLoader` 后面挂两套 loader：

```text
ULevelStreaming / StreamableManager / World / NetGUID
        │
        ▼
LoadPackageAsync(...)
        │
        ▼
GPackageLoader : IAsyncPackageLoader
        │
        ├─ AsyncLoading2：IoStore / PackageStore / FPackageId 路线
        │
        └─ FAsyncLoadingThread：传统 LinkerLoad + EDL/非 EDL 路线
        │
        ▼
ProcessAsyncLoading 每帧分片推进，或 FlushAsyncLoading 阻塞完成
        ▼
I/O -> 创建包/对象 -> 序列化 -> PostLoad -> GC clusters -> completion delegate
```

**关键文件**

- [AsyncPackageLoader.h](./Engine/Source/Runtime/CoreUObject/Public/Serialization/AsyncPackageLoader.h:68)：统一接口、统计、线程设置。
- [AsyncPackageLoader.cpp](./Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncPackageLoader.cpp:567)：创建 `GPackageLoader`，并暴露 `LoadPackageAsync`、`FlushAsyncLoading`、`ProcessAsyncLoading`。
- [AsyncLoading.h](./Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncLoading.h:25) / [AsyncLoading.cpp](./Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncLoading.cpp:4107)：传统 loader，含 EDL 事件图和旧非 EDL 分阶段加载。
- [AsyncLoading2.h](./Engine/Source/Runtime/CoreUObject/Public/Serialization/AsyncLoading2.h:346) / [AsyncLoading2.cpp](./Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncLoading2.cpp:2205)：IoStore/PackageStore 新 loader。
- [UObjectGlobals.cpp](./Engine/Source/Runtime/CoreUObject/Private/UObject/UObjectGlobals.cpp:483)：每帧 `StaticTick` 调 `ProcessAsyncLoading`。
- [LevelStreaming.cpp](./Engine/Source/Runtime/Engine/Private/LevelStreaming.cpp:1062) 和 [StreamableManager.cpp](./Engine/Source/Runtime/Engine/Private/StreamableManager.cpp:1120)：主要业务入口。

**当前默认行为**

[BaseEngine.ini](./Engine/Config/BaseEngine.ini:994) 里默认：

```ini
s.AsyncLoadingThreadEnabled=False
s.EventDrivenLoaderEnabled=True
s.UseBackgroundLevelStreaming=True
```

所以默认配置下，“异步加载”通常是主线程分帧推进，不一定有独立 Async Loading Thread。只有 cooked 平台、`THREADSAFE_UOBJECTS`、配置/命令行、`FApp::ShouldUseThreadingForPerformance()` 等条件满足时，才会真正启用 ALT。`AsyncLoading2` 在 `WITH_ASYNCLOADING2 && FIoDispatcher::IsInitialized()` 时走 IoStore 路线。

**值得注意的分支特征**

你们当前分支在旧 EDL loader 里有同步加载优化相关逻辑：`AsyncPackagesReadyForTick`、`ForceFlushAsyncPackagesForTickCount`、`s.SyncLoaderOptimizationLogEnabled`，并在事件队列推进处标注了“优先处理同步 PostLoad”。对应位置在 [AsyncLoadingThread.h](./Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncLoadingThread.h:227) 和 [AsyncLoading.cpp](./Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncLoading.cpp:4168)。这块是分析卡顿、同步加载抢占、异步包饥饿时最该看的地方。

**主要风险点**

- `FlushAsyncLoading()` 会阻塞主线程，GC、ResetLoaders、部分关卡流送策略都会触发它。
- `PostLoad` 里再加载对象会造成额外阻塞，代码里已有 warning。
- `AsyncLoadingThreadEnabled=False` 时，加载仍“异步请求”，但执行预算来自主线程 tick。
- `AsyncLoading2` 强依赖 IoStore/PackageStore；找不到 `StoreEntry` 会跳过并延迟失败回调。
- OpenHarmony 侧我没看到单独覆盖这些流送配置，应该继承通用路径，重点仍是 cooked 平台属性和线程能力判断。

> 1. 项目中这套异步加载，相对于UE原生引擎异步加载做了哪些定制化
> 2. FlushAsyncLoading为什么会阻塞主线程
> 3. 在UE5的哪个版本对这个阻塞问题是否有修复

<details><summary>13 previous messages</summary>

> 我会继续用 `openspec-explore` 做只读分析：这三个问题需要把本地定制点和 UE 原生路径分开讲，第三点还要查 UE5 公开资料/源码脉络，避免凭记忆拍版本号。
>
> 本地仓库有未提交改动，但都在 Slate/UMG 一带，和异步加载无关。我先不碰这些；下面会从 `AsyncLoading*` 的 git 历史和代码特征里找“项目定制”的证据，再用公开资料核 UE5 版本点。
>
> <details><summary>Searched the web</summary>
>
> Searched the web for `Unreal Engine 5 async loading flush blocking main thread AsyncLoading2 version fix`
>
> Searched the web for `site:github.com/EpicGames/UnrealEngine AsyncLoading2 FlushLoading TickAsyncLoadingFromGameThread UE5.0 UE5.1`
>
> Searched the web for `"FAsyncLoadingThread2::FlushLoading" "RequestId"`
>
> Searched the web for `Unreal Engine 5.3 async loading release notes package loading`
>
> Searched the web for `Unreal Engine 5.4 release notes async loading IoStore`
>
> Searched the web for `site:dev.epicgames.com/documentation/en-us/unreal-engine unreal engine 5 async loading package streaming`
>
> Searched the web for `"AsyncPackageLoader2" "UE5"`
>
> Searched the web for `"AsyncLoading2 - Created"`
>
> </details>
>
> 本仓库明确是 `++UE4+Release-4.27`，也就是说“原生对比”的基线更像 Epic UE4.27.2，而不是 UE5。git 历史里异步加载相关定制提交很集中，主题基本都是“同步加载优先级 / ALT&EDL / 卡顿优化”。
>
> 有一个很关键的结论：定制并不是“大改整套 async loader”，而是主要卡在旧 EDL/ALT 路径里的同步加载抢占策略。提交名也很直白：`同步加载修改ImportPackage优先级，优化加载卡顿问题`、`优化ALT&EDL下同步加载优先级`、`修改同步加载优先级的判断条件`。
>
> 还有一个全局开关：`GSyncLoaderOptimizationEnabled`。当前源码里默认值是 `false`，但历史上有过提交把它默认改成 true 后又可能被后续格式/合并覆盖；实际运行是否启用，要看你们分支最终值、命令行/配置是否还有外部改写。
>
> <details><summary>Searched the web</summary>
>
> Searched the web for `Unreal Engine 5 Zen Loader Async Loading 2`
>
> Searched the web for `"Zen Loader" "Unreal Engine"`
>
> </details>
>
> 公开搜索对这个内部实现名命中很差，Epic 源码很多在登录后的 GitHub 里。为了把“UE5 哪个版本”说得更稳，我准备读一下你机器上现有的 UE5 源码目录版本和对应 `FlushLoading` 实现，只读，不改任何东西。
>
> 本机 UE5 源码是 5.5.0。虽然 PowerShell profile 报了执行策略警告，但文件读取成功；我继续只读对照 UE5.5 的 `AsyncLoading2`，看 Flush 是否仍然是阻塞式。
>
> UE5.5 仍然有 `FlushAsyncLoading`，并且 `FAsyncLoadingThread2::FlushLoading` 仍是 while 循环驱动加载直到请求完成，不是“调用后立即返回”的异步 API。UE5 的改进更像缩短/隔离阻塞面，而不是取消 Flush 阻塞语义。
>
> UE5.5 里另有 `FAsyncLoadingFlushContext`，这是“非阻塞等待加载完成”的辅助上下文：它用 ticker 每帧检查 `IsAsyncLoading`，完成后回调。但这不是替换 `FlushAsyncLoading()`，而是给调用方一个别再同步卡住主线程的选择。
>
> 这个 UE5 仓库有 5.0 到 5.5 的标签，太好了。接下来我用标签做只读抽样：看 `AsyncLoadingFlushContext` 从哪个版本存在，以及 `FlushLoading` 在各版本是否仍是阻塞循环。
>
> `AsyncLoadingFlushContext` 从 UE5.0 release 就存在了，不是 5.5 才加的。也就是说 UE5 一开始就有“不要同步 Flush、改成 ticker 等完成”的工具，但 `FlushAsyncLoading()` 本身在 5.x 仍保留同步阻塞语义。
</details>

**1. 本项目相对 UE4.27 原生的定制点**

当前仓库基线是 `UE4.27.2`：见 [Build.version](./Engine/Build/Build.version:1)。

主要定制集中在“同步加载触发的异步加载抢占”：

- 增加 `GSyncLoaderOptimizationEnabled` 全局开关，用来控制同步加载优化路径：[CoreGlobals.cpp](./Engine/Source/Runtime/Core/Private/Misc/CoreGlobals.cpp:324)。
- 同步 `LoadPackageInternal` 不再只是普通优先级异步加载后 flush，而是在开关启用时使用 `FAsyncLoadEvent::UserPriority_SyncLoad`：[UObjectGlobals.cpp](./Engine/Source/Runtime/CoreUObject/Private/UObject/UObjectGlobals.cpp:1138)。
- 已在加载中的包如果被同步加载依赖，会递归提升其 import package 优先级，避免同步加载等待低优先级依赖慢慢跑：[AsyncLoading.cpp](./Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncLoading.cpp:413)、[AsyncLoading.cpp](./Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncLoading.cpp:3805)。
- 增加 `AsyncPackagesReadyForTick` 的同步加载优先排序和 `ForceFlushAsyncPackagesForTickCount`，在同步包进入 PostLoad 阶段后，暂缓普通 EDL 事件队列，优先完成同步包 PostLoad：[AsyncLoadingThread.h](./Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncLoadingThread.h:227)、[AsyncLoading.cpp](./Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncLoading.cpp:3778)、[AsyncLoading.cpp](./Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncLoading.cpp:4168)。
- 增加 `s.SyncLoaderOptimizationLogEnabled` 和一批 `SyncLoaderOptimizationLogPrint`，用于追同步加载链路：[AsyncLoading.cpp](./Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncLoading.cpp:192)。

简化说：不是重写异步加载框架，而是在 UE4.27 的 ALT + EDL 路径上做了“同步加载优先级抢占”和“同步 PostLoad 优先完成”的卡顿优化。

**2. FlushAsyncLoading 为什么阻塞主线程**

因为它的语义就是同步屏障：调用返回时，目标 request 或全部 async loading 必须完成。实现上 `FlushAsyncLoading` 直接调用 `GPackageLoader->FlushLoading`：[AsyncPackageLoader.cpp](./Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncPackageLoader.cpp:618)。

旧 loader 里 `FlushLoading` 会在主线程循环：

```text
while (IsAsyncLoadingPackages())
    TickAsyncLoading(false, false, 0)
```

见 [AsyncLoading.cpp](./Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncLoading.cpp:7082)。

它阻塞的核心原因：

- `Flush` 禁用了时间片预算，必须一直 tick 到完成。
- 很多 UObject 阶段必须在 Game Thread 做，比如非线程安全 `PostLoad`、completion callback、cluster 创建。
- 如果 I/O 未完成，主线程会等 ALT/I/O 推进。
- 同步 `LoadObject/LoadPackage` 路径会 `LoadPackageAsync` 后立刻 `FlushAsyncLoading(RequestID)`，所以它天然会造成卡顿窗口。

**3. UE5 哪个版本修了这个阻塞问题**

如果说“让 `FlushAsyncLoading()` 本身不阻塞”，结论是：我没看到 UE5 修成非阻塞。你本机 UE5.5.0 里它仍然调用 `GPackageLoader->FlushLoading`：[UE5 AsyncPackageLoader.cpp](F:/UE/MyUE_git/UE5/Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncPackageLoader.cpp:298)，`FAsyncLoadingThread2::FlushLoading` 仍然是阻塞式推进：[UE5 AsyncLoading2.cpp](F:/UE/MyUE_git/UE5/Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncLoading2.cpp:9789)。

UE5 从 `5.0.0-release` 起就有 `FAsyncLoadingFlushContext`，它用 ticker 每帧检查加载是否完成并回调，不阻塞当前帧：[UE5 AsyncLoadingFlushContext.cpp](F:/UE/MyUE_git/UE5/Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncLoadingFlushContext.cpp:28)。这算是“提供非阻塞等待 async loading 完成的替代工具”，但不是修掉 `FlushAsyncLoading` 的阻塞语义。

所以版本判断可以这样记：

- **UE5.0 起**：有 `AsyncLoading2`/IoStore 体系和 `FAsyncLoadingFlushContext` 这类降低同步等待的方案。
- **到 UE5.5.0 仍然如此**：`FlushAsyncLoading()` 本身仍是同步阻塞 API。真正规避卡顿，需要减少调用它，或把调用点改成异步等待/回调式流程。

> 对于你上述的第1点，我提几个问题：
> 问题1：同步 LoadPackageInternal 不再只是普通优先级异步加载后 flush，这句话如何理解？我理解这个描述是针对UE4.27.2原生的逻辑，对么？
> 问题2：已在加载中的包如果被同步加载依赖，会递归提升其 import package 优先级。这句话如何理解呢？
> 问题3：避免同步加载等待低优先级依赖慢慢跑。这句话如何理解，对于一个资源是否加载完成的判断条件是什么？所有的依赖项都要完成么？
> 问题4：EDL事件是指什么算法，在这里又是什么作用呢？
> 对于你上述的第2点，我提几个问题：
> 问题1：难道只要有同步加载资源就会flush卡顿么？有没有其他的先决条件。例如：当前异步加载队列积攒到某种程度？
> 对于你上述的第3点，我提几个问题：
> 问题1：对于UE5以上的版本而言的话，你可以再搜索一下网路上的博客，以及官方的帖子。查看异步加载逻辑相对于UE4而言增加了哪些优化以及特性。

<details><summary>2 previous messages</summary>

> 我会继续按探索模式回答，这次会把“我前面那几句话”拆成更精确的代码语义，并补充 UE5 公开资料。先快速对照本地修改前后的代码，再去搜官方和博客资料。
>
> <details><summary>Searched the web</summary>
>
> Searched the web for `Unreal Engine 5 Async Loading 2 IoStore package store optimization blog`
>
> Searched the web for `site:dev.epicgames.com/documentation/en-us/unreal-engine "Zen Loader"`
>
> Searched the web for `Unreal Engine 5.0 release notes IoStore Async Loading`
>
> Searched the web for `UE5 异步加载 AsyncLoading2 IoStore 博客`
>
> Searched the web for `Unreal Engine documentation Zen Storage Server Io Store`
>
> Searched the web for `"Use Io Store" "Unreal Engine"`
>
> Searched the web for `https://dev.epicgames.com/documentation/en-us/unreal-engine/packaging-your-project`
>
> Searched the web for `https://dev.epicgames.com/documentation/en-us/unreal-engine/packaging-your-project`
>
> Searched the web for `"AsyncLoading2" "PackageStore" "UE5"`
>
> Searched the web for `"UE5" "AsyncLoading2.cpp"`
>
> Searched the web for `Unreal Engine 5 Io Store performance blog`
>
> Searched the web for `Unreal Engine documentation Asset Manager asynchronous loading streamable manager`
>
> Searched the web for `https://dev.epicgames.com/documentation/en-us/unreal-engine/asset-management-in-unreal-engine`
>
> Searched the web for `https://dev.epicgames.com/documentation/en-us/unreal-engine/world-partition-in-unreal-engine`
>
> Searched the web for `UE5 traversal stutter asset loading blog IoStore World Partition`
>
> Searched the web for `"AsyncLoading2" "源码"`
>
> Searched the web for `"FAsyncLoadingThread2" "Unreal"`
>
> </details>
</details>

下面我按你的问题逐个修正和展开。先说一句：我前面第 1 点有一句话略容易误导，更准确的说法是：

> UE4.27 原生同步加载路径本来就可能是 `LoadPackageAsync(...) + FlushAsyncLoading(RequestID)`；你们项目定制的核心不是“首次引入 async+flush”，而是把同步加载请求及其依赖包的优先级提高，并在 EDL 调度里优先完成同步包的 PostLoad。

**第1点的问题**

问题1：`LoadPackageInternal 不再只是普通优先级异步加载后 flush` 怎么理解？

对，是针对 UE4.27.2 原生逻辑的对比，但要精确一点。

原生 UE4.27 的 cooked/EDL 同步加载，大致是：

```cpp
RequestID = LoadPackageAsync(..., Priority = UserPriority0);
FlushAsyncLoading(RequestID);
```

你们当前项目变成：

```cpp
const TAsyncLoadPriority LoadPriority =
    GSyncLoaderOptimizationEnabled
        ? FAsyncLoadEvent::UserPriority_SyncLoad
        : FAsyncLoadEvent::UserPriority0;

RequestID = LoadPackageAsync(..., LoadPriority);
FlushAsyncLoading(RequestID);
```

位置在 [UObjectGlobals.cpp](./Engine/Source/Runtime/CoreUObject/Private/UObject/UObjectGlobals.cpp:1138)。

所以这句话的意思不是“UE4 原生同步加载不用 async”，而是：

```text
UE4 原生：同步加载请求也可能走 async request，但优先级普通。
本项目：同步加载请求被标成 SyncLoad 高优先级，再 flush 等它完成。
```

问题2：已在加载中的包如果被同步加载依赖，会递归提升其 import package 优先级，怎么理解？

场景是这样的：

```text
普通异步队列里已经有 A、B、C 正在慢慢加载
某一帧同步加载 X
X 依赖 A
A 又依赖 B、C
```

如果只把 X 提高优先级，但 A/B/C 仍然是普通低优先级，那么 `FlushAsyncLoading(X)` 还是要等 A/B/C。于是项目新增了递归逻辑：

```text
同步请求 X
  └─ 发现依赖包 A
       └─ 提升 A priority
            └─ 扫 A 的 ImportMap
                 ├─ 提升 B priority
                 └─ 提升 C priority
```

代码在 [AsyncLoading.cpp](./Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncLoading.cpp:3805)。它会遍历 `Linker->ImportMap`，找到 import package 对应的 `FAsyncPackage`，再 `ModifyPriorityRecursively(ImportPackage, NewPriority + 1)`。

问题3：避免同步加载等待低优先级依赖慢慢跑，资源完成判断是什么？所有依赖都要完成么？

不是“项目里所有依赖”，而是当前包的加载闭包里必须满足加载语义的依赖，主要是 hard import / linker import 这条链。Soft reference 不会天然全部加载，除非被显式请求、Asset Bundle 请求，或运行中访问触发同步加载。

旧 loader 里一个包要被认为足够完成，关键条件包括：

- 自己的 export 已创建并序列化。
- `PostLoad` 已处理完，尤其是 `DeferredPostLoadObjects` 全部完成。
- 依赖 import package 的 export 也已序列化。
- 相关对象不再带 `RF_NeedLoad` / `RF_NeedPostLoad`。
- completion callback 可以安全触发。

你们代码里 `AreAllDependenciesFullyLoadedInternal` 的检查非常直观：如果 import package 的 `bAllExportsSerialized` 为 false，或者 `DeferredPostLoadIndex` 还没走完，或者 export object 仍有 `RF_NeedLoad | RF_NeedPostLoad`，就返回 false：[AsyncLoading.cpp](./Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncLoading.cpp:4375)。

所以更准确地说：

```text
同步加载 X 完成
= X 自己完成
+ X 的 hard/import 依赖链达到“可安全使用”的加载阶段
+ 相关 PostLoad / DeferredPostLoad 完成
```

问题4：EDL 事件是什么算法？这里的作用是什么？

EDL 是 Event Driven Loader。你可以把它理解为“加载阶段 DAG 调度器”。

传统非 EDL 更像线性状态机：

```text
CreateLinker -> FinishLinker -> LoadImports -> CreateImports
-> CreateExports -> PreLoadObjects -> PostLoad
```

EDL 会把这些阶段拆成事件节点和依赖边：

```text
Package_CreateLinker
  -> Package_FinishLinker
      -> Package_SetupImports
      -> Package_SetupExports
          -> Import/Create/Serialize nodes
          -> Export/Create/Serialize nodes
              -> ExportsDone
                  -> PostLoadWait
                      -> StartPostLoad
```

它本质上做的是：

```text
依赖图 + 优先级事件队列 + 拓扑推进
```

作用是让多个 package 的 import/export 创建、I/O、序列化可以交错执行，同时保证依赖顺序正确。项目里的 `EventQueue.PopAndExecute` 就是在跑普通 EDL 事件；你们新增的同步加载优化，会在 `ForceFlushAsyncPackagesForTickCount > 0` 时暂缓普通事件队列，优先处理 `AsyncPackagesReadyForTick` 里的同步 PostLoad：[AsyncLoading.cpp](./Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncLoading.cpp:4168)。

**第2点的问题**

问题1：只要有同步加载资源就会 flush 卡顿么？有没有其他先决条件？

不一定。同步加载不等于必然明显卡顿。

会不会卡，取决于这些条件：

- 资源是否已经完全加载。如果已经 loaded / postloaded，基本不会卡。
- 是否进入 cooked + EDL 下的 `LoadPackageAsync + FlushAsyncLoading(RequestID)` 路径。
- 目标包是否冷加载，需要磁盘 I/O、反序列化、创建对象、PostLoad。
- 目标包是否依赖其他还没完成的 import package。
- 当前异步队列是否有积压。积压不是必要条件，但会放大卡顿。
- 目标包或依赖包是否已经在队列中以低优先级加载。
- `PostLoad` 里是否又触发新的 `LoadObject` / `LoadPackage`。
- 是否有非线程安全 PostLoad 必须回到 Game Thread。

所以更准确是：

```text
同步加载未完成资源时，会创建一个“必须现在完成”的屏障。
如果这个屏障背后还有 I/O、序列化、依赖包、PostLoad 或队列积压，就会卡。
```

哪怕队列没有积压，一个很大的冷资源也会卡。队列积压只是让它更糟糕。你们这套同步优先级优化主要解决的是“同步包被普通异步队列拖住”的情况。

**第3点的问题**

我又查了官方资料，也对照了你机器上的 UE5.5 源码。公开博客里低层 `AsyncLoading2` 源码分析命中很少，官方文档更多讲上层系统，但可以归纳出 UE5 相对 UE4 的几个方向：

- **IoStore / Zen Store 容器化**：UE5 packaging 支持 `.utoc/.ucas` IoStore 容器和 Zen Store，而不是只围绕 `.pak`。官方 Packaging Settings 写明 `Use Io Store` 使用 `.utoc/.ucas`，`Use Zen Store` 使用 Zen storage server：[Epic Project Settings](https://dev.epicgames.com/documentation/unreal-engine/project-section-of-the-unreal-engine-project-settings)。
- **AsyncLoading2 更主流**：UE5 源码里低层加载以 `FAsyncLoadingThread2`、`FPackagePath`、`FPackageStore`、`FIoDispatcher`、export bundle event 为主；比 UE4.27 旧 loader 的 `FLinkerLoad + ImportMap/ExportMap` 线性推进更结构化。
- **Flush 语义没有消失，但更结构化**：UE5.5 仍然有阻塞式 `FlushAsyncLoading`；但源码里增加了 sync load context、多个 request id flush、visibility filter、潜在 stall warning 等机制：[UE5 AsyncLoading2.cpp](F:/UE/MyUE_git/UE5/Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncLoading2.cpp:9789)。
- **有非阻塞等待工具**：UE5 从 5.0 起已有 `FAsyncLoadingFlushContext`，它不是直接同步 flush，而是 ticker 每帧检查 async loading 是否完成，完成后回调：[UE5 AsyncLoadingFlushContext.cpp](F:/UE/MyUE_git/UE5/Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncLoadingFlushContext.cpp:28)。
- **World Partition 改变了上层加载粒度**：UE5 官方强调 World Partition 会把大世界拆成 grid cells，并根据 streaming source 距离自动加载/卸载；这减少了手工 sublevel 管理，也让运行时加载粒度更细：[Epic World Partition](https://dev.epicgames.com/documentation/en-us/unreal-engine/world-partition-in-unreal-engine)。
- **Asset Manager / StreamableManager 仍是推荐上层异步接口**：官方文档说 `Streamable Manager` 执行异步加载，并通过 Streamable Handles 保持对象驻留：[Epic Asset Management](https://dev.epicgames.com/documentation/en-us/unreal-engine/asset-management-in-unreal-engine)。

一句话总结 UE5：

```text
UE5 没有把 FlushAsyncLoading 本身变成非阻塞。
它主要是通过 IoStore + AsyncLoading2 + 更细的 World Partition 流送 + 非阻塞等待工具，
减少你不得不同步 flush 的场景，并降低每次加载的粒度和调度混乱度。
```

对你们 UE4.27 项目来说，当前这套同步加载优先级优化，正是在 UE4.27 旧 EDL 路径上补一块 UE5 更系统化处理的问题：同步请求来了，别让它被普通异步加载队列慢吞吞地拖住。

> 那我是不是可以这么理解：当前异步队列中有一些待加载资源，此时进来了一个同步加载资源，那么如果在异步队列中存在同步加载资源所依赖的资源，对于同步加载资源完成的时机就要必须使它所依赖的资源加载完成，也就是要等待异步队列中某个资源加载完成，所以带来了Flush卡顿，是么？那如果异步队列中不存在同步加载资源所依赖的资源呢？是不是就不会发生卡顿了。请你先重述我的问题，然后再回答。

你的问题我重述一下：

你是在问：当前异步加载队列里已经有一些资源在加载，这时又来了一个同步加载资源 `X`。如果异步队列里正好有 `X` 依赖的资源 `A/B/C`，那么 `X` 要完成就必须等这些依赖也加载完成，于是 `FlushAsyncLoading(X)` 会等队列里的依赖资源跑完，从而造成卡顿。这个理解是否正确？进一步，如果异步队列里没有 `X` 依赖的资源，是不是就不会卡顿？

回答：你的理解是对的，但还不完整。

更准确是：

```text
同步加载 X
  -> LoadPackageAsync(X)
  -> FlushAsyncLoading(RequestID of X)
  -> 等 X 达到可用状态
       -> X 自己要加载完成
       -> X 的 hard/import 依赖也要达到可用状态
       -> 必要的序列化、PostLoad、DeferredPostLoad 要完成
```

如果异步队列里已经有 `X` 依赖的资源，比如：

```text
异步队列已有：A、B、C、D
同步加载：X
X 依赖：B、C
```

那么 `FlushAsyncLoading(X)` 不能只等 `X` 自己，它必须等 `B/C` 也推进到满足依赖条件。这个时候如果 `B/C` 原本在异步队列中优先级低、进度慢，就会把同步加载卡住。你们项目的定制优化正是为了解这个问题：同步加载 `X` 来了以后，把 `X` 以及 `X` 依赖链上的包提升优先级，别让它们排在普通异步资源后面慢慢跑。

但第二个判断：“如果异步队列中不存在同步加载资源依赖的资源，是不是就不会卡顿？”答案是：不一定。

因为 Flush 卡顿的根本原因不是“队列中已有依赖”，而是“同步加载要求当前帧等资源变成可用”。即使异步队列里没有 `X` 的依赖，也可能卡：

```text
情况 A：X 已经完全加载
-> 基本不卡，Flush 很快返回。

情况 B：X 未加载，但没有复杂依赖
-> 仍要读取 X、创建对象、序列化、PostLoad，可能卡。

情况 C：X 未加载，依赖不在队列里
-> 加载 X 时发现依赖 A/B/C，现场把依赖加入加载流程，仍然要等它们完成，可能卡。

情况 D：X 未加载，异步队列里有大量无关资源
-> 理论上 Flush(RequestID) 目标是等 X，不是等所有无关包；但调度、ALT、EDL 事件、I/O、主线程 PostLoad 仍可能被已有工作影响。
```

所以可以这样理解：

```text
“异步队列里已有 X 的依赖”是 Flush 卡顿的放大器，
但不是 Flush 卡顿的必要条件。
```

真正的必要条件更像是：

```text
同步加载请求的目标资源此刻还没达到可安全使用状态。
```

只要它还没 ready，`FlushAsyncLoading(RequestID)` 就会把当前线程变成“加载推进器”，一直推进到目标 request 完成。队列里有没有依赖，只决定它要等的链路有多长、调度是否会被低优先级资源拖住。

> [$openspec-explore](F://Pan1//UE_git//.agents//skills//openspec-explore//SKILL.md)
> 你来梳理下对于上层业务的各种情况，最好用案例说明

<details><summary>2 previous messages</summary>

> 我会继续按 `openspec-explore` 做只读梳理：这次重点不再钻底层实现，而是把上层业务“什么时候会触发异步、同步、Flush、卡顿”的场景分门别类，用案例把链路讲清楚。
>
> 当前没有 active OpenSpec change。搜索出来的上层入口很典型：关卡流送、StreamableManager/软引用、UMG 图片异步加载、SoundWave 异步加载、网络 NetGUID 按路径加载，以及大量 `LoadObject/TryLoad/LoadSynchronous` 同步路径。
</details>

可以，我们把“上层业务”按触发方式梳理成几类。这里先给一个总图：

```text
业务代码想用资源
        │
        ├─ 资源已经在内存里
        │      └─ 直接用，基本不卡
        │
        ├─ 可以晚点用
        │      └─ RequestAsyncLoad / LoadPackageAsync
        │             └─ 回调里使用，通常不卡当前帧
        │
        └─ 必须现在用
               └─ LoadObject / TryLoad / LoadSynchronous / blocking level load
                      └─ 可能触发 FlushAsyncLoading / WaitUntilComplete
                             └─ 当前帧等待资源、依赖、PostLoad 完成
```

**场景 1：资源已加载，业务只是拿引用**

案例：UI 图片设置软纹理时，`UImage::RequestAsyncLoad` 先判断 `SoftObject.Get()`。如果对象已经在内存里，直接执行回调，不走加载：[Image.cpp](./Engine/Source/Runtime/UMG/Private/Components/Image.cpp:373)。

```text
背包图标 A 已经被前一个界面加载过
再次打开界面
SoftObject.Get() 成功
直接 SetBrushFromTexture
不卡
```

这类最安全。所谓“同步加载卡顿”，前提通常是目标资源还没 ready。

**场景 2：软引用异步加载，业务接受延迟**

案例：`UImage::SetBrushFromSoftTexture` 用 `RequestAsyncLoad`，加载完成后回调里再 `SetBrushFromTexture`：[Image.cpp](./Engine/Source/Runtime/UMG/Private/Components/Image.cpp:424)。

```text
商城打开
头像框、商品图标还没加载
先显示占位图
RequestAsyncLoad(icon)
加载完成回调替换图片
当前帧不卡
```

这类是推荐业务形态。资源没好就先不要用，等回调。

风险点是：回调里又同步加载别的资源，或者 UI 要求“这一帧必须显示真实图”，就会从异步退化成同步。

**场景 3：软引用同步加载，业务必须马上拿到对象**

案例：`FStreamableManager::LoadSynchronous` 会走 `RequestSyncLoad`，内部先发 request，然后 `WaitUntilComplete`：[StreamableManager.cpp](./Engine/Source/Runtime/Engine/Private/StreamableManager.cpp:1245)。

```text
技能释放时才发现特效资源没加载
业务代码调用 EffectPath.LoadSynchronous()
必须马上 Spawn 特效
引擎只能等资源完成
可能卡一帧甚至多帧
```

这里即使异步队列里没有它的依赖，也可能卡，因为资源本身要读盘、创建 UObject、序列化、PostLoad。

更糟的情况是：

```text
同步加载 SkillEffect
SkillEffect hard 依赖 Material、Texture、Niagara、Sound
这些依赖部分已在异步队列低优先级加载
Flush 必须等它们也完成
卡顿放大
```

你们当前的“同步加载优先级优化”主要就是为了这个场景。

**场景 4：关卡流送，后台异步加载**

案例：`ULevelStreaming::RequestLevel` 发 `LoadPackageAsync`，完成后走 `AsyncLevelLoadComplete`：[LevelStreaming.cpp](./Engine/Source/Runtime/Engine/Private/LevelStreaming.cpp:1062)。

```text
玩家接近新区域
后台 LoadPackageAsync(SubLevel_A)
当前帧继续游戏
加载完成后再切 visibility / 注册 actor
```

这类通常不卡在发起加载的那一帧，但后续仍可能在 actor 注册、组件注册、可见性切换时产生分帧压力。

**场景 5：关卡流送，但业务要求阻塞**

同一个函数里，如果 `BlockPolicy == AlwaysBlock`，或者 always loaded level 需要阻塞，它会直接 `FlushAsyncLoading()`：[LevelStreaming.cpp](./Engine/Source/Runtime/Engine/Private/LevelStreaming.cpp:1068)。

注意这里是无参数 `FlushAsyncLoading()`，不是只 flush 某个 request。

```text
切地图 / 强制加载常驻子关卡
业务要求加载完成才能继续
FlushAsyncLoading()
可能把当前所有异步加载都推进完
大概率卡
```

这是比 `FlushAsyncLoading(RequestID)` 更危险的形态，因为它不是只等目标包，而是“清场”。

**场景 6：PostLoad 里触发同步加载**

这是很容易藏住的卡顿点。`StaticLoadObject` 检测到在 async loading 的 PostLoad 里调用，会直接打 warning：[UObjectGlobals.cpp](./Engine/Source/Runtime/CoreUObject/Private/UObject/UObjectGlobals.cpp:914)。

```text
加载角色蓝图
某个组件 PostLoad
PostLoad 里 LoadObject 一个材质 / 数据表 / 音效
引擎正在异步加载，却被迫插入新的同步需求
产生 hitch
```

底层 `PostLoadDeferredObjects` 里也写得很明白：如果 PostLoad 期间加载了新对象，会 blocking preload 它们：[AsyncLoading.cpp](./Engine/Source/Runtime/CoreUObject/Private/Serialization/AsyncLoading.cpp:6551)。

业务建议是：PostLoad 只做轻量修复，不做资源拉取。需要资源的，提前 hard reference、预加载，或延迟到 BeginPlay/初始化流程异步处理。

**场景 7：音频资源的折中处理**

`USoundNodeWavePlayer::LoadAsset` 很有代表性：[SoundNodeWavePlayer.cpp](./Engine/Source/Runtime/Engine/Private/SoundNodeWavePlayer.cpp:34)。

```text
如果当前正在 async loading：
    尽量 LoadPackageAsync(sound package)
    避免直接同步拿半加载对象

否则：
    SoundWaveAssetPtr.LoadSynchronous()
```

案例：

```text
SoundCue 正在 PostLoad
里面引用的 SoundWave 还没序列化完
此时不能直接用 SoundWave
否则可能拿到 RF_NeedLoad 的半成品
```

这说明上层业务有时不是“想异步就异步”，还要考虑对象是否已经达到可安全使用状态。

**场景 8：构造函数 / 初始加载期 / async 回调里再加载**

`StreamableManager` 里有这个判断：[StreamableManager.cpp](./Engine/Source/Runtime/Engine/Private/StreamableManager.cpp:1080)。

```cpp
if (GIsInitialLoad || ThreadContext.IsInConstructor > 0 || bForceSynchronousLoads)
{
    StaticLoadObject(...)
}
else
{
    LoadPackageAsync(...)
}
```

案例：

```text
UObject 构造函数里 LoadSynchronous 某资源
或 CDO 初始化时拉资源
或 async loading callback 里再递归请求同步资源
```

这些位置引擎会更保守，可能强制同步，因为递归 tick async loading 不安全。

**场景 9：网络路径按 NetGUID / asset path 加载**

`PackageMapClient` 会根据网络同步过来的路径 `LoadPackageAsync`：[PackageMapClient.cpp](./Engine/Source/Runtime/Engine/Private/PackageMapClient.cpp:3066)。

案例：

```text
服务器复制一个客户端尚未加载的资产引用
客户端发现本地没有
异步加载 package
加载完成后 resolve NetGUID
```

这类通常是异步友好的。但如果业务随后立刻要求对象可用，或者 fallback 到 `StaticLoadObject`，就会转成同步风险。

**场景 10：系统级 Flush，不一定由业务显式加载触发**

这些不是普通业务加载，但业务行为可能间接触发：

```text
GC 前后
ResetLoaders
Package reload
SavePackage
退出游戏 / 切世界
```

例如 GC 中会在 async loading 时调用 `FlushAsyncLoading`，保证对象状态一致。这类卡顿看起来像“GC 卡”，但根因可能是前面业务积累了未完成异步加载。

**几个业务判断口诀**

1. `RequestAsyncLoad` 不等于绝对不卡
它只是“不在发起点等待”。回调、PostLoad、后续使用仍可能卡。

2. `LoadSynchronous/TryLoad/LoadObject` 在运行时热点路径要高度警惕
尤其是战斗中、滑动列表中、PostLoad 中、Tick 中。

3. `FlushAsyncLoading(RequestID)` 是等目标请求和必要依赖
卡顿范围相对可控。

4. `FlushAsyncLoading()` 无参数是清空所有异步加载
这类通常更危险，关卡流送 block policy 就是典型例子。

5. 异步队列里有没有目标依赖，只决定卡顿是否被放大
即使没有依赖，冷资源同步加载也可能卡。

6. 你们项目优化解决的是“同步请求被普通异步队列拖住”
不是让同步加载消失，也不是让 Flush 不阻塞。

**一个完整案例**

```text
玩家打开角色详情页
页面需要角色模型、皮肤材质、武器图标、技能特效预览
```

较差写法：

```text
打开页面当帧：
LoadSynchronous(CharacterMesh)
LoadSynchronous(SkinMaterial)
LoadSynchronous(WeaponIcon)
LoadSynchronous(SkillEffect)
```

可能结果：

```text
每个资源都可能触发 LoadPackageAsync + Flush
依赖链互相穿插
PostLoad 里又加载材质函数/纹理
一帧明显卡顿
```

较好写法：

```text
进入详情页前：
RequestAsyncLoad(页面资源列表)

打开页面：
已加载的直接显示
未加载的显示占位 / loading
回调里逐步替换
```

如果业务硬要求“打开详情页必须完整显示”，那就把同步加载前移：

```text
点击角色按钮后先进入过渡/loading
异步加载完成
再切详情页
```

本质是把不可避免的等待从“交互关键帧”挪到“用户能接受等待的阶段”。这比单纯调 loader 优先级更稳。

> 【同步加载 SkillEffect
> SkillEffect hard 依赖 Material、Texture、Niagara、Sound
> 这些依赖部分已在异步队列低优先级加载
> Flush 必须等它们也完成
> 卡顿放大】
> 对于这种情况，是会把异步队列全部加载完成么？还是指将依赖部分加载完成即可。

这种情况通常是：**只需要把同步加载目标及其依赖链加载完成即可，不是必须把整个异步队列全部加载完。**

更具体地说，如果调用的是：

```cpp
FlushAsyncLoading(RequestID);
```

它的目标是等这个 `RequestID` 对应的 package 完成。为了让 `SkillEffect` 完成，引擎会继续推进异步加载系统，但完成条件主要围绕：

```text
SkillEffect
  ├─ Material
  │   └─ Texture
  ├─ Niagara
  └─ Sound
```

这些 hard/import 依赖需要达到可用状态。异步队列里其他无关资源，比如：

```text
MapChunk_03
LobbyIcon_20
EnemyMesh_B
```

理论上不是 `SkillEffect` 的依赖，不是这个 request 的完成条件。

但是有两个现实细节：

1. `Flush` 期间引擎是在推进整个 async loading 系统
所以它可能顺手执行一些无关 package 的事件、I/O 回调、PostLoad 片段。不是目标要求它们完成，而是调度过程可能会碰到。

2. 无参数 `FlushAsyncLoading()` 不一样
如果调用的是无参数版本，它就是清空当前异步加载，所有队列里的包都要完成。这种会比 `FlushAsyncLoading(RequestID)` 更容易大卡。

所以可以这样记：

```text
FlushAsyncLoading(RequestID)
= 等目标资源 + 必要依赖链完成
= 可能顺手推进无关资源
≠ 必须清空整个异步队列

FlushAsyncLoading()
= 等当前所有异步加载完成
= 清空队列
```

在你举的 `SkillEffect` 例子里，如果是同步 `LoadObject/LoadPackageInternal` 触发的，一般走的是带 `RequestID` 的 flush，因此核心等待的是 `SkillEffect` 依赖链。

> 异步加载和同步加载都会进入，同一个加载队列中么？

是的，**大体可以理解为都会进入同一套底层 package loading 系统/队列**，但它们的“等待方式”和“优先级”不同。

```text
异步加载：
LoadPackageAsync / RequestAsyncLoad
  -> 进入 async loading 队列
  -> 后台或分帧推进
  -> 完成后回调

同步加载：
LoadObject / TryLoad / LoadSynchronous
  -> 如果目标 package 需要加载
  -> 也可能先转成 LoadPackageAsync
  -> 进入同一套 async loading 队列
  -> 然后立刻 FlushAsyncLoading(RequestID) 等它完成
```

所以差别不是“同步加载完全走另一套队列”，而是：

```text
异步加载：排队后不等，后面回调。
同步加载：排队后马上等，当前线程阻塞。
```

在你们项目的定制逻辑里，同步加载还会带更高优先级：

```cpp
GSyncLoaderOptimizationEnabled
    ? FAsyncLoadEvent::UserPriority_SyncLoad
    : FAsyncLoadEvent::UserPriority0
```

也就是说，同步加载请求可能和普通异步加载请求进入同一套队列/事件系统，但会被标记成更高优先级，依赖链也可能被递归提权。

补一个细节：不是所有同步加载都会进入队列。如果资源已经在内存中，或者某些编辑器/构造期路径直接 `StaticLoadObject` 完成，它可能不需要排队。但在 cooked + EDL 这类路径下，同步加载常见形态就是：

```text
LoadPackageAsync(...)
FlushAsyncLoading(RequestID)
```

> 对于UE原生的加载逻辑而言，异步加载队列中，进入了一个同步加载资源，那是不是要等同步加载资源前的资源处理完成后，才会轮到这个同步资源，是这样吗？以及同步加载资源所依赖的资源又是什么逻辑

对 UE 原生加载逻辑，可以这么理解，但要稍微修正一下：

**不是严格“同步资源前面的资源全部处理完，才轮到同步资源”。**
更准确是：

```text
同步加载资源也进入同一套 async loading 系统
然后立刻 FlushAsyncLoading(RequestID)
Flush 会持续推进加载系统，直到这个 RequestID 完成
```

在 UE4.27 原生里，如果同步加载请求和普通异步请求是同优先级，那么它确实**可能被前面已经排队或已经在执行的包拖住**。但这不是一个简单 FIFO 队列：

```text
不是：
A -> B -> C -> SyncX
必须 A/B/C 全部完成才到 SyncX

更像：
队列 + 优先级 + EDL事件依赖图 + 当前正在执行的包
Flush 时不断推进，直到 SyncX 完成
```

所以几种情况要分开看：

```text
1. SyncX 还只是排队，前面有很多同优先级包
   -> 原生逻辑下可能要先处理一部分前面的包，SyncX 才被创建/推进。

2. SyncX 已经创建成 FAsyncPackage
   -> 不一定等所有前面的包完成，但它的事件要等自身依赖满足。

3. 前面队列里的包和 SyncX 无关
   -> 理论上不是 SyncX 完成条件；但调度过程中可能仍被顺手推进。

4. 前面队列里的包正好是 SyncX 的依赖
   -> 那就必须等这些依赖达到可用状态。
```

同步资源依赖逻辑大致是：

```text
SyncX.uasset
  ├─ ImportMap / hard reference -> MaterialA
  ├─ MaterialA -> TextureB
  ├─ NiagaraSystemC
  └─ SoundD
```

这些依赖来自 package 的 import/export 关系，也就是 cooked 后 linker 能看到的 hard dependency。SyncX 要完成，不能只把 SyncX 自己读出来，还要保证它需要的 import package、export 对象、序列化、PostLoad 达到安全状态。

所以原生 UE4.27 的问题在于：

```text
普通异步包 A/B/C 正在低优先级加载
同步加载 SyncX 进来
SyncX 依赖 B/C
但 B/C 仍按普通异步优先级慢慢跑
FlushAsyncLoading(SyncX) 只能等
于是卡顿变大
```

你们项目的定制优化，就是让这个过程变成：

```text
发现 SyncX 是同步加载
提高 SyncX 优先级
递归提高 SyncX 依赖包 B/C 的优先级
优先推进同步包及其依赖的 PostLoad
```

一句话总结：

```text
UE 原生不是严格等“同步资源前面所有资源”完成，
而是等“同步资源及其必要依赖链”完成。
但因为原生同步请求优先级不够特殊，它可能被队列中已有的普通异步包或低优先级依赖拖慢。
```

> [$ai-conversation-archive](C:/Users/yongkang.guo//.codex/skills/ai-conversation-archive/SKILL.md)

<details><summary>1 previous message</summary>

> System error
>
> Error running remote compact task: unexpected status 401 Unauthorized: Missing bearer or basic authentication in header, url: https://api.openai.com/v1/responses, cf-ray: a4b4cef56d5a15f7-SJC, request id: req_eab39b92101b4b4b98b08e262ca8127f
</details>

> [$ai-conversation-archive](C:/Users/yongkang.guo//.codex/skills/ai-conversation-archive/SKILL.md)

<details><summary>1 previous message</summary>

> System error
>
> unexpected status 401 Unauthorized: Missing bearer or basic authentication in header, url: https://api.openai.com/v1/responses, cf-ray: a4b4d0629c77ee01-SJC, request id: req_c55a9ea329d34f30b2b46026dcc42361
</details>
