# 搜索UE UI底层优化策略

> 请你联网搜索，关于UE4或UE5，UI界面、UMG、Slate的优化策略有哪些？特别是涉及到底层方面的优化机制。

<details><summary>4 previous messages</summary>

> 我会先用 `using-superpowers` 规范这次调研流程，然后联网查阅 Epic 官方文档、Unreal Insights/源码相关资料和高质量技术文章，重点拆解 UMG/Slate 底层的失效、缓存、绘制批次、Tick 与纹理更新机制。
>
> <details><summary>Searched the web</summary>
>
> Searched the web for `site:dev.epicgames.com/documentation/en-us/unreal-engine UMG optimization invalidation Retainer Box Global Invalidation Slate ...`
>
> </details>
>
> 初步资料显示，真正决定 UI 成本的不是“Widget 数量”这一项，而是每帧触发了多少次树遍历、布局重算、绘制元素重建与批次切换。接下来我会补齐 UE4.27 与 UE5 的版本差异，以及列表虚拟化、纹理/字体图集、裁剪与离屏 RT 的 GPU 侧机制。
>
> <details><summary>Searched the web</summary>
>
> Searched the web for `site:dev.epicgames.com/documentation/en-us/unreal-engine listview tileview virtualized UMG performance ...`
>
> Searched the web for `site:dev.epicgames.com/community/learning/tutorials UMG Slate optimization invalidation Epic Games ...`
>
> Searched the web for `site:dev.epicgames.com/documentation/en-us/unreal-engine Slate rendering thread FSlateApplication DrawWindows FSlateRenderer ...`
>
> Searched the web for `site:dev.epicgames.com/documentation/en-us/unreal-engine "Display Texture Atlases" Slate ...`
>
> Searched the web for `site:dev.epicgames.com/documentation/en-us/unreal-engine?application_version=4.27 UMG optimization invalidation box retainer box ...`
>
> Searched the web for `site:dev.epicgames.com/documentation/en-us/unreal-engine UMG background blur performance low quality fallback ...`
>
> Searched the web for `site:dev.epicgames.com/documentation/en-us/unreal-engine EVisibility HitTestInvisible SelfHitTestInvisible Slate performance ...`
>
> Searched the web for `site:dev.epicgames.com/documentation/en-us/unreal-engine UMG Viewmodel FieldNotify performance binding MVVM ...`
>
> </details>
</details>

调研结论：UE 的 UI 优化核心不是单纯“减少 Widget 数量”，而是同时控制四类成本：

1. Game Thread 上的 Tick、Binding、Prepass、布局和 Widget 树遍历。
2. Paint 阶段生成 `FSlateDrawElement` 的成本。
3. Render Thread 上的 Element Batching、顶点/索引构建和 Draw Call。
4. GPU 上的透明混合、UI Material、模糊、Retainer RenderTarget 和 Overdraw。

投入产出比通常是：

> 事件驱动更新 → Invalidation → 列表虚拟化/对象复用 → 减少 Layer/批次 → 谨慎使用 Retainer → 控制 GPU Overdraw

## 一、Slate 底层一帧是怎么工作的

UMG 的 `UWidget/UUserWidget` 最终会构造对应的 Slate `SWidget`。每次 Slate 更新大致经历：

```text
数据/Tick/属性绑定
    ↓
Prepass：自底向上 ComputeDesiredSize
    ↓
Arrange：父节点分配子节点 Geometry
    ↓
Paint：递归 OnPaint，生成 FSlateDrawElement
    ↓
FSlateElementBatcher：按 Layer 和渲染状态排序、合批
    ↓
FSlateRenderBatch → Vertex/Index Buffer → RHI Draw Call
```

