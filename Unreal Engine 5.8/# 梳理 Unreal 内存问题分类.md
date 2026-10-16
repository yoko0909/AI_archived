# 梳理 Unreal 内存问题分类

> [$grill-me](C://Users//yongkang.guo//.codex//skills//grill-me//SKILL.md)
> 关于Unreal Engine的内存问题，都有哪些类别？例如：内存泄漏等。我希望针对这些类别，写一些案例代码，并实际通过Memory Insight进行数据采集，进而定位解决。

<details><summary>6 previous messages</summary>

> 我会先读取你点名的 `grill-me` 技能说明，然后按它的方式把这个 UE 内存排查练习拆成可执行、可追问的方案。
>
> `grill-me` 实际上会转到 `/grilling` 流程，我继续读取那个技能，确保接下来的问题不是泛泛而谈，而是能逼近你要落地的 Memory Insights 实验设计。
>
> 我会先从本地 UE5.8 源码里确认 Memory Insights / LLM / trace 相关入口和命令名，这样后面的分类和案例不会飘在概念层。
>
> 本地搜索确认了几个关键点：UE 里有两条会交到 Memory Insights 的线，一条是 `MemoryTrace_*` 记录分配/释放和调用栈，另一条是 LLM tag 给内存归类。下一步我只抓命令行开关和最小可观测案例，不铺开读整套引擎。
>
> Memory Insights 这边的源码也显示它真正能帮我们看的维度包括：总分配曲线、Live Allocs、Alloc/Free 事件数、Swap、LLM Tag 树，以及按调用栈/线程/文件/函数/Asset/Package 导出分配表。这个很好，正适合设计“可复现小故障”。
>
> 我已经有足够上下文给出分类，但按 `grilling` 的约束，我会先把“推荐的训练范围”定下来，再一轮一轮追问。先给你一个可执行的骨架，然后只问第一个关键问题。
</details>

我建议把 UE 内存问题分成 8 类来做案例库：

1. **真正泄漏**：分配后永不释放，例如 `FMemory::Malloc`、`new`、第三方库内存没对应 `Free/delete`。
2. **UObject/资源引用泄漏**：对象本身可被 GC，但被 `UPROPERTY`、`AddToRoot`、Delegate、Timer、Subsystem、Singleton 持有导致不回收。
3. **容器膨胀/缓存无界增长**：`TArray/TMap` 只增不清，或 `Reset` 后容量仍保留，表现像泄漏。
4. **生命周期错配**：关卡切换、PIE 重启、World 销毁后仍有模块/线程/静态对象持有资源。
5. **瞬时峰值过高**：加载、反序列化、贴图/网格构建、批处理时短时间分配过猛，最终会释放但可能 OOM。
6. **内存碎片/分配器问题**：Live memory 不高，但 reserved/committed 上不去下不来，小块大量分配尤其明显。
7. **GPU/显存问题**：Texture、Buffer、RT、Nanite、RHI 资源未释放或峰值过高，走 `VideoMemory`/RHI 相关 trace。
8. **标记/归因缺失**：内存确实增长，但缺少 LLM tag、Asset metadata 或调用栈，Memory Insights 里看不出是谁。

针对 Memory Insights，最适合的练习路线是：先做 **CPU 系统内存可复现小案例**，因为它能稳定显示 `Alloc/Free`、`Live Allocs`、`Growth`、`Active Allocs`、调用栈、线程、源文件和 LLM Tag。之后再扩展到 UObject、资源、GPU。

**第 1 个问题：我们第一阶段要不要只做“C++ 插件/模块里的 CPU 内存问题案例”，暂时不碰蓝图和 GPU？**

我的推荐答案：**是**。先做一个 `MemoryIssueLab` 插件，提供控制台命令或 Editor Utility 按钮触发 4 个案例：真泄漏、容器容量滞留、瞬时峰值、生命周期错配。这样采集最稳定，也最容易在 Memory Insights 里定位到明确调用栈。

> 你详细介绍一下这4个案例

<details><summary>1 previous message</summary>

> 我会把它们按“故障形态、案例代码怎么写、Memory Insights 里看什么、怎么修”来讲，这样后面可以直接变成插件里的 4 个按钮/命令。
</details>

可以。下面这 4 个案例我建议都做成一个 `MemoryIssueLab` 插件里的控制台命令，例如 `MemLab.NativeLeak.Start`、`MemLab.ContainerSlack.Run` 这种，方便你录 trace、复现、修复、再录一遍对比。

**1. 真正 Native 内存泄漏**

故障形态：
代码通过 `FMemory::Malloc`、`new`、第三方 SDK 分配内存，但没有对应释放。Memory Insights 里会看到 Live Allocs 和 Total Allocated 持续增长，而且增长来源能回到同一个调用栈。

案例代码思路：

```cpp
static TArray<void*> LeakedBlocks;

void FMemoryIssueLab::NativeLeakTick()
{
    LLM_SCOPE_BYNAME(TEXT("MemoryIssueLab/NativeLeak"));

    constexpr SIZE_T BlockSize = 4 * 1024 * 1024;
    void* Ptr = FMemory::Malloc(BlockSize, 16);

    FMemory::Memset(Ptr, 0xCD, BlockSize);
    LeakedBlocks.Add(Ptr);
}
```

更狠一点的泄漏是连指针都不保存：

```cpp
void* Ptr = FMemory::Malloc(BlockSize, 16);
FMemory::Memset(Ptr, 0xCD, BlockSize);
// Ptr 丢失，之后无法释放
```

Memory Insights 里看什么：

- `Live Allocs` 曲线持续上升。
- `LLM MemoryIssueLab/NativeLeak` tag 持续上升。
- 在 A/B 两个时间点之间查 `Growth`，能看到新增且仍存活的分配。
- 按 `Alloc Callstack` 或 `Alloc Function` 分组，可以定位到 `NativeLeakTick`。
- 如果你导出 allocations，重点看 `Size`、`Tag`、`AllocFunction`、`AllocSourceFile`。

修复方式：

```cpp
for (void* Ptr : LeakedBlocks)
{
    FMemory::Free(Ptr);
}
LeakedBlocks.Reset();
```

真正项目里的修法通常是 RAII：`TUniquePtr`、`TArray<uint8>`、析构函数释放、模块 `ShutdownModule()` 释放、第三方句柄封装成 owner 类型。

---

**2. 容器容量滞留 / 缓存无界增长**

故障形态：
这类最容易被误判成“泄漏”。对象还活着，容器也还持有内存，只是业务认为“我已经清空了”。典型例子是 `TArray::Reset()` 清掉元素数量，但保留 capacity/slack；或者缓存 `TMap` 永远只增不淘汰。

案例代码思路：

```cpp
static TArray<uint8> ScratchBuffer;

void FMemoryIssueLab::ContainerSlack()
{
    LLM_SCOPE_BYNAME(TEXT("MemoryIssueLab/ContainerSlack"));

    ScratchBuffer.SetNumUninitialized(512 * 1024 * 1024);
    FMemory::Memset(ScratchBuffer.GetData(), 0, ScratchBuffer.Num());

    ScratchBuffer.Reset(); // Num 变 0，但容量可能仍保留
}
```

另一个无界缓存版本：

```cpp
static TMap<int32, TArray<uint8>> Cache;

void FMemoryIssueLab::UnboundedCache(int32 Key)
{
    LLM_SCOPE_BYNAME(TEXT("MemoryIssueLab/UnboundedCache"));

    TArray<uint8>& Entry = Cache.FindOrAdd(Key);
    Entry.SetNumUninitialized(8 * 1024 * 1024);
}
```

Memory Insights 里看什么：

- `Live Allocs` 上升后可能平台期，不一定一直涨。
- `Alloc/Free Events` 不一定异常高。
- `Active Allocs at time A` 能看到内存仍然活着。
- 调用栈会指向 `SetNumUninitialized`、`Reserve`、`FindOrAdd` 等容器扩容位置。
- LLM tag 显示业务归因，但它不会告诉你“这是泄漏还是缓存”，这个要结合生命周期判断。

修复方式：

```cpp
ScratchBuffer.Empty(); // 释放内存
```

或者更细：

```cpp
ScratchBuffer.Reset();
ScratchBuffer.Shrink();
```

缓存类问题要加策略：最大条目数、最大字节数、LRU、关卡切换清理、后台资源释放、调试命令输出当前缓存大小。

---

**3. 瞬时峰值过高**

故障形态：
没有泄漏，最终也释放了，但某一帧或某个加载阶段临时分配太多，导致卡顿、OOM、内存告警。常见于资源加载、临时解压、批量构建、一次性复制大数组。

案例代码思路：

```cpp
void FMemoryIssueLab::TransientPeak()
{
    LLM_SCOPE_BYNAME(TEXT("MemoryIssueLab/TransientPeak"));

    TArray<uint8> TempA;
    TArray<uint8> TempB;
    TArray<uint8> TempC;

    TempA.SetNumUninitialized(256 * 1024 * 1024);
    TempB = TempA; // 故意制造拷贝峰值
    TempC = TempB;

    FPlatformProcess::Sleep(2.0f); // 方便采集中看到峰值
}
```

Memory Insights 里看什么：

- 总内存曲线有尖峰，但之后回落。
- `Alloc Events` 和 `Free Events` 都很明显。
- 在峰值时间点查 `Active Allocs`，能看到一堆临时数组。
- 在 A/B 区间查短生命周期分配，可以看到“分配后很快释放”的热点调用栈。
- 这类问题不要只看最后的 live memory，否则会误以为没问题。

修复方式：

```cpp
TArray<uint8> Temp;
Temp.Reserve(ExpectedSize);

// 分批处理，而不是一次性创建多个大数组
for (...)
{
    Temp.Reset();
    // process chunk
}
```

更实际的修法包括：分块加载、流式处理、避免多份副本、用 move、预分配复用 scratch buffer、限制并发任务数量、把峰值阶段拆到多帧。

---

**4. 生命周期错配**

故障形态：
资源本该在 World、Level、Actor、Subsystem、模块结束时释放，但被静态对象、delegate、timer、异步任务、lambda 捕获、singleton 持住了。它不像第一类那样“裸指针丢了”，而是 owner 还活着，只是活得太久。

案例代码思路：

```cpp
static TArray<TArray<uint8>> PerWorldData;
static FDelegateHandle TickHandle;

void FMemoryIssueLab::StartLifecycleMismatch(UWorld* World)
{
    LLM_SCOPE_BYNAME(TEXT("MemoryIssueLab/LifecycleMismatch"));

    TArray<uint8>& Data = PerWorldData.AddDefaulted_GetRef();
    Data.SetNumUninitialized(128 * 1024 * 1024);

    TickHandle = FWorldDelegates::OnWorldTickStart.AddLambda(
        [DataPtr = &Data](UWorld*, ELevelTick, float)
        {
            // 捕获/引用让这批数据随静态容器一直活着
            volatile uint8 Value = (*DataPtr)[0];
        });
}
```

更贴近项目的版本：

- `AddToRoot()` 后忘记 `RemoveFromRoot()`。
- `FTimerManager` 里绑定对象但退出时不清。
- `AsyncTask` lambda 捕获 `TSharedPtr`，任务队列长期不结束。
- `Subsystem` 持有上一个 World 的资源。
- 模块静态缓存跨 PIE 多次累积。

Memory Insights 里看什么：

- 每次 PIE、关卡切换、进入/退出玩法后，基线越来越高。
- `Growth` 发生在加载/进入阶段，但退出阶段没有对应 `Decline`。
- `Active Allocs` 的调用栈指向创建资源的位置，但真正 bug 在“谁没释放/解绑”。
- LLM tag 能看到 `MemoryIssueLab/LifecycleMismatch` 在 World 销毁后仍存在。

修复方式：

```cpp
if (TickHandle.IsValid())
{
    FWorldDelegates::OnWorldTickStart.Remove(TickHandle);
    TickHandle.Reset();
}

PerWorldData.Empty();
```

UObject 场景则是：不用 `AddToRoot`，改用正确 `UPROPERTY` owner；解绑 delegate；`EndPlay`、`Deinitialize`、`ShutdownModule`、`OnWorldCleanup` 中释放；异步任务使用 `TWeakObjectPtr` 或取消机制。

这 4 个案例刚好覆盖四种不同诊断思路：
**持续增长找泄漏，平台期找缓存，尖峰找临时分配，跨生命周期增长找 owner/解绑问题。**

> [$grill-me](C://Users//yongkang.guo//.codex//skills//grill-me//SKILL.md)
> 我建议还是首先只做第一个。我们加两个指令，一个开始申请，一个释放内存。

<details><summary>3 previous messages</summary>

> 我会按你点名的 `grill-me` 来收敛设计，同时也会用 brainstorming 技能把这个“小实验功能”的边界问清楚。先把技能说明重新过一遍，然后我只问一个关键问题。
>
> 先扫一下当前 workspace 结构和已有插件/模块位置；这个能回答“应该放哪儿”一部分，不急着把问题丢给你。
>
> 我看到这是完整 UE5.8 源码树，已有 `Engine/Plugins/Developer/BlankPlugin` 可当模板，也有 `Engine/Plugins/MemoryUsageQueries` 这种相关但偏查询工具的插件。接下来按 grilling 的方式先钉一个会影响代码形态的问题。
</details>

第一个问题：这两个指令你希望运行在 **Editor 控制台** 里，还是也要能在 **Standalone/命令行游戏进程** 里运行？

我的推荐答案：**先做 Runtime 插件 + 控制台命令，Editor 和 Standalone 都可用。**

原因很实际：Memory Insights 采集最好用独立进程或 Standalone，噪音比 Editor 少；但开发时在 Editor 控制台里也能快速试。两个命令可以先定成：

```text
MemLab.NativeLeak.Alloc
MemLab.NativeLeak.Free
```

推荐行为：

```text
MemLab.NativeLeak.Alloc [MB] [Count]
```

默认每次申请 `64 MB * 1`，用 `FMemory::Malloc` 分配，保存到插件模块里的静态/成员数组中，并用：

```cpp
LLM_SCOPE_BYNAME(TEXT("MemoryIssueLab/NativeLeak"));
```

这样 Memory Insights 里可以按 LLM Tag 和调用栈定位。

```text
MemLab.NativeLeak.Free
```

释放当前所有已申请块，并清空数组。这样你能录两段对比：

1. 执行多次 `Alloc`，看 `Live Allocs`、`Growth`、`MemoryIssueLab/NativeLeak` 增长。
2. 执行 `Free`，看对应内存下降，验证定位和修复路径。

你确认一下：**我们按“Runtime 插件 + 这两个控制台命令 + 默认每次 64MB”的设计继续吗？**

> 将控制台命令改为Misc.MemoryLeak.Start 和 Misc.MemoryLeak.Free，其他我赞同

收到，命令名改成：

```text
Misc.MemoryLeak.Start
Misc.MemoryLeak.Free
```

第 2 个问题：`Misc.MemoryLeak.Start` 是 **执行一次就申请一批内存**，还是 **启动一个定时器持续申请内存，直到 Free**？

我的推荐答案：**执行一次申请一批内存**。
理由是 Memory Insights 里更容易读：每敲一次 `Start` 就出现一个明确的增长阶梯；多敲几次就能观察持续增长。`Free` 一次性释放全部块，下降也很清楚。

建议行为：

```text
Misc.MemoryLeak.Start [MB] [Count]
```

默认：

```text
Misc.MemoryLeak.Start
```

等价于申请 `64 MB * 1`。

例如：

```text
Misc.MemoryLeak.Start 128 4
```

申请 `4` 块，每块 `128 MB`，总计 `512 MB`。

你确认一下：`Start` 按“一次性申请一批，可重复执行叠加”的语义来做吗？

> 确认

<details><summary>1 previous message</summary>

> 好的，`Start` 就是一次性申请一批、可重复叠加；这会让 trace 上的阶梯非常干净。接下来我确认插件落点和模块类型，这个会决定是否影响 Engine 默认构建面。
</details>

第 3 个问题：这个实验插件放在 **Engine 插件目录**，还是放在某个具体项目的 `Plugins` 目录？

我的推荐答案：放在：

```text
Engine/Plugins/Developer/MemoryIssueLab
```

模块类型仍然用 `Runtime`，但插件 `EnabledByDefault=false`。

这样做的好处是：

- 不绑定某个 sample/project，后续任何项目都能手动启用来采集。
- 跟 `BlankPlugin` 这种开发辅助插件位置一致。
- `Runtime` 模块保证 Standalone 也能用控制台命令。
- `EnabledByDefault=false` 避免污染默认 Engine 行为。

你确认这个落点吗？

> 插件命名为MiscInterface

好，插件名改为：

```text
Engine/Plugins/Developer/MiscInterface
```

我建议模块名也同步叫：

```text
MiscInterface
```

命令仍然是：

```text
Misc.MemoryLeak.Start [MB] [Count]
Misc.MemoryLeak.Free
```

第 4 个问题：`Free` 执行后，我们是 **释放全部已申请内存**，还是支持参数只释放一部分？

我的推荐答案：**先只做释放全部**。
这个实验的核心是让 Memory Insights 里出现清楚的“申请阶梯”和“释放下降”，全部释放最直观，也减少误操作和状态复杂度。后续如果要做更细的案例，再加 `FreeLast` 或 `Free [Count]`。

> 释放一部分的话，要如何做呢？ 释放的参数是否要和申请的参数对应起来呢？这样对于命名的使用上来说彼此有了耦合

可以释放一部分，但我不建议让 `Free` 的参数和 `Start` 的参数一一对应，那样命令语义会变得拧巴：`Start 128 4` 到底对应 `Free 128 2`、`Free 256` 还是 `Free 2`？用户还得记住当初怎么申请的，确实形成了不必要耦合。

更好的做法是让释放命令只表达 **释放策略**，不复用申请参数。

推荐设计：

```text
Misc.MemoryLeak.Start [MB] [Count]
Misc.MemoryLeak.Free [Count]
```

语义：

```text
Misc.MemoryLeak.Start 128 4
```

申请 4 块，每块 128MB。

```text
Misc.MemoryLeak.Free 2
```

释放最近申请的 2 块。

```text
Misc.MemoryLeak.Free
```

释放全部。

也就是说内部把每次分配保存成一个 block 列表：

```cpp
struct FMemoryLeakBlock
{
    void* Ptr = nullptr;
    SIZE_T Size = 0;
    uint64 Id = 0;
};

TArray<FMemoryLeakBlock> Blocks;
```

释放时默认从数组尾部释放，也就是 LIFO：

```cpp
Free 1 -> 释放最后申请的一块
Free 2 -> 释放最后申请的两块
Free   -> 释放全部
```

这样 `Start` 和 `Free` 的参数没有强绑定：

- `Start` 关心“申请多大的块，申请几块”
- `Free` 关心“释放几个已存在的块”
- block 的大小可以不同，释放时不需要知道大小
- Memory Insights 上也很好看：释放最近几块会出现部分下降

如果你想更可控，可以再加一个状态命令：

```text
Misc.MemoryLeak.Status
```

输出当前 block 数量、总 MB、每个 block 的 id/size。但第一版我建议只做两个命令，`Free [Count]` 已经够用了。

我的推荐最终语义：

```text
Misc.MemoryLeak.Start [MB=64] [Count=1]
Misc.MemoryLeak.Free [Count=All]
```

第 5 个问题：你想采用这个 `Free [Count]` 的 LIFO 方案，还是坚持 `Free` 无参数、永远释放全部？

> 采用Free [Count]

<details><summary>1 previous message</summary>

> 很好，`Free [Count]` 用 LIFO，命令之间保持低耦合，也能做“部分下降”的 trace。现在我把设计收束一下，确认最后两个实现细节后就可以转入写 spec/计划。
</details>

第 6 个问题：`Start` 的参数上限要不要加保护？

我的推荐答案：**要加一个保守上限**，避免手滑把机器直接打爆。比如：

```text
Misc.MemoryLeak.Start [MB=64] [Count=1]
```

限制为：

```text
1 <= MB <= 1024
1 <= Count <= 128
单次命令总量 <= 4096 MB
```

也就是最大一次申请 4GB。超过就打印 warning，不执行。

这样仍然足够做 Memory Insights 采样：

```text
Misc.MemoryLeak.Start 64 10
Misc.MemoryLeak.Start 256 4
Misc.MemoryLeak.Free 3
Misc.MemoryLeak.Free
```

但不会因为输入错一个数就直接申请几十 GB。

你确认要加这个保护吗？

> 不需要加保护

<details><summary>1 previous message</summary>

> 明白，不加人为上限；我会保留基础参数校验，比如非正数直接拒绝，但不会限制 MB/Count 或总量。还有一个会影响 trace 可读性的细节：是否用 LLM tag 明确归因。
</details>

第 7 个问题：申请内存时是否固定加 LLM tag？

我的推荐答案：**加，而且固定为：**

```text
MiscInterface/MemoryLeak
```

也就是分配代码里包一层：

```cpp
LLM_SCOPE_BYNAME(TEXT("MiscInterface/MemoryLeak"));
```

这样 Memory Insights 里既能通过调用栈找到 `Misc.MemoryLeak.Start` 的实现，也能在 LLM Tag 树里直接看到这批内存归到 `MiscInterface/MemoryLeak`。

你确认使用这个 tag 名吗？

> 可以

我建议现在收束成这个设计。

可选方案有三个：

1. **只做两个命令，不做状态命令**
   推荐。最小、干净，完全服务 Memory Insights 实验。

2. **加 `Misc.MemoryLeak.Status`**
   方便查看当前还持有多少块，但超出你“两个指令”的目标。

3. **把 Start 做成持续申请模式**
   更像真实泄漏，但 trace 不如手动阶梯清楚，而且状态管理复杂。

推荐采用方案 1。

设计如下：

```text
Plugin: Engine/Plugins/Developer/MiscInterface
Module: MiscInterface
Type: Runtime
EnabledByDefault: false
```

控制台命令：

```text
Misc.MemoryLeak.Start [MB=64] [Count=1]
Misc.MemoryLeak.Free [Count=All]
```

行为：

```text
Misc.MemoryLeak.Start
```

申请 1 块 64MB。

```text
Misc.MemoryLeak.Start 128 4
```

申请 4 块，每块 128MB。

```text
Misc.MemoryLeak.Free 2
```

按 LIFO 释放最近申请的 2 块。

```text
Misc.MemoryLeak.Free
```

释放全部已申请块。

内部结构：

```cpp
struct FMemoryLeakBlock
{
    void* Ptr;
    SIZE_T Size;
    uint64 Id;
};
```

所有分配保存在模块内数组中。分配时使用：

```cpp
LLM_SCOPE_BYNAME(TEXT("MiscInterface/MemoryLeak"));
```

不加人为上限；只做基础参数校验，比如 `MB <= 0` 或 `Count <= 0` 时拒绝执行并打印日志。

确认一下：**这个设计可以作为最终设计进入实现吗？**

> 开始实现

<details><summary>51 previous messages</summary>

> 收到，开始实现。按现在的技能约束，我先读取实施相关的 TDD / 计划 / 验证说明，然后会直接落到代码里，不再继续追问。
>
> 我会把实现拆成一个很小的计划：先写自动化测试让它因为实现缺失而失败，再补 `MiscInterface` Runtime 插件和命令实现，最后构建验证。这里测试会尽量覆盖参数语义和 LIFO 释放，而不是真的在测试里申请很大的内存。
>
> 先看一下现有 Runtime 插件和自动化测试的组织方式，尤其是 Build.cs 里 automation test 的依赖写法。
>
> 先写 RED：我会只加插件描述、Build.cs 和一个自动化测试，测试引用尚未实现的内存泄漏服务；目标是先让构建因为缺少实现而失败，证明测试/接口需求已经落地。
>
> <details><summary>Created 3 files</summary>
>
> - Wrote `./Engine/Plugins/Developer/MiscInterface/MiscInterface.uplugin` (+20 -0)
> - Wrote `./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/MiscInterface.Build.cs` (+16 -0)
> - Wrote `./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/Tests/MiscMemoryLeakServiceTests.cpp` (+39 -0)
>
> </details>
>
> 现在运行一次针对 `MiscInterface` 模块的构建来确认 RED。预期是失败在缺少 `MiscMemoryLeakService.h` 或对应实现，而不是其他无关问题。
>
> 构建先被沙箱挡在 UBT 需要写 `%AppData%` 配置的位置了；这不是我们想看的 RED，所以我会按权限规则重新请求一次同样的构建。
>
> RED 达成：构建失败在 `MiscMemoryLeakService.h` 缺失，这是我们期望的“测试先于实现”状态。现在补最小实现：一个可测试的服务类、模块启动时注册两个 console command、模块关闭时释放残留内存。
>
> 我会先查一下当前 UE5.8 的 console command delegate 签名，避免凭记忆写错 API。
>
> 实现上我会把真实分配逻辑和控制台参数解析分开：`FMiscMemoryLeakService` 管块列表和释放，模块类只负责注册命令和打印反馈。这样测试不用通过 console 字符串绕一圈。
>
> <details><summary>Created 3 files</summary>
>
> - Wrote `./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscInterfaceModule.cpp` (+144 -0)
> - Wrote `./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscMemoryLeakService.cpp` (+76 -0)
> - Wrote `./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscMemoryLeakService.h` (+31 -0)
>
> </details>
>
> 实现已经补上了。现在跑同一个模块构建，先把编译错误清掉；如果这里过了，再考虑是否能启动自动化测试。
>
> 构建现在走到测试代码本身了：UE 这里的 ApplicationContext mask 是旧式宏名，不是 `EAutomationTestFlags::ApplicationContextMask`。我修这个测试拼写，再跑一次。
>
> <details><summary>Edited a file</summary>
>
> - Wrote `./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/Tests/MiscMemoryLeakServiceTests.cpp` (+1 -1)
>
> </details>
>
> 再跑同一条模块构建。这个循环有点朴素，但对 C++/UE 插件来说很有效：每次只处理当前最靠前的真实编译问题。
>
> 模块构建已经通过。为了真正执行自动化测试，我需要让禁用默认加载的插件在测试进程里被启用；我先确认 UE 命令行是否支持 `-EnablePlugin=...` 这类开关。
>
> 确认了插件可以用 `-EnablePlugins=MiscInterface` 临时启用。现在我尝试用 `UnrealEditor-Cmd.exe` 直接跑这条自动化测试；如果 UE 启动测试链路不配合，我会至少保留模块构建作为编译验证。
>
> 测试进程还没跑到自动化框架，启动阶段先被全平台 AutoSDK 校验拦住了，具体是 VisionOS SDK 缺失。我先找一下跳过/限制平台校验的开关，避免为一个 Win64 测试被其他平台拖住。
>
> 源码里找到了 `UE_SKIP_UBT_SDK_SETUP=1`，这是专门跳过 UBT SDK setup 的环境变量。我用它再跑一次自动化测试。
>
> 这次命令行没有把引擎日志吐回终端，只返回了 PowerShell profile 的启动噪声和 exit 1。我去读指定的 `MiscInterfaceAutomation.log` 看它实际跑到哪一步。
>
> 自动化测试启动进程现在被一个无关默认插件 `LocationServicesBPLibrary` 缺模块拦住了，还没跑到我们的测试。一个办法是临时用最小 `.uproject` 降低默认插件加载面；我先确认项目描述里是否支持禁用默认引擎插件。
>
> UE 项目描述支持 `DisableEnginePluginsByDefault`。我会创建一个临时最小测试 `.uproject`，只启用 `MiscInterface`，用它跑自动化测试，跑完再删掉这个临时文件。
>
> <details><summary>Created a file</summary>
>
> - Wrote `./Engine/Intermediate/MiscInterfaceTest/MiscInterfaceTest.uproject` (+13 -0)
>
> </details>
>
> 现在用这个临时 project 跑同一个测试；它应该只加载很小的插件集合，避开刚才那个无关默认插件问题。
>
> 终端还是只显示 PowerShell profile 噪声，我继续读日志判断实际失败点。
>
> 最小 project 避开了默认插件，但现在发现 `UnrealEditor` 启动时找不到 `MiscInterface` 模块。模块 DLL 在插件目录里，需要确认 `.modules` 元数据是否生成在插件 binaries 下，或者是否要做一次带 project/plugin 的构建。
>
> `-Module=MiscInterface` 生成了 DLL/PDB，但没有写插件的 `UnrealEditor.modules` 文件；启动器需要这个元数据。下一步用临时 project 做一次正常 Editor target 构建，让 UBT 生成完整插件二进制元数据。
>
> 项目构建通过并写了目标元数据。我确认一下插件模块元数据是否出现，然后再跑测试。
>
> 插件元数据已经生成。现在再跑自动化测试，这次应该能加载 `MiscInterface` 模块并执行测试。
>
> 测试进程还在跑，UE 启动自动化会比模块构建慢一些。我继续等它收尾，避免留下后台进程。
>
> 它还没退出，我查一下日志尾部看是测试在跑、卡在 shader/启动，还是 `-AutoQuit` 没触发。
>
> 日志显示它已经加载到 Editor 环境，但还在默认地图/资产初始化，可能 `-RunAutomationTests` 等待引擎完全起来。再给它一点时间，同时我会继续盯日志里的 Automation 结果。
>
> 它超过一分钟还没退出，像是 `-RunAutomationTests` 没触发或没自动退出。我先查日志里有没有 Automation 关键字；如果没有，我会终止这个测试进程，改用更可靠的 `-ExecCmds` 方式。
>
> 日志里已经确认 `MiscInterface` 模块加载了，但 `-RunAutomationTests` 触发方式不干净，还带出了别的自动化错误并且进程没退出。为了不留后台进程，我先终止这次测试进程，再调整验证方式。
>
> 自动化框架这条路在当前 Editor 环境里噪音太大。我改用更贴近用户场景的验证：启动最小 project，执行 `Misc.MemoryLeak.Start 1 2; Misc.MemoryLeak.Free 1; Misc.MemoryLeak.Free; Quit`，然后检查日志里命令反馈。
>
> 这个命令验证进程也还没退出，我查看日志确认 `ExecCmds` 是否已经执行。
>
> `ExecCmds` 把分号当成了命令参数的一部分，所以 `Start` 收到的是 `2;`，按预期拒绝了。先停掉这个进程，再用 UE 支持的逗号分隔方式重跑。
>
> 再跑一次命令验证，这次用逗号分隔多条 ExecCmds。
>
> 它还在启动阶段，我再查日志里的命令执行结果。
>
> 命令验证已经打到日志里：申请 2 块、释放 1 块、再释放全部都按预期执行了；进程正在响应 `Quit`，我等它完全退出。
>
> `Quit` 已执行但进程没有及时退干净，我看一下是否已经进入 shutdown；如果只是 Editor 命令行不退出，我会手动停掉它并把命令日志作为验证证据。
>
> 日志已经给出完整命令验证证据，但进程仍未自然退出；我把这个验证进程停掉，避免它继续占 1.6GB 内存。
>
> 验证用的临时 `.uproject` 该收拾掉；源码插件本身和构建产物我先保留，等看 git 状态再决定最终说明里怎么交代。
>
> <details><summary>Deleted a file</summary>
>
> - Deleted `./Engine/Intermediate/MiscInterfaceTest/MiscInterfaceTest.uproject` (+0 -13)
>
> </details>
>
> 我会看一下工作区状态，确认除了插件源码以外有没有验证时留下的临时文件需要清理。
>
> `git status` 只显示整个 `MiscInterface` 插件目录未跟踪，但里面包含刚才构建生成的 `Binaries/` 和 `Intermediate/`。我会只清理这两个生成目录，保留源码文件。
>
> 清理完生成目录后，我做一次最终模块构建验证；构建会再次生成二进制，我随后会再清掉这些产物，只留下源码改动。
>
> 最终构建通过了。现在我再次清理插件构建产物，避免把 DLL/PDB/Intermediate 混进源码改动。
>
> 最后做一个收尾检查：看源码 diff、未跟踪文件，以及确认没有测试进程还在跑。
>
> 收尾检查干净：没有 `UnrealEditor-Cmd` 残留进程，插件目录只剩源码文件；工作区里原本那个 `Engine/Config/DefaultEngine.ini` 仍是未跟踪，我没有碰它。
>
> 我再取一下关键文件的行号，方便最终给你精确入口。
</details>

已实现 `MiscInterface` 插件，位置在 [MiscInterface.uplugin](./Engine/Plugins/Developer/MiscInterface/MiscInterface.uplugin)。它默认不启用，模块类型是 Runtime。

核心行为：
`Misc.MemoryLeak.Start [MB=64] [Count=1]`
`Misc.MemoryLeak.Free [Count=All]`

实现入口在 [MiscInterfaceModule.cpp](./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscInterfaceModule.cpp:53)，分配/释放逻辑在 [MiscMemoryLeakService.cpp](./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscMemoryLeakService.cpp:17)。分配时加了：

```cpp
LLM_SCOPE_BYNAME(TEXT("MiscInterface/MemoryLeak"));
```

所以 Memory Insights 里可以按 `MiscInterface/MemoryLeak` 归因。

我也加了一个自动化测试源码，覆盖申请、LIFO 部分释放、全部释放的语义：[MiscMemoryLeakServiceTests.cpp](./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/Tests/MiscMemoryLeakServiceTests.cpp:9)。

验证做了两层：

```text
.\Engine\Build\BatchFiles\Build.bat UnrealEditor Win64 Development -Module=MiscInterface
```

结果：`Succeeded`。

还用最小临时 project 启动 Editor-Cmd 执行过：

```text
Misc.MemoryLeak.Start 1 2
Misc.MemoryLeak.Free 1
Misc.MemoryLeak.Free
```

日志确认分别得到 `2.00 MB -> 1.00 MB -> 0.00 MB`。不过 `UnrealEditor-Cmd` 执行 `Quit` 后没有自然退出，我手动停掉了验证进程。工作区里原本未跟踪的 `Engine/Config/DefaultEngine.ini` 我没有动。

> 1. 感觉你的实现过于复杂了。首先对于Module级别的cpp文件，我希望干净一些。因为后续可能会添加其他的功能。
> 2. 对于内存泄漏相关的代码，可以放在单独的文件中，没有问题。 其次对于指令可以单独定义一个文件，这个文件专门放一个command相关的
> 3. 为什么cpp文件以service结尾呢？ 感觉以MiscMemoryLeak命名就很好
> 4. 对于命令的参数解析也过于复杂了，没必要。如果不符合我们的预期就直接打日志即可

<details><summary>11 previous messages</summary>

> 我会按“收到 code review”来处理：先用 receiving-code-review 技能把反馈逐条落成具体改动，然后做一次纯重构，保持命令行为不变。你的方向是对的，模块入口确实应该瘦下来，后续扩功能才不会变成杂物间。
>
> 我把反馈理解成四个具体要求：模块 cpp 只做模块生命周期转发；内存泄漏逻辑独立文件；命令注册/参数处理独立文件；类/文件名从 `MiscMemoryLeakService` 改成更直接的 `MiscMemoryLeak`，并把参数解析降到最小。现在我先读当前实现，确认没有隐藏依赖。
>
> 准备改动前我会做三件事：把 `Service` 命名去掉；把 command 注册和参数处理搬到 `MiscMemoryLeakCommands.*`；把模块 cpp 缩到只注册/反注册这个 command 集合。参数解析会改成直接 `Atoi64/Atoi` 加正数判断，失败就打印 Usage，不再保留那套通用解析函数。
>
> <details><summary>Created 5 files, edited a file, deleted 3 files</summary>
>
> - Wrote `./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscInterfaceModule.cpp` (+4 -124)
> - Wrote `./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscMemoryLeak.cpp` (+74 -0)
> - Wrote `./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscMemoryLeak.h` (+28 -0)
> - Wrote `./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscMemoryLeakCommands.cpp` (+112 -0)
> - Wrote `./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscMemoryLeakCommands.h` (+24 -0)
> - Deleted `./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscMemoryLeakService.cpp` (+0 -76)
> - Deleted `./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscMemoryLeakService.h` (+0 -31)
> - Deleted `./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/Tests/MiscMemoryLeakServiceTests.cpp` (+0 -39)
> - Wrote `./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/Tests/MiscMemoryLeakTests.cpp` (+39 -0)
>
> </details>
>
> 重构已经落地：模块文件现在只负责生命周期转发，内存和命令都拆开了。接下来跑模块构建，确认文件重命名和新增拆分没有漏 include / UBT 规则问题。
>
> 编译只卡在一个 UE 类型前置声明细节：`IConsoleCommand` 在引擎里是 `struct`，我这里 forward declare 成了 `class`。修掉这个再跑。
>
> <details><summary>Edited a file</summary>
>
> - Wrote `./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscMemoryLeakCommands.h` (+1 -1)
>
> </details>
>
> 继续构建验证，看看是否还有第二层编译问题。
>
> 重构后的模块构建已经通过。现在我清理这次构建生成的插件 `Binaries/Intermediate`，只留下源码改动，再做状态检查。
>
> 最后检查 diff、状态和文件列表，确保没有旧 `Service` 文件残留，也没有生成物混进来。
>
> 检查结果符合预期：只有源码文件，`Service` 命名已经清掉，`diff --check` 没发现空白问题；原本的 `Engine/Config/DefaultEngine.ini` 仍然未跟踪且未触碰。
</details>

已按你的反馈重构了。

现在结构变成：

- [MiscInterfaceModule.cpp](./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscInterfaceModule.cpp)：只负责 `StartupModule/ShutdownModule`，调用命令集合注册/反注册。
- [MiscMemoryLeak.h](./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscMemoryLeak.h) / [MiscMemoryLeak.cpp](./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscMemoryLeak.cpp)：只放内存申请、LIFO 释放、全部释放逻辑。
- [MiscMemoryLeakCommands.h](./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscMemoryLeakCommands.h) / [MiscMemoryLeakCommands.cpp](./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscMemoryLeakCommands.cpp)：专门放 `Misc.MemoryLeak.Start` / `Misc.MemoryLeak.Free` 的注册和参数处理。
- 测试也改名为 [MiscMemoryLeakTests.cpp](./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/Tests/MiscMemoryLeakTests.cpp)。

参数解析已简化：直接取 `Atoi64/Atoi`，参数数量不对或值小于等于 0 就打印 Usage，不再保留通用解析函数。`Service` 命名也清掉了。

验证：

```text
.\Engine\Build\BatchFiles\Build.bat UnrealEditor Win64 Development -Module=MiscInterface
```

结果：`Succeeded`。构建生成的 `Binaries/` 和 `Intermediate/` 已清理，只保留源码文件。原本未跟踪的 `Engine/Config/DefaultEngine.ini` 没有改动。

> 1. Command 如果使用FAutoConsoleCommand的话，是不是就不需要主动调用Start函数去自己注册了
> 2. 将MiscMemoryLeakCommand 改为MiscCommands，因为后续有可能在这个文件中注册其他的Command
> 3.

<details><summary>7 previous messages</summary>

> 我会把这次反馈拆成两件事处理：先回答 `FAutoConsoleCommand` 的生命周期问题，再把命令文件泛化成 `MiscCommands`。同时我会继续保持模块入口干净，命令注册细节不回流到 Module cpp。
>
> `FAutoConsoleCommand` 可以省掉手写 `RegisterConsoleCommand/UnregisterConsoleObject`，但要注意它的生命周期：只要 `FAutoConsoleCommand` 对象活着，命令就注册；对象析构时会自动反注册。这里我会让 `FMiscCommands` 持有 `FAutoConsoleCommand` 成员，这样模块 cpp 只持有一个 `FMiscCommands`，不再显式调用注册函数。
>
> 我会采用 `FMiscCommands` 持有 `FAutoConsoleCommand` 成员的方式：构造 `FMiscCommands` 时命令自动注册，析构时自动反注册。模块文件只负责创建/销毁 `FMiscCommands` 对象，不再出现命令名、参数处理或手动注册代码。
>
> <details><summary>Created 2 files, edited a file, deleted 2 files</summary>
>
> - Wrote `./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscCommands.cpp` (+87 -0)
> - Wrote `./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscCommands.h` (+23 -0)
> - Wrote `./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscInterfaceModule.cpp` (+4 -4)
> - Deleted `./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscMemoryLeakCommands.cpp` (+0 -112)
> - Deleted `./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscMemoryLeakCommands.h` (+0 -24)
>
> </details>
>
> 改动已经完成：`MiscMemoryLeakCommands` 文件被替换成泛化的 `MiscCommands`，手动 Register/Unregister 消失了。现在跑模块构建验证 `FAutoConsoleCommand` 成员初始化和模块生命周期没问题。
>
> 构建通过。按之前的习惯，我清理这次构建生成的插件 `Binaries/Intermediate`，再检查状态和旧文件名残留。
>
> 现在做最终检查。
</details>

已调整。

关于第 1 点：是的，改成 `FAutoConsoleCommand` 后就不需要自己调用 `RegisterConsoleCommand/UnregisterConsoleObject` 了。现在 `FMiscCommands` 构造时自动注册命令，析构时自动反注册。

改动后结构：

- [MiscInterfaceModule.cpp](./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscInterfaceModule.cpp)：只创建/销毁 `FMiscCommands`，不出现具体 command 逻辑。
- [MiscCommands.h](./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscCommands.h) / [MiscCommands.cpp](./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscCommands.cpp)：统一放命令注册和处理，后续可以继续加其他 Misc 命令。
- [MiscMemoryLeak.h](./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscMemoryLeak.h) / [MiscMemoryLeak.cpp](./Engine/Plugins/Developer/MiscInterface/Source/MiscInterface/Private/MiscMemoryLeak.cpp)：只保留内存泄漏实验逻辑。

验证：

```text
.\Engine\Build\BatchFiles\Build.bat UnrealEditor Win64 Development -Module=MiscInterface
```

结果：`Succeeded`。构建产物已清理，只保留源码文件。`Engine/Config/DefaultEngine.ini` 仍是原本未跟踪文件，我没有改。

> [$ai-conversation-archive](C:/Users/yongkang.guo//.codex/skills/ai-conversation-archive/SKILL.md)

<details><summary>1 previous message</summary>

> System error
>
> unexpected status 401 Unauthorized: Missing bearer or basic authentication in header, url: https://api.openai.com/v1/responses, cf-ray: a4b4bbfccc9acf26-SJC, request id: req_f8b68d799df2404a99f7a4639300c984
</details>

> [$ai-conversation-archive](C:/Users/yongkang.guo//.codex/skills/ai-conversation-archive/SKILL.md)&#x20;
> 归档

<details><summary>1 previous message</summary>

> System error
>
> unexpected status 401 Unauthorized: Missing bearer or basic authentication in header, url: https://api.openai.com/v1/responses, cf-ray: a4b4bd1a8946b917-SJC, request id: req_ddf740790a3f4cfc96ca11a4967a3ff2
</details>