Slate 布局本身是两阶段的：先缓存 Desired Size，再排列子节点；Paint 会遍历可见 Widget 并产生绘制元素。[Slate Architecture](https://dev.epicgames.com/documentation/unreal-engine/understanding-the-slate-ui-architecture-in-unreal-engine)、[FSlateApplication::TickAndDrawWidgets](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/Slate/Framework/Application/FSlateApplication/TickAndDrawWidgets)

因此，一个看起来静止的复杂界面如果没有使用失效缓存，仍可能不断执行属性读取、布局遍历、Paint 和批次构建。

## 二、最关键的 CPU 优化：Invalidation

### 1. Invalidation 的底层机制

Invalidation Root 会保存 Widget 的层级、布局信息和缓存绘制元素。当属性变化时，Widget 被加入 Dirty List，下一帧只处理脏节点及必要的父子链，而不是重建整个窗口。

常见失效类型的相对成本：

| 失效类型 | 典型原因 | 影响 |
|---|---|---|
| Paint | 颜色、Brush 等不改变尺寸的外观 | 重绘当前节点，较低 |
| RenderTransform | 位移、缩放、旋转 | Geometry/绘制信息更新 |
| Layout | Desired Size、Padding、尺寸改变 | 向父节点传播并重新布局 |
| Visibility | Visible/Hidden/Collapsed 改变 | 通常隐含布局更新 |
| ChildOrder | 添加、删除、重排子节点 | 可能重建失效代理列表，最高 |
| Volatility | 动态节点绕过绘制缓存 | 每帧重绘指定子树 |

Epic 当前文档也明确区分了 Paint、Layout、Child invalidation，并指出频繁增删子节点成本最高。[UI Invalidation](https://dev.epicgames.com/documentation/unreal-engine/invalidation-in-slate-and-umg-for-unreal-engine)

底层优化原则是：

- 自定义 Slate 控件只请求真正需要的失效类型。
- 只改颜色时不要调用 Layout 失效。
- Setter 中先比较新旧值，相同就不更新、不失效。
- 避免每帧 `AddChild/RemoveChild/ClearChildren`。
- 高频变化节点标记为 Volatile，但只标记最小叶子子树，不能把整个 HUD 根节点标为 Volatile。

### 2. Invalidation Box、Global Invalidation、Retainer 的区别

| 机制 | 主要减少什么 | 代价与限制 |
|---|---|---|
| Invalidation Box | Prepass、Paint、Draw Element 重建等 CPU 成本 | 占用缓存内存；频繁失效时收益消失 |
| Global Invalidation | 让整个 `SWindow` 成为 Invalidation Root | 开启后普通 Invalidation Box 会被停用 |
| Retainer Box | 降低重绘频率，并把子树最终压成一个纹理 Quad | 额外 RT、显存、离屏绘制和更新尖峰 |

Global Invalidation 可通过 `Slate.EnableGlobalInvalidation 1` 测试；它不是简单地把所有 Invalidation Box 叠加起来，而是由 `SWindow` 统一管理。[UI Invalidation](https://dev.epicgames.com/documentation/unreal-engine/invalidation-in-slate-and-umg-for-unreal-engine)

对于当前这个 UE4.27 代码库，Global Invalidation/Fast Path 已经存在，但默认关闭。UE4.27 项目更应该先把局部 Invalidation Box 用正确，再做全局模式的 A/B Test。UE4.27 文档还特别提醒：某些属性需要主动调用 `Invalidate Layout And Volatility`，不能假设所有自定义控件都会自动失效。[UE4.27 Invalidation Box](https://dev.epicgames.com/documentation/unreal-engine/invalidation-box?application_version=4.27)

### 3. Retainer 什么时候才值得用

适合：

- 大块 HUD 基本静止，只需 15/30Hz 更新。
- 低端移动平台受 Draw Call 限制。
- 需要对整组 UI 施加统一材质效果。
- 多个 Retainer 可以用不同 Phase 错开刷新，削平单帧尖峰。

不适合：

- 每帧都变化的血条、滚动列表、输入框。
- 大面积高分辨率全屏界面频繁刷新。
- 仅仅为了“听说 Retainer 能优化”而包住所有界面。

Retainer 更新帧本身仍要完整绘制子树到 RT，然后再绘制 RT；一个 1920×1080 BGRA8 RT 单张约 7.9 MiB，实际还可能受对齐、缓冲和平台格式影响。Epic 也建议先使用 Invalidation，只有仍需压缩 Draw Call 时再上 Retainer。[Retainer Box API](https://dev.epicgames.com/documentation/unreal-engine/API/Runtime/UMG/URetainerBox?lang=en-US)

## 三、数据更新与 Tick

### 高收益措施

- 不要使用 UMG 原始 Property/Function Binding 更新血量、弹药、任务文本。
- 不要在每个子 Widget 的 Tick 中查询 Pawn、PlayerState 或 Subsystem。
- 由 Gameplay 数据变化事件主动调用 UI：
  - `OnHealthChanged`
  - `OnAmmoChanged`
  - `OnInventoryChanged`
  - `OnQuestStateChanged`
- 可以按一帧聚合多个变化，帧末只刷新 UI 一次。
- 对倒计时、网络延迟等允许低频的数据，使用 5–10Hz 定时更新，不必 60/120Hz。

Epic 当前优化指南明确指出原始 Binding 会每帧轮询，应改为 Event Dispatcher/Delegate。[UMG Optimization Guidelines](https://dev.epicgames.com/documentation/en-us/unreal-engine/optimization-guidelines-for-umg-in-unreal-engine)

UE5 可以考虑 MVVM 的 `FieldNotify`。它只在字段真正变化时推送更新，属于事件驱动绑定；但当前官方仍将该功能标为 Beta，投产前需结合具体 UE5 小版本评估。[UMG Viewmodel](https://dev.epicgames.com/documentation/en-us/unreal-engine/umg-viewmodel-for-unreal-engine)

## 四、布局、层级与 Draw Call

### 1. 少嵌套 Canvas Panel 和 Overlay

Slate 合批首先受 `LayerId` 限制。Canvas Panel 为了支持任意 ZOrder，通常会增加子节点 Layer；Overlay 也会增加 Layer。不同 Layer 之间不能合批。

建议：

- HUD/菜单根部保留一个 Canvas 可以接受。
- 按钮、列表项、复合控件内部不要再套 Canvas。
- 优先使用 Horizontal/Vertical/Grid/UniformGrid。
- 单元素控件不要额外套 Canvas。
- 避免为了 Padding 连续嵌套 Border、SizeBox、Overlay。

Epic 明确指出 Canvas/Overlay 的 Layer 行为会导致更多 Draw Call。[UMG Optimization Guidelines](https://dev.epicgames.com/documentation/en-us/unreal-engine/optimization-guidelines-for-umg-in-unreal-engine)

### 2. 理解 Slate 的合批条件

Slate 的一个 Render Batch 除了 Layer 相同，还要求以下状态兼容：

- `ShaderResource`，即纹理、字体 Atlas 页或材质资源。
- Shader 类型和参数。
- Blend/Draw Effect。
- Primitive 类型。
- Clipping State。
- Custom Drawer、Instance Data 等。

官方 API 特别说明：一个 `FSlateElementBatch` 不能包含多个纹理资源。[FSlateElementBatch](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/SlateCore/FSlateElementBatch)

因此应当：

- 让相邻元素尽量共享纹理或图集页。
- 复用同一个 Material/Material Instance，避免每个节点独立创建 MID。
- 减少交错的纹理顺序，例如 A/B/A/B。
- 减少不必要的 ZOrder。
- 用 Widget Reflector 的 Texture/Font Atlas Visualizer 验证实际是否进入同一 Atlas，而不是只凭资源命名判断。

### 3. 谨慎使用 Clipping

不同裁剪区域不能跨区合批。轴对齐裁剪通常使用 Scissor，任意旋转/复杂裁剪可能使用 Stencil，后者更昂贵。

建议只让最外层 ScrollBox/容器负责裁剪，不要给每个列表项和内部 Image 都开启 Clip to Bounds。[Slate Clipping System](https://dev.epicgames.com/documentation/unreal-engine/using-the-slate-clipping-system-in-unreal-engine)

## 五、长列表、创建销毁与内存

### 1. 使用虚拟化列表

物品栏、邮件、排行榜等不要用：

```text
ScrollBox
  ├─ Entry 1
  ├─ Entry 2
  ├─ ...
  └─ Entry 2000
```

使用 `ListView/TileView/TreeView`。它们只创建屏幕可见范围内的 Entry。例如 200 个 Item、屏幕只能显示 5 个时，只需要约 5 个 Entry Widget。[UListView](https://dev.epicgames.com/documentation/unreal-engine/API/Runtime/UMG/UListView?lang=en-US)

Entry 会被复用，因此要在 `OnListItemObjectSet` 中完整重置文本、Brush、选择态和 Delegate，避免残留状态。

### 2. 按使用频率决定缓存还是销毁

- 高频打开：背包、计分板、快捷菜单——提前创建并复用。
- 低频大型页面：异步加载，关闭后释放。
- 伤害数字、提示条、聊天气泡——对象池。
- 不要频繁 `CreateWidget → AddViewport → Remove → GC`。

所有 Widget Blueprint 子节点即使当前不显示，也会参与构造并占内存；大型界面应拆分为按需加载模块。[UMG Optimization Guidelines](https://dev.epicgames.com/documentation/en-us/unreal-engine/optimization-guidelines-for-umg-in-unreal-engine)

## 六、动画、文本与 GPU

### 动画

成本从低到高通常是：

1. 纯 Material 参数/UV 动画：无 UI Game Thread 动画成本，但仍有 GPU 像素成本。
2. 简单 Blueprint Tween。
3. UMG Sequencer 动画：存在初始化和属性路径解析成本。
4. 每帧改变尺寸、Padding、Wrap、子节点结构：最贵。

短而高频的伤害数字动画更适合轻量 Tween；几何动画应隔离为小型 Volatile 子树。Epic 的动画成本表也把导致 Layout 改变的动画列为最昂贵类型。[UMG Animation Costs](https://dev.epicgames.com/documentation/en-us/unreal-engine/optimization-guidelines-for-umg-in-unreal-engine)

### 文本

- RichText 比普通 TextBlock 昂贵。
- 避免每帧 `FText::AsNumber`、格式化、本地化和自动换行重算。
- 数字变化时才重新格式化。
- 尽量统一字体、字号和 Font Material，减少字体 Atlas 页及批次切换。
- 字体 `Stream` 节省内存但首次遇到字形可能产生 I/O；关键 UI 可考虑 Lazy Load/Inline，按平台权衡。[Font Asset and Editor](https://dev.epicgames.com/documentation/en-us/unreal-engine/font-asset-and-editor-in-unreal-engine)

### GPU 与 Overdraw

- UI 大多使用透明混合，完全透明的纹理空白区域仍可能产生像素处理。
- 裁掉 UI 图片周围的大面积透明 Padding。
- 避免叠加多层全屏半透明 Image。
- 普通美术优先 Texture，Material 仅用于确实需要的动态效果。
- UI Material 保持低指令、低采样数，避免复杂噪声、循环和多层透明。
- Background Blur 是额外后处理；低端平台使用低质量替代 Brush。
- 3D Widget Component 本质上还要更新 RT：降低 Draw Size，设置 Redraw Time，静态内容启用 Manually Redraw。

UE4.27 官方同样建议普通 UI 优先 Texture、事件驱动更新和 Invalidation Box。[UE4.27 UMG Best Practices](https://dev.epicgames.com/documentation/unreal-engine/umg-best-practices?application_version=4.27)

## 七、极限场景下的 Slate/C++ 优化

当一个界面有数百个简单 Image/Text Widget 时，可以考虑把它们折叠成一个自定义 Slate Widget：

- 继承 `SLeafWidget` 或轻量 `SWidget`。
- 在一次 `OnPaint` 中生成多个 Quad。
- 更进一步使用 `MakeCustomVerts`/自定义 Mesh Widget 批量提交顶点。
- 自己实现 `ComputeDesiredSize`。
- 属性变化时精确调用 `Invalidate(Paint)` 或 `Invalidate(Layout)`。
- 不为每个装饰元素创建一个 `UObject + SWidget + HitTest Entry`。

这种方案能显著减少 Widget 树、虚函数遍历、内存分配、HitTest 和 Draw Element 数量，但可维护性和设计师工作流会变差，通常只适合小地图图标、海量标记、复杂曲线图、技能节点图等明确热点。

## 八、推荐的性能排查流程

1. 在目标设备的 Standalone/Development 包中测试，不以编辑器帧率作为结论。
2. 使用：
   - `stat slate`
   - `stat slatememory`
   - `stat unit`
   - `stat gpu`
3. 关注：
   - `SlatePrepass`
   - `Tick Widgets`
   - `UMG Binding`
   - Painted/Ticked Widget 数量
   - Draw Elements、Recached Elements
   - Render Batches、Slate GPU Pass
4. 开启 Slate Insights，启动参数使用 `-trace=slate`，查看每帧的：
   - `L/P/U/C/R/V` 失效原因。
   - `Tick/ActiveTimer/Repaint/VolatilePaint` 更新原因。[Slate Insights](https://dev.epicgames.com/documentation/en-us/unreal-engine/slate-insights-in-unreal-engine)
5. 使用 Widget Reflector：
   - Pick Painted Widgets。
   - Pick Hit-Testable Widgets。
   - Invalidation、Update、Paint 调试。
   - Texture/Font Atlas。
   - Clipping 和 Culling。
6. 对同一场景做 A/B Test：
   - Global Invalidation 0/1。
   - Retainer 开/关。
   - Background Blur 正常/低质量替代。
   - 传统 ScrollBox/虚拟化 ListView。
   - 原始 Binding/事件驱动。

如果只能先做五件事，我建议依次是：删除原始 Binding 和无意义 Tick、把长列表换成 ListView、正确划分静态与动态 Invalidation 子树、减少嵌套 Canvas/裁剪/材质资源、最后再根据测量结果引入 Retainer。

> 有没有通过修改底层的优化方式呢？或者在引擎层面添加机制，而不是客户端资源层面的修改。请你再联网搜索下

<details><summary>4 previous messages</summary>

> 这次我会把范围收紧到“可在引擎源码中实现的机制”：失效系统、更新调度、批处理、并行化、列表/Widget 池、字体缓存、Retainer/RT 复用，以及可观测性。重点区分哪些适合回移植到 UE4.27，哪些只适合借鉴 UE5。
>
> <details><summary>Searched the web</summary>
>
> Searched the web for `site:dev.epicgames.com/documentation/en-us/unreal-engine release notes Slate invalidation performance optimization parallel rendering UMG ...`
>
> Searched the web for `site:dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-5 release notes "Slate" "performance" "invalidation" ...`
>
> Searched the web for `site:dev.epicgames.com/documentation/en-us/unreal-engine API TSlateAttribute invalidation reason performance ...`
>
> Searched the web for `site:dev.epicgames.com/documentation/en-us/unreal-engine SlateIM immediate mode performance UI ...`
>
> Searched the web for `site:youtube.com/@UnrealEngine UMG Slate optimization performance ...`
>
> </details>
>
> 从 UE5 的演进路线看，Epic 自己也在持续做引擎级改造：更细粒度的 Invalidation Root/WidgetProxy 队列、缓存 RenderBatch、Widget Pool、属性变更通知，以及把部分 Slate 渲染迁到异步 RDG。现在我会对照你这份 UE4.27 源码，筛出可直接启用、适合回移植和需要重构的三类方案。
>
> Context compaction
>
> Source: automatic
> Status: completed
</details>

有，而且 UE5 后续版本本身就在持续进行这类“引擎层优化”。我对照了 Epic 官方资料和你当前 UE4.27 的源码：不少基础设施已经存在，真正值得改的是失效粒度、调度、缓存层级、RT 复用和渲染阶段并行，而不是重新造一套 Slate。

## 适合 UE4.27 的引擎级方案

| 机制 | 主要解决 | 收益方向 | 实现风险 |
|---|---|---:|---:|
| Global Invalidation 完善 | 重复 Prepass/Paint/树遍历 | Game Thread | 低～中 |
| 回移植 `TSlateAttribute` | Binding 每帧轮询 | Game Thread | 中 |
| Widget 更新分级调度 | Tick、Timer、Prepass 集中尖峰 | Game Thread | 中 |
| 全局 Widget Pool | 创建、销毁、GC 尖峰 | GT/内存 | 低 |
| Retainer RT 池与预算调度 | RT 内存和同帧更新尖峰 | GPU/RT | 中 |
| Cook 阶段 UI Atlas | Batch 被纹理切碎 | Render Thread/GPU | 中 |
| 持久化 GPU 几何缓存 | 每帧重建 VB/IB | Render Thread | 高 |
| Slate 渲染后半段并行化 | Batch/RDG 提交开销 | Render Thread | 高 |
| WidgetComponent 共享 RT Atlas | 大量世界空间 UI | GPU/显存 | 中～高 |
| 增量 HitTestGrid | 大型交互树重建 | Game Thread | 中 |

---

## 1. 把 Global Invalidation 做成真正可靠的默认路径

你的 UE4.27 已经有 Global Invalidation，只是默认关闭：

- [SlateCoreClasses.cpp](./Engine/Source/Runtime/SlateCore/Private/SlateCoreClasses.cpp:46)
- [SlateInvalidationRoot.h](./Engine/Source/Runtime/SlateCore/Public/FastUpdate/SlateInvalidationRoot.h:63)

可以先测试：

```ini
[ConsoleVariables]
Slate.EnableGlobalInvalidation=1
```

它把 `SWindow` 作为整个窗口的 Invalidation Root，缓存 Widget 树、布局和绘制结果，只更新脏 Widget。官方当前架构也是这个方向；启用 Global Invalidation 后，嵌套 Invalidation Box 通常不再发挥作用。[Epic：UI Invalidation](https://dev.epicgames.com/documentation/unreal-engine/invalidation-in-slate-and-umg-for-unreal-engine)

但 UE4.27 的关键问题通常不是“没有 Fast Path”，而是：

- 某个属性修改后没有调用正确的 `Invalidate`。
- 本应是 `Paint` 的变化被升级成 `Layout` 或 `ChildOrder`。
- 某个 Widget 导致 Root 回退到 Slow Path。
- 动画每帧造成大范围脏传播。

建议在引擎中添加一套 Invalidation Correctness/Diagnostics：

- 记录每次失效的 Widget、原因、传播深度。
- 统计每帧进入 Slow Path 的 Root 和触发者。
- 非 Shipping 下周期性使用 Slow Path 重绘，与缓存结果进行校验。
- 对 Setter 自动比较新旧值，没有变化就不失效。
- 区分 `Paint`、`Layout`、`Visibility`、`ChildOrder`，禁止随意使用最高级别失效。

UE5 当前 `FSlateInvalidationRoot` 已经把待更新对象进一步分为 PreUpdate、PrepassUpdate、PostUpdate 等队列，而 UE4.27 基本还是一个 `WidgetsNeedingUpdate` 堆。这部分很适合回移植。[Epic：FSlateInvalidationRoot](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/SlateCore/FSlateInvalidationRoot)

---

## 2. 回移植 UE5 的 `TSlateAttribute`：从轮询改为精确失效

这是我认为 UE4.27 最值得做的一项底层改造。

UE4.27 缺少 UE5 的 `Types/SlateAttribute.h` 体系。传统 UMG Binding 往往每帧执行 Getter，即使返回值完全没有变化，也会产生：

- Blueprint VM 调用。
- UObject 属性读取。
- 文本转换和格式化。
- 不必要的 Paint/Layout 判断。
- 大量分散的小函数调用。

UE5 的注册式 Slate Attribute 可以把属性变化映射成准确的失效原因，例如：

```cpp
Text变化       -> Paint
Color变化      -> Paint
Padding变化    -> Layout
Visibility变化 -> Layout / Visibility
Children变化   -> ChildOrder
```

官方也明确指出旧式 UMG Attribute Binding 会持续更新，而事件驱动的 Field Notify/Viewmodel 方式更高效。[Epic：UE5.1 Release Notes](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-5.1-release-notes?application_version=5.1)、[TSlateAttributeInvalidationReason](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/SlateCore/TSlateAttributeInvalidationReaso-)

回移植时可分两层：

1. Slate 层实现注册属性、Owner、Getter、缓存值和失效 Predicate。
2. Widget Blueprint 编译器把 Binding 转成 Field Notify 订阅；只有依赖字段改变时才重新求值。

这样大部分客户端 UI 无须手动修改 Tick，只需逐步让数据模型支持变更通知。

---

## 3. 重构 Widget 更新调度器

UE4.27 已经有统一的动画管理器：

- [UMGSequenceTickManager.h](./Engine/Source/Runtime/UMG/Public/Animation/UMGSequenceTickManager.h:19)

因此不必再做一个动画 Manager，应该把范围扩展到通用 Widget 更新调度：

```cpp
enum class EWidgetUpdatePolicy
{
    Never,
    WhenVisible,
    OnDemand,
    EveryFrame,
    Interval
};
```

引擎集中管理：

- Widget Tick。
- Active Timer。
- Blueprint Binding 更新。
- 动画求值。
- Prepass 请求。
- Retainer 刷新。
- 延迟布局请求。

可以增加频率桶，例如 1/2/4/8/16 帧更新，避免每个 Widget 独立做时间判断。隐藏、折叠、脱离窗口或完全被遮挡的 Widget 自动降频或暂停。

UE5.8 最近也在优化 Slate Timer，通过合并遍历减少 Timer Update Path 开销，说明集中调度仍然是 Epic 的优化方向。[Epic：UE5.8 Release Notes](https://dev.epicgames.com/documentation/unreal-engine/unreal-engine-5-8-release-notes)

---

## 4. 在现有 `FUserWidgetPool` 上建立引擎级全局池

UE4.27 已经有：

- [UserWidgetPool.h](./Engine/Source/Runtime/UMG/Public/Blueprint/UserWidgetPool.h:24)

ListView、DynamicEntryBox 也已经使用它。因此不需要重新实现池，而应该建立一个 `UWidgetPoolSubsystem`，让伤害数字、Toast、浮动提示、任务条目等统一复用。

池键可以是：

```text
WidgetClass + OwningPlayer + World + SlateStyleVariant
```

底层需要补充：

- `Acquire/Release` 生命周期协议。
- 回收时解除 Delegate、动画和 Active Timer。
- 最大空闲数和内存压力裁剪。
- World/Player 销毁时清池。
- 正确释放 Slate Resources，避免 UObject/SWidget 循环引用。

官方 `FUserWidgetPool` 文档也特别提醒要释放 Slate 资源以避免循环引用。[Epic：FUserWidgetPool](https://dev.epicgames.com/documentation/en-us/unreal-engine/API/Runtime/UMG/FUserWidgetPool)

它主要改善创建尖峰、GC 和内存抖动，不一定显著降低稳定状态下的 Paint 时间。

---

## 5. Retainer Box 改为“RT 池 + 全局刷新预算”

UE4.27 的每个 `SRetainerWidget` 基本上各自持有 `UTextureRenderTarget2D`：

- [SRetainerWidget.cpp](./Engine/Source/Runtime/UMG/Private/Slate/SRetainerWidget.cpp:169)

因此大量 Retainer 会造成：

- 每个控件一张 RT。
- 分辨率变化时重新分配。
- 多个 Retainer 同帧刷新导致 GPU 峰值。
- 页面隐藏后 RT 仍可能占用显存。

可以添加两个引擎服务。

### `FSlateRenderTargetPool`

按以下键复用 RT：

```text
QuantizedWidth + QuantizedHeight + PixelFormat + Gamma
```

尺寸做 64/128 像素量化，隐藏页面把 RT 归还池，而不是销毁；发生内存压力时统一裁剪。

### `FSlateRetainerScheduler`

每帧限制：

- 最大刷新 Retainer 数。
- 最大更新像素数。
- 最大预计 GPU 时间。
- 高优先级交互 UI 优先。
- 脏但不可见的 Retainer 延迟。
- 长时间静态的 Retainer 自动降低刷新率。

Retainer 本身官方也说明存在独立 RT 内存和重绘成本。[Epic：UI Invalidation](https://dev.epicgames.com/documentation/unreal-engine/invalidation-in-slate-and-umg-for-unreal-engine)

---

## 6. 从 Batch 底层解决 Draw Call：Cook Atlas 或多纹理 Batch

Slate 的 Batch 是否能合并，受以下状态限制：

- Shader Resource/纹理。
- Layer。
- Clipping State。
- Shader Type。
- Draw Effects。
- Primitive Type。
- Instance Data。
- Custom Drawer。

其中纹理是最常见的切批原因。你当前 UE4.27 的 `FSlateElementBatch`/`FSlateRenderBatch` 基本要求同一 Batch 使用同一个 Shader Resource。

### 相对稳妥：Cook 阶段自动 Atlas

增加 UI Cook Commandlet：

1. 收集所有 Slate Brush/UMG Texture。
2. 按格式、Filter、sRGB、压缩设置分组。
3. 打成 Atlas Page。
4. Cook 时重写 Brush 的资源句柄和 UV。
5. 运行时对业务完全透明。

同时在 `FSlateBatchData::MergeRenderBatches` 中记录“为什么没有合批”，例如：

```text
DifferentTexture
DifferentClip
DifferentLayer
DifferentShader
DifferentEffect
```

这样能精准判断 Atlas 是否值得做。UE5.4 曾把 Slate Texture Atlas 查找路径优化约 4 倍，也说明 Atlas/资源查找依旧是引擎热点。[Epic：UE5.4 Release Notes](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-5.4-release-notes?application_version=5.4)

### 激进方案：Texture Array / Bindless Slate

在 Slate Vertex 中加入 Texture Index，一个 Batch 绑定 Texture Array 或 Descriptor Array，从而允许同 Layer、同 Clip、不同纹理的元素合并。

这个方案潜力大，但风险也高：

- 不同尺寸和格式的纹理管理。
- RHI 平台能力差异。
- UE4.27 移动端兼容性。
- Slate 材质和普通 Brush 的统一。
- Alpha 顺序不能任意重排。

因此建议先做 Batch Split 统计和自动 Atlas，再考虑 Bindless。

---

## 7. 把缓存推进到 GPU：持久化 Vertex/Index Buffer

UE4.27 已经有 Cached Elements 和 `CachedBatches`，但缓存主要还是 Slate 绘制元素和 Batch 数据，最终 GPU Buffer 构造仍可能发生。

更深一层的方案是：

```text
Invalidation Root
 ├─ Persistent Vertex Buffer
 ├─ Persistent Index Buffer
 ├─ Batch Range Table
 └─ Dirty Range List
```

Widget 没有失效时直接复用 GPU Buffer 区段；只有发生 Paint/Layout 变化的节点才上传 Dirty Range。`ChildOrder` 改变时再重排或压缩 Buffer。

适合：

- 大型常驻 HUD。
- 大量静态图标和文本。
- 主机/PC。
- UI 基本不改变尺寸的项目。

难点是裁剪、文字 Atlas 更新、层级变化、Buffer 碎片整理和资源生命周期，所以属于高风险方案。

---

## 8. Slate 并行化：只并行“不可变快照之后”的阶段

不建议直接把 `SWidget::OnPaint` 或 Widget 树遍历扔到 Worker Thread，因为其中可能调用：

- UMG UObject。
- Blueprint。
- Attribute Getter。
- Delegate。
- 动态材质。
- 非线程安全的字体和资源状态。

安全的边界应该是：

```text
Game Thread
Widget Tick / Attribute / Prepass / OnPaint
                 ↓
        不可变 DrawElement 快照
                 ↓
Worker Threads
Element Expand / Batch Build / Clip Resolve
                 ↓
Render Thread / RHI
RDG Pass / Buffer Upload / Draw
```

可以按 Window 或 Invalidation Root 并行生成 Batch，最后按稳定顺序合并。

UE5.5 已经把 Slate 部分渲染迁移到异步 RDG，官方称大约一半 Slate 渲染可异步执行，这验证了“并行渲染后半段，而不是并行 UObject/Widget 求值”的方向。[Epic：UE5.5 Release Notes](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-5-5-release-notes?lang=en-US)、[SlateRHIRenderer API](https://dev.epicgames.com/documentation/unreal-engine/API/Runtime/SlateRHIRenderer?lang=en-US)

---

## 9. 大量世界空间 UI：共享 RT Atlas

如果项目有大量 `WidgetComponent`，例如血条、名字、伤害提示，那么“一控件一 RenderTarget”通常成本很高。

可以在引擎里做 `FWorldWidgetAtlasManager`：

- 相同尺寸等级的 Widget 分配到共享 RT Atlas。
- 仅刷新脏 Tile。
- Actor 上只绘制一个采样 Atlas 子区域的 Quad。
- 相同材质的 Quad 做 Instancing。
- 距离远时降低 Tile 更新频率或分辨率。

这一机制对 MMORPG、RTS、大量 NPC 项目通常比普通 UMG 微调更有效。

---

## 10. 增量更新 HitTestGrid

大型交互界面的另一项隐性成本是 Hit Test、焦点和导航数据重建。

当前 Invalidation Root 已经持有 Root HitTestGrid，可以继续加强为：

- Geometry/Layout 变化只更新受影响的 Cell。
- 纯 Paint 变化不更新 HitTestGrid。
- Visibility 变化只增删对应 Widget。
- ChildOrder 改变只重排相关范围。
- 不可交互子树跳过 HitTest 数据生成。
- 缓存导航邻接关系，只在布局变化时更新。

这类优化不会减少 Draw Call，但能降低复杂菜单、背包和编辑器式 UI 的 Game Thread 时间。

---

## 推荐的 UE4.27 落地顺序

第一阶段，投入产出比最高：

1. Global Invalidation A/B 测试。
2. 添加 Slow Path 和失效原因追踪。
3. 修复过度 `Layout/ChildOrder` 失效。
4. 回移植注册式 Slate Attribute/Field Notify。
5. 建立全局 Widget Pool。
6. Retainer RT 池和刷新预算。

第二阶段：

1. Widget Tick/Timer 分级调度。
2. Batch Split 原因统计。
3. Cook 阶段 UI Atlas。
4. 增量 HitTestGrid。
5. 若存在大量 `WidgetComponent`，实现共享 RT Atlas。

第三阶段，作为实验分支：

1. 持久化 GPU 几何缓存。
2. DrawElement 快照后的并行 Batch。
3. RDG 异步 Slate Pass。
4. Texture Array/Bindless 多纹理 Batch。

最关键的判断是：

- **GT 高**：优先失效系统、Attribute、Tick/Timer、HitTest。
- **RT 高**：优先 Batch、Atlas、GPU 几何缓存、并行 Batch。
- **GPU 高**：优先 Overdraw、Retainer 调度、共享 RT、减少 Effect Pass。
- **偶发卡顿**：优先 Widget Pool、RT Pool、字体预热、资源异步加载。

对于你这份 UE4.27，我最推荐首先做“**Global Invalidation 正确性追踪 + UE5 Slate Attribute 回移植 + Retainer 全局预算**”。这三项不要求业务美术资源重做，又能覆盖 Game Thread、绑定求值和 GPU 峰值三个主要问题。

> 以上方案是针对UE5的改进，有没有其他游戏项目现有的优化机制呢？请你联网搜索下

<details><summary>2 previous messages</summary>

> 我会把检索范围扩展到公开的商业项目与厂商技术分享，优先找能确认“已实际落地”的机制，并把 Unreal 项目、其他引擎项目和可回移植到 UE4.27 的部分分开。
>
> <details><summary>Searched the web</summary>
>
> Searched the web for `site:dev.epicgames.com/community/learning/talks-and-demos Unreal Engine UMG UI optimization game project Slate ...`
>
> Searched the web for `游戏 UI 性能优化 技术分享 项目 UMG Slate UE4 腾讯 ...`
>
> Searched the web for `site:gdcvault.com/play UI rendering game one draw call ...`
>
> Searched the web for `'performance'`
>
> Searched the web for `"User Interface in Cyberpunk 2077" optimizations slides ...`
>
> Searched the web for `Call of Duty LUI UI performance optimization Lua rendering ...`
>
> Searched the web for `"OPERATION UI CODE" "Lua O(n) iteration" ...`
>
> Searched the web for `Star Citizen Building Blocks UI performance optimization engine ...`
>
> Searched the web for `"Dead by Daylight" "Integrating Shaders into the User Interface Pipeline" slides ...`
>
> Searched the web for `site:noesisengine.com games case study UI optimization rendering ...`
>
> </details>
</details>

有，而且一些商业项目采用的机制比 UE5 的常规 Invalidation/Retainer 更激进。公开资料中最值得参考的是《Call of Duty》《赛博朋克 2077》《SimCity》以及 Guerrilla 的 Decima UI Renderer。

需要先说明：这些项目大多使用自研引擎，不能直接复制代码，但机制可以移植到 UE4.27/Slate。

## 一、《Call of Duty》：为 UI 建立严格的 CPU 预算系统

Activision 2024 年公布的案例非常有价值。《Call of Duty》的 HUD 在 PS4 基础型号、Ground War 25 个 Bot 场景中，主线程持续占用曾达到 7～8.5ms；UI 更新甚至会造成连续丢帧或超过 100ms 的停顿。

他们采取的措施包括：

- Element Invalidation。
- Quad Cache。
- 减少 UI 树遍历。
- Lazy Initialization。
- Staggered Processing，跨帧分摊工作。
- Lua 查询从 `O(n)` 改为原生 C++ 的 `O(log n)`。
- 数据缓存，以内存换计算。
- Direct Event Dispatch，绕过不必要的事件链。
- Font Cache Priming。
- UI 垃圾回收时间切片。
- 自动性能测试和 Dashboard。

尤其值得注意的是 UI 脚本 GC：他们不是让 Lua 一次完成完整 GC，而是给每帧 GC 一个约 100μs 的时间槽。最终公布的 PS4 基础平台 UI 时间为：

| 场景 | UI CPU 时间 |
|---|---:|
| 单人 | 1.2ms |
| 多人 | 2.1ms |
| Ground War | 3.1ms |
| Warzone | 3.9ms |

来源：[Activision Research](https://research.activision.com/publications/2024/06/operation_-ui_-code)、[完整技术幻灯片](https://www.activision.com/cdn/research/2024REAC-ATVI-Simon-Eschbach-Optimizing-UI.pdf)

### 可移植到 UE4.27 的机制

可以实现一个 `FUIFrameBudgetManager`，不再允许 UI 工作无限制地集中在某一帧：

```text
本帧 UI 预算
├─ Widget 创建：最多 4 个 / 300μs
├─ 异步资源完成后的实例化：400μs
├─ 非关键动画求值：300μs
├─ 字体 Glyph 预热：100μs
├─ Pool 清理：100μs
└─ Retainer 更新：最大 2 个 / 最大 N 像素
```

任务超出预算就进入下一帧，但鼠标、焦点、输入反馈和关键 HUD 保持即时执行。

比单纯“减少 Tick”更重要的是将 UI 任务分类：

- `Critical`：输入、准星、受击反馈。
- `Interactive`：当前打开菜单。
- `Visible`：可见但没有交互。
- `Background`：预加载、字体生成、缓存建立。
- `Disposable`：对象池裁剪、延迟销毁。

这套机制是 UE4.27 很值得增加的，因为原生 Slate/UMG 缺少面向整个项目的统一微秒预算器。

---

## 二、《Call of Duty》：把脚本层数据扫描下沉到 C++ 索引服务

他们公开的一个问题是类似：

```text
Weapon.GetStickers()
```

脚本层进行线性遍历，数据增长后出现数百万次迭代，最坏达到几十秒。解决方式是把 Lua 的 `O(n)` 查询替换成 C++ 的 `O(log n)` 索引查询。[Activision 幻灯片](https://www.activision.com/cdn/research/2024REAC-ATVI-Simon-Eschbach-Optimizing-UI.pdf)

对应到 UMG，常见问题是：

- Binding 中遍历背包。
- 每帧搜索任务列表。
- 根据 ID 线性查找配置表。
- `GetAllActorsOfClass` 后筛选。
- Blueprint 数组每帧排序。
- 每个格子单独查询 Gameplay 系统。

可在引擎或项目基础层添加 `UUIDataStoreSubsystem`：

```cpp
struct FUIDataSnapshot
{
    uint32 Version;
    TArray<FItemViewData> VisibleItems;
    TMap<FItemId, int32> ItemIndex;
    FBitArray ChangedFields;
};
```

关键机制是：

1. Gameplay 一次生成不可变 Snapshot。
2. UI 不再逐控件跨 UObject/Blueprint 查询。
3. Widget 使用 ID 访问 C++ 索引。
4. 只把 Delta 推送给变化控件。
5. Snapshot 可以安全地在工作线程准备。
6. Game Thread 只提交最终变化。

这比普通 MVVM 更进一步：它不仅是事件驱动，还是专门面向 UI 的只读数据副本和查询加速层。

---

## 三、《赛博朋克 2077》：UI Layer 级异步调度

《赛博朋克 2077》的 INK UI 系统要支持夜之城中数千个独立 UI 实例，包括：

- HUD 和菜单。
- 车辆内部 UI。
- 武器显示。
- 广告牌。
- 道路标志。
- 电视节目。
- 世界空间本地化文字。

CDPR 最初的 UI 是单线程执行，公开报告称优化起点约 10～15ms，目标是在高同步线程上控制到 3～5ms。其核心不是简单地给 Widget 加缓存，而是先把 UI 分成独立 Layer：

```text
Screen Layer
World UI Layer
Advertisement Layer
Road Sign Layer
TV Layer
Offscreen Layer
Debug Layer
```

每个 Layer 具有独立的：

- 更新链。
- 动画调度。
- 资源管理。
- 绘制逻辑。
- Render Target。
- 多线程任务。
- 生命周期状态。

独立计算异步执行，最后只进行同步合并。来源：[GDC 官方介绍](https://gdcvault.com/play/1034160/User-Interface-in-Cyberpunk-2077)、[GDC 演讲详细整理](https://www.gcores.com/articles/182620)、[4Gamer 演讲报道](https://www.4gamer.net/games/189/G018964/20230714067/)

### 对 UE4.27 的启发

不要尝试让所有 `SWidget::OnPaint` 并行，而是建立粗粒度的 `FSlateUILayer`：

```text
FSlateUILayer
├─ Invalidation Root
├─ Data Snapshot
├─ Animation Queue
├─ Pending Spawn Queue
├─ Draw Snapshot
├─ Render Target（可选）
└─ Frame Budget / Priority
```

更新过程：

```text
工作线程：
数据准备 → 动画插值 → 可见性筛选 → 生成更新命令

游戏线程：
应用属性 → Slate Prepass/OnPaint → 生成不可变 Draw Snapshot

渲染线程：
Batch → RT/Backbuffer
```

这样既能多线程，又不会让 UObject、Blueprint 和 Slate Widget 直接暴露给工作线程。

---

## 四、《赛博朋克 2077》：Active/Passive 双状态 UI

他们把 UI 实例分成两种执行状态：

### Active

- 执行完整逻辑。
- 更新布局。
- 求值动画。
- 处理事件。
- 生成绘制数据。

### Passive

- 不执行普通 Tick。
- 不计算不可见动画。
- 只推进逻辑时间。
- 只派发关键事件。
- 保留轻量状态。
- 恢复 Active 时根据当前时间直接求值。

其中不可见动画不会逐帧插值，只推进时间并保留有意义的事件；动画模板只存一份，每个实例只保留轻量参数。[GDC 演讲整理](https://www.gcores.com/articles/182620)

UE4.27 可以在 `UUMGSequencePlayer`/动画管理器中增加：

```cpp
enum class EUIExecutionState
{
    Active,
    Passive,
    Suspended
};
```

例如一个隐藏 5 秒的动画，不需要计算中间 300 帧；恢复时直接求值到第 5 秒，只补发必须执行的逻辑事件。

这对大量常驻页面、世界空间 UI 和 Live Service 菜单非常有效。

---

## 五、《赛博朋克 2077》：异步创建，并限制每帧激活数量

INK 的 UI 实例默认可以异步生成，并支持：

- 每帧最大实例激活数量。
- 延迟或取消创建。
- 资源加载队列。
- 实例池。
- 保证异步任务的提交顺序。
- 只在有帧时间余量时处理 Offscreen 工作。

这解决的不是平均帧时间，而是打开背包、地图、商店时的卡顿。

对应 UE4.27，可将：

```cpp
CreateWidget()
RebuildWidget()
SynchronizeProperties()
Slate Prepass
字体首次生成
纹理首次加载
```

拆成一条可取消管线：

```text
请求页面
 → 异步加载 WidgetClass/资源
 → 创建 UObject
 → 分帧构建子树
 → 分帧预热字体和材质
 → 最后提交到 Viewport
```

需要注意：`CreateWidget` 和 Slate Widget 构造仍然应在 Game Thread，所谓异步主要是资源加载、数据准备和跨帧分摊，并非在工作线程创建 UObject。

---

## 六、《赛博朋克 2077》：世界空间 UI 的 HLOD 与剔除系统

这是非常值得 UE4 项目借鉴、而 UE 默认 `WidgetComponent` 又比较缺乏的机制。

CDPR 为大量广告牌、道路标志和世界 UI 增加了：

- 距离剔除。
- 视锥剔除。
- 运动预测和惯性，防止边缘频繁开关。
- 自定义软件遮挡剔除。
- 不可见实例自动进入 Passive。
- 根据屏幕占比调整质量。
- 远距离使用静态纹理替代动态 UI。
- 世界 UI HLOD。
- Render Target Atlas。
- 独立 Render Object Pool。
- 仅绘制 RT 中实际需要的局部区域。

CDPR 还专门创建了 Render Target Manager，根据距离选择 UI HLOD，或者保留 RT 尺寸但仅刷新其中一部分，以降低内存压力和碎片。[4Gamer 报道](https://www.4gamer.net/games/189/G018964/20230714067/)

UE4.27 可实现：

```text
屏幕面积 > 20,000 px²：完整动态 UMG，60/30Hz
屏幕面积 > 4,000 px²：简化 Widget，15Hz
屏幕面积 > 500 px²：缓存静态 RT，2Hz
屏幕面积更小：图标/代理 Mesh
被遮挡：Passive
远距离：完全剔除
```

再增加滞回：

```text
进入高 LOD：屏幕面积 > 5000
退出高 LOD：屏幕面积 < 3500
```

避免 UI 在阈值附近反复创建和销毁。

---

## 七、《SimCity》：Dirty Region + 硬件合成层

Maxis 的《SimCity》使用 EA WebKit 和自研 MUiLE。虽然不是 Unreal，但它暴露了几个很典型的问题：

- C++ 与脚本桥接通信很贵。
- 重复跨桥查询会产生大量开销。
- 层叠透明动画在软件渲染下可能产生约 10ms 卡顿。
- 应使用硬件合成层。
- 必须能够可视化 Dirty Rect。
- 两侧缓存数据，减少 C++/JS 往返。
- 大量动态分配应改为对象池。
- 游戏与 UI 使用异步 Callback，更适合多线程。

来源：[Maxis GDC 完整幻灯片](https://media.gdcvault.com/gdc2015/presentations/Chin_YeeCheng_HowToImplement.pdf)、[GDC 演讲页](https://www.gdcvault.com/play/1022054/How-to-Implement-AAA-Game)

映射到 UE：

```text
Gameplay UObject  ←昂贵边界→  Blueprint/UMG
```

因此不要让 100 个 Widget 各自调用 Getter；应一次性发送：

```cpp
FInventoryUIDelta
{
    AddedItems;
    RemovedItems;
    ModifiedItems;
    NewSelection;
};
```

硬件合成层则可映射成“自动 Retainer 提升”：

- 大面积移动/淡入淡出子树临时提升为 RT Layer。
- 动画结束后释放 RT。
- 静态背景保持普通 Invalidation Cache。
- 不为每个小 Widget 永久创建 Retainer。

这更接近浏览器的 Compositor Layer，而不是由 UI 美术手工放置 Retainer Box。

---

## 八、Decima：Tile-Based UI，一次调用绘制数千个图元

Guerrilla 的 Decima 工具 UI 使用 `UIPainter`：

1. 把矩形、圆角、线段、曲线等转换成统一的图元描述。
2. 通过 Tile Binning 把图元分配到屏幕 Tile。
3. 使用 SDF 表达抗锯齿形状。
4. GPU 按 Tile 计算图元覆盖。
5. 数千个图元可以在一次 Draw Call 中完成。

来源：[GDC 2024：UIPainter—Tile-Based UI Rendering in One Draw Call](https://gdcvault.com/play/1034768/UIPainter-Tile-Based-UI-Rendering)

需要诚实说明：公开演讲针对 Decima 编辑器 UI，不是某个游戏 HUD；但它属于实际生产环境中的引擎 UI Renderer，而非理论论文。

### 在 Slate 中如何实现

可以增加一条混合渲染路径：

```text
普通 Slate Renderer
├─ Texture/Image
├─ Material
├─ Video
└─ Custom Drawer

Simple Primitive Renderer
├─ Rectangle
├─ Rounded Rectangle
├─ Border
├─ Line
├─ Circle
├─ Progress Shape
└─ SDF Icon
```

简单图元写入 Structured Buffer：

```cpp
struct FSlateGPUPrimitive
{
    FVector4 Bounds;
    FVector4 Color;
    FVector4 Params;
    uint32 Type;
    uint32 ClipIndex;
    int32 Layer;
};
```

然后：

```text
Compute：Tile Binning
Compute/Pixel：SDF Coverage
Composite：一次或少数几次 Draw
```

这可以绕过 Slate 传统的“一批 Quad 顶点 + 不同资源切 Batch”模式。

限制是：

- 复杂纹理和 UI Material 仍走原 Slate。
- 严格 Z 顺序和嵌套裁剪比较困难。
- 移动端 Compute 能力需单独评估。
- 文本最好先保持现有 Font Atlas。

因此适合做 Hybrid Renderer，而不是完全替换 Slate。

---

## 最值得从这些项目移植到 UE4.27 的五项机制

按投入产出比排序：

1. **Call of Duty 式 UI Frame Budgeter**
   统一限制创建、加载完成提交、字体预热、Retainer 刷新和清理任务。

2. **UI Data Snapshot + 原生索引查询**
   消除 Blueprint Binding、跨 UObject 调用和脚本线性扫描。

3. **Cyberpunk 式 Active/Passive Layer 调度**
   不可见动画只推进时间，Layer 独立调度，最终同步提交。

4. **世界空间 UI HLOD + RT Atlas Manager**
   如果项目存在大量血条、名字、路牌或设备屏幕，这是收益最大的一项。

5. **自动化性能回归系统**
   记录 Median、1% Worst、0.1% Worst，而不只是平均时间；对背包打开、任务刷新、队员死亡等事件建立自动测试。

如果项目主要瓶颈是常规 HUD/菜单，应优先做前 3 项；如果有大量 `WidgetComponent`，则世界 UI HLOD 和共享 RT Atlas 应提升到最高优先级。Decima 式 Tile/SDF Renderer 很有吸引力，但属于需要独立研发和长期维护的高风险方案。

> 你来分析源码，你觉得有可以增加哪些优化的机制呢？从技术路线说明可行性与风险点。

<details><summary>5 previous messages</summary>

> 这次我会直接以你当前 UE4.27 源码为基准做架构审计，不再泛泛列最佳实践。先定位 Slate Tick/失效、绘制合批、UMG 动画、WidgetComponent 与 Retainer 的关键路径，再给出可插入机制、改动边界和风险分级。
>
> 目前源码里已经能确认两个很明确的结构性机会：一是失效系统把 Tick、ActiveTimer、重绘混在同一个堆里串行处理；二是 SlateRHI 即使复用了 CachedBatch，仍会在每帧合并后把整份最终顶点/索引数据重新上传。这两点分别对应低～中风险的“更新队列分级”和高收益但高风险的“持久化 GPU 缓存”。我继续核对 Retainer、WidgetComponent、字体和池化路径。
>
> Context compaction
>
> Source: automatic
> Status: completed
>
> 我会再补查字体缓存与文本塑形路径，确认是否已有缓存、哪些“优化建议”其实已经被 UE4.27 实现，避免把存量能力重复包装成新机制。之后我会按收益、侵入性和回归风险给出分阶段路线。为保证分析流程完整，我也会使用 `using-superpowers` 技能核对这类源码调研该如何组织验证边界。
>
> 这个问题属于“探索可行机制、尚未要求落地实现”，我会同时按 `openspec-explore` 的边界来做：只形成基于源码的候选设计、插入点和风险，不擅自改引擎代码。
</details>

基于这份 UE4.27 源码，我认为最值得做的不是再加一个“更大的 InvalidationBox”，而是把 UI 管线改成：

```text
属性/动画变化
      ↓ 精确变化原因
按 Root 调度失效、Tick、Timer
      ↓
增量生成 CPU Batch
      ↓
稳定哈希合批
      ↓
静态几何 GPU 常驻 + 动态几何 RingBuffer
      ↓
Retainer / WidgetComponent 统一预算调度
```

当前源码有几个明显的结构性瓶颈：

- 全局失效系统存在但默认关闭：[SlateCoreClasses.cpp](./Engine/Source/Runtime/SlateCore/Private/SlateCoreClasses.cpp:46)。
- Dirty、Volatile、Tick、ActiveTimer 共用一个 `WidgetsNeedingUpdate` 堆：[SlateInvalidationRoot.h](./Engine/Source/Runtime/SlateCore/Public/FastUpdate/SlateInvalidationRoot.h:158)。
- UMG 动画虽然已经支持并行求值，但播放期间仍可能每帧 `InvalidatePrepass()`：[UserWidget.cpp](./Engine/Source/Runtime/UMG/Private/UserWidget.cpp:1402)。
- CPU 侧已经缓存了 Slate 顶点、索引和 Batch，但最终仍在每帧重新拼接，并把完整 VB/IB 上传 GPU：[DrawElements.h](./Engine/Source/Runtime/SlateCore/Public/Rendering/DrawElements.h:330)、[SlateRHIRenderingPolicy.cpp](./Engine/Source/Runtime/SlateRHIRenderer/Private/SlateRHIRenderingPolicy.cpp:135)。
- Retainer 的预算只按“本帧画了几个”计数，完全不区分 256×256 和 4K RT：[SRetainerWidget.cpp](./Engine/Source/Runtime/UMG/Private/Slate/SRetainerWidget.cpp:368)。
- WidgetComponent 主要依赖“最近 0.5 秒是否被渲染”做粗粒度裁剪，没有统一预算、屏幕面积 LOD 和优先级：[WidgetComponent.cpp](./Engine/Source/Runtime/UMG/Private/Components/WidgetComponent.cpp:1240)。

## 1. 每个 Invalidation Root 独立启用 Fast Path

不要直接全局打开 `Slate.EnableGlobalInvalidation`，而是增加：

- `Auto / ForceFast / ForceSlow` 三种 Root 模式。
- 窗口、Retainer、复杂面板分别拥有独立状态。
- 每隔 N 帧抽样执行一次 Slow Path 校验。
- 比较 DesiredSize、HitTestGrid、绘制元素摘要和失效传播结果。
- 检测到不兼容 Widget 后，只把对应 Root 降级，不拖累全局。

技术插入点主要在 [SlateInvalidationRoot.cpp](./Engine/Source/Runtime/SlateCore/Private/FastUpdate/SlateInvalidationRoot.cpp:500)。

可行性：高。现有 FastPath、Root、GenerationNumber 和 SlowPath 都已具备。

风险：中。旧的自定义 Slate Widget 可能没有正确调用 `Invalidate()`。校验器必须是开发版能力，Shipping 只保留降级名单。

这是我建议最先做的机制之一，因为它能让已有 FastPath 真正安全落地。

## 2. 事件驱动绑定与精确失效图

当前 UMG `PROPERTY_BINDING` 最终仍是绑定到 Getter 的 `TAttribute`：[Widget.h](./Engine/Source/Runtime/UMG/Public/Components/Widget.h:106)。它没有“这个字段发生变化，只影响 Paint”这样的依赖图。

可以增加 `FCompiledUIBinding`：

```text
源字段版本变化
    ↓
更新缓存值
    ↓
Layout / Paint / Visibility / RenderTransform
    ↓
只把目标 Widget 放进对应失效队列
```

技术路线：

- Widget Blueprint 编译阶段生成绑定访问器和失效原因。
- 为支持通知的数据增加 `INotifyUIFieldChanged` 或生成字段版本号。
- Slate Attribute 读取缓存值，不再每次调用 Blueprint Getter。
- 任意函数绑定、带副作用函数继续走 Legacy Polling。
- 提供三种模式：`Notify`、`IntervalPoll`、`EveryFrameLegacy`。

可行性：中高；预计是复杂 UMG 项目里 CPU 收益最大的方向。

主要风险：

- Blueprint Getter 可能偷偷依赖多个字段，甚至包含副作用。
- 并非所有 UObject 属性写入都经过统一 Setter。
- 生命周期、Delegate 解绑和对象复用容易产生悬挂订阅。

因此必须采用“显式 opt-in + 编译器警告 + Legacy fallback”，不能自动把全部绑定转成通知模式。

## 3. 动画轨道级失效

UE4.27 已经通过 `UUMGSequenceTickManager` 做了集中、并行的 MovieScene 求值：[UMGSequenceTickManager.cpp](./Engine/Source/Runtime/UMG/Private/Animation/UMGSequenceTickManager.cpp:66)。真正的问题是求值之后失效过重。

可以在动画编译阶段给 Track 分类：

- Opacity、Color：`Paint`
- RenderTransform：`RenderTransform | Paint`
- Visibility：`Visibility`
- Slot Size、Padding、Text：`Layout`
- 自定义或未知轨道：保守回退 `Layout`

求值结束后汇总实际变化的 Mask，一次性提交给 Root，不再统一 `InvalidatePrepass()`。

还可以增加被遮挡动画的 Passive 模式：

- 不逐帧求值视觉属性。
- 只推进时间和必须触发的事件。
- 再次可见时直接 Seek 到当前时间。

可行性：高。

风险：中。自定义轨道可能间接改变布局；动画事件、Latent Action 和结束回调的顺序必须保持。未知 Track 一律走保守路径。

## 4. 把 Batch 合并从近似 O(B²) 降到 O(B)

当前 `MergeRenderBatches()` 先稳定排序，然后同层向后扫描寻找兼容 Batch：[ElementBatcher.cpp](./Engine/Source/Runtime/SlateCore/Private/Rendering/ElementBatcher.cpp:165)。复杂界面同层 Batch 多时会出现平方级扫描。

可以构造完整 BatchKey：

- ShaderResource
- ShaderType、DrawFlags、Primitive、DrawEffects
- ShaderParams
- ClippingState
- InstanceData
- DynamicOffset
- CustomDrawer、SceneIndex

稳定遍历排序结果，用：

```cpp
TMap<LayerId, TMap<BatchKey, RepresentativeBatch>>
```

直接找到当前层第一个可合并 Batch。这样仍保持现有“同层稳定顺序”和“第一个可合并批次吸收后续批次”的语义。

可行性：很高。风险：低。

需要注意：

- `bIsMergable` 当前具有非对称语义，不能只把它塞进 Key。
- 哈希命中后仍做一次完整 `IsBatchableWith()` 校验。
- 用 CVar 保留旧算法，并比较最终 Batch 数、索引数据和渲染截图。

这是最适合先落地的低风险源码优化。

## 5. 持久化 GPU 几何缓存

这是底层收益潜力最大、风险也最高的路线。

当前 FastPath 已经缓存：

- `FSlateCachedFastPathRenderingData::Vertices`
- `Indices`
- `FSlateCachedElementData::CachedBatches`

但 `BuildRenderingBuffers()` 仍会把所有缓存和动态数据重新拼成 `FinalVertexData/FinalIndexData`，然后锁定并覆盖整个 Master Buffer。

可以改成双通道：

```text
静态/缓存 Widget → GPU Slab，只有失效时更新
动态/Volatile   → 每帧 RingBuffer
最终 Batch      → 引用 BufferHandle + Vertex/Index Offset
```

源码甚至留下了不完整的相关接口痕迹，如 `FSlateCachedBufferPayload`、`FSlateRenderDataHandle` 和渲染循环中的 `LastHandle`，但在该分支里没有形成完整路径。

可行性：中。

风险：高，主要包括：

- GPU Slab 碎片与压缩。
- RHI Fence 和跨线程销毁。
- 16 位索引边界。
- 字体 Atlas、纹理、材质资源版本变化。
- ClipState 和根节点变换变化。
- 同一个最终 Batch 跨静态、动态 Buffer 后可能增加 DrawCall。

建议先做原型：只支持完全静态、无 CustomDrawer、无 InstanceData、矩形裁剪的 Batch；命中失败自动回到 Master Buffer。

## 6. Retainer 与 WidgetComponent 统一预算调度

Retainer 当前按数量限制工作量，WidgetComponent 则各自 Tick、各自 RT、各自决定是否重绘。可以增加统一的 `FUIRenderBudgetManager`：

```text
请求优先级 =
    是否可见
  × 屏幕面积
  × 距离
  × 交互状态
  × 距离上次更新时长
  ÷ 预计渲染成本
```

预算不再是“每帧 3 个”，而是：

- 最大 RT 像素数。
- 最大预计 CPU 时间。
- 最大预计 GPU 时间。
- Aging 保证低优先级对象不会永久饿死。
- 交互中、获得焦点、刚进入屏幕的 UI 强制提升优先级。

WidgetComponent 进一步增加：

- 屏幕面积 LOD。
- 更新频率分桶，例如 60/30/15/5 Hz。
- 分辨率升降滞回，避免频繁 Resize。
- 逻辑 DrawSize、碰撞尺寸与实际 RT 分辨率解耦，避免 LOD 变化触发 `RecreatePhysicsState()`。

可行性：高，尤其适合大量头顶血条、名称板、世界交互面板的项目。

风险：中。可能出现视觉延迟、分辨率跳变、输入坐标偏差。HitTest 和逻辑布局应保持实时，只允许视觉 RT 延迟。

## 7. RenderTarget 池化；Atlas 作为后续路线

Retainer 和 WidgetComponent 当前基本都是一对象一张 RT。建议先做尺寸分桶池：

- 按 Format、Gamma、ClearColor 类型和量化尺寸建立池。
- 延迟若干帧归还，等待渲染 Fence。
- 尺寸使用 64/128 像素粒度，并增加 Resize 滞回。
- 内存压力下按 LRU 回收。

可行性：中高；风险：中。

把多个 UI 放入同一 RT Atlas 能进一步减少 RT 数和状态切换，但风险明显更高：UV 重映射、边缘污染、部分清屏、材质参数、输入映射和生命周期都会复杂化。建议池化稳定后再考虑 Atlas。

## 8. 字体 Atlas 从“整体 Flush”改成分页淘汰

文本塑形和 Glyph 已经有缓存，不能简单宣称“增加字体缓存”。真正可以改进的是：Atlas 超阈值时当前会请求整体 Flush：[FontCache.cpp](./Engine/Source/Runtime/SlateCore/Private/Fonts/FontCache.cpp:1174)。

可增加：

- Atlas 页级 Generation。
- 冷页 LRU 淘汰，不清空全部字体数据。
- ShapedGlyph 记录 AtlasPage + Generation，失配时局部重新栅格化。
- 常用语言字符集异步预热。
- FreeType 后台任务生成位图，渲染线程只做 Atlas 上传。

可行性：中。风险：高。

风险集中在旧 UV 指向已复用页面、异步 FreeType 上下文安全、首帧缺字，以及文化切换。必须保留缺字占位和同步 fallback。

## 建议实施顺序

1. 先增加贯穿“属性变化 → 失效 → Paint → Batch → GPU Upload/RT”的因果 Trace。
2. 实现哈希合批、Retainer 加权预算、每 Root FastPath 开关与校验器。
3. 实现事件驱动 Binding 和动画轨道精确失效。
4. 如果项目有大量 3D UI，再做 WidgetComponent 管理器与 RT 池。
5. 最后分别对“GPU 持久几何缓存”和“字体分页淘汰”做隔离原型。

不建议优先做“多线程 OnPaint”。Slate 绘制期间会接触 SWidget、UObject、字体缓存、动态材质、资源对象和 HitTest 状态，线程安全改造面过大。更现实的路线是先减少需要 Paint 的数量，再优化 Batch，最后让已经缓存的几何真正驻留 GPU。

最终优先级取决于 Insights 数据：

- `SlatePrepass/Paint` 高：优先精确绑定和动画失效。
- `SlateRTCreateBatches/UpdateBuffer` 高：优先哈希合批和 GPU 常驻。
- `WidgetRenderer/RT` 高：优先统一预算和 RT 池。
- 世界 UI 数量大：优先 WidgetComponent 中央调度。

> [$ai-conversation-archive](C:/Users/yongkang.guo//.codex/skills/ai-conversation-archive/SKILL.md)

<details><summary>1 previous message</summary>

> System error
>
> unexpected status 401 Unauthorized: Incorrect API key provided: sk-d49b4*******************************************************fdd8. You can find your API key at https://platform.openai.com/account/api-keys., url: https://api.openai.com/v1/responses, cf-ray: a4b4a17fae3db1b4-SJC, request id: req_bd184e0c22974226941e2a420d5bd2c7, auth error: 401, auth error code: invalid_api_key
</details>

> [$ai-conversation-archive](C:/Users/yongkang.guo//.codex/skills/ai-conversation-archive/SKILL.md)

<details><summary>1 previous message</summary>

> System error
>
> unexpected status 401 Unauthorized: Missing bearer or basic authentication in header, url: https://api.openai.com/v1/responses, cf-ray: a4b4ae2c6be4eb21-SJC, request id: req_85997832295b48d888ba9094aeb39d4d
</details>

> [$ai-conversation-archive](C:/Users/yongkang.guo//.codex/skills/ai-conversation-archive/SKILL.md)

<details><summary>1 previous message</summary>

> System error
>
> unexpected status 401 Unauthorized: Missing bearer or basic authentication in header, url: https://api.openai.com/v1/responses, cf-ray: a4b4d1c61c7beb28-SJC, request id: req_ac8f783cd4484810abed04774bdd8a4d
</details>
