# Find Shared_ptr members

> 智能指针的Shared_Ptr 的成员有哪些

<details><summary>1 previous message</summary>

> 我先按 C++ 的 `std::shared_ptr` 来解释，并补充 Unreal Engine 的 `TSharedPtr` 区别；我会先核对当前可用的工作规范。
</details>

如果你指的是 C++ 标准库的 `std::shared_ptr<T>`，常用成员函数有：

- `get()`：取得原始指针
- `use_count()`：当前共享引用计数
- `unique()`：是否只有一个拥有者（C++20 已弃用）
- `reset()`：释放或更换管理对象
- `swap()`：交换两个智能指针
- `operator*()`：解引用
- `operator->()`：访问对象成员
- `operator bool()`：判断是否为空
- `operator=()`：赋值
- `owner_before()`：比较控制块的所有权顺序

概念上，`shared_ptr` 内部通常保存两个成员：

```cpp
T* ObjectPointer;       // 指向被管理对象
ControlBlock* Control;  // 控制块
```

控制块中通常保存：

```cpp
强引用计数   // shared_ptr 数量
弱引用计数   // weak_ptr 数量
删除器
分配器等信息
```

如果你问的是 Unreal Engine 的 `TSharedPtr<T>`，其核心成员类似：

```cpp
ObjectType* Object;
FSharedReferencer SharedReferenceCount;
```

常用接口包括：

```cpp
Ptr.Get();
Ptr.IsValid();
Ptr.Reset();
Ptr.GetSharedReferenceCount();
Ptr.IsUnique();
Ptr.ToSharedRef();

*Ptr;
Ptr->Member;
if (Ptr) { ... }
```

因此，最核心的答案是：共享智能指针主要由“对象指针”和“引用计数控制块”两部分组成。

> Unreal Engine的Shared_Ptr智能指针包含哪些成员

<details><summary>1 previous message</summary>

> 我按 UE 5.8 源码里的 `TSharedPtr` 定义核对一下，区分它的内部数据成员和对外成员函数。
</details>

Unreal Engine 的 `TSharedPtr<ObjectType, Mode>` 本身只有两个核心私有数据成员：

```cpp
ObjectType* Object;

SharedPointerInternals::FSharedReferencer<Mode>
    SharedReferenceCount;
```

具体含义如下：

1. `Object`

```cpp
ObjectType* Object;
```

指向实际被管理对象的裸指针，允许为 `nullptr`。

2. `SharedReferenceCount`

```cpp
SharedPointerInternals::FSharedReferencer<Mode> SharedReferenceCount;
```

引用计数管理接口。它内部保存一个控制块指针：

```cpp
TReferenceControllerBase<Mode>* ReferenceController;
```

控制块中真正保存：

```cpp
SharedReferenceCount  // 强引用数量
WeakReferenceCount    // 弱引用数量
```

因此整体结构可以理解为：

```text
TSharedPtr
├── Object -----------------> 实际对象
└── SharedReferenceCount
    └── ReferenceController -> 引用控制块
        ├── SharedReferenceCount
        ├── WeakReferenceCount
        └── DestroyObject()
```

此外，`TSharedPtr` 还有两个公开的类型信息成员：

```cpp
using ElementType = ObjectType;
static constexpr ESPMode Mode = InMode;
```

需要注意：`TSharedPtr` 并不直接把引用次数存成一个整数，而是通过 `FSharedReferencer` 指向所有共享指针共同使用的控制块。

源码位置：[SharedPointer.h](./Engine/Source/Runtime/Core/Public/Templates/SharedPointer.h:1283) 和 [SharedPointerInternals.h](./Engine/Source/Runtime/Core/Public/Templates/SharedPointerInternals.h:47)。

> 具体说说TSharedPtr 并不直接把引用次数存成一个整数，而是通过 FSharedReferencer 指向所有共享指针共同使用的控制块。

可以把 `TSharedPtr` 理解成一个“本地指针 + 共享账本入口”。

每个 `TSharedPtr` 对象自身大致包含：

```cpp
template<class ObjectType, ESPMode Mode>
class TSharedPtr
{
private:
    ObjectType* Object;

    SharedPointerInternals::FSharedReferencer<Mode>
        SharedReferenceCount;
};
```

而 `FSharedReferencer` 内部不是引用计数整数，而是一个指针：

```cpp
template<ESPMode Mode>
class FSharedReferencer
{
private:
    TReferenceControllerBase<Mode>* ReferenceController;
};
```

真正的计数位于所有相关智能指针共同指向的控制块：

```cpp
template<ESPMode Mode>
class TReferenceControllerBase
{
    RefCountType SharedReferenceCount{1};
    RefCountType WeakReferenceCount{1};

    virtual void DestroyObject() = 0;
};
```

整体关系如下：

```text
TSharedPtr A                         实际对象
├── Object ------------------------> FMyObject
└── FSharedReferencer
    └── ReferenceController -----┐
                                 │
TSharedPtr B                      │
├── Object ----------------------│----> 同一个 FMyObject
└── FSharedReferencer             │
    └── ReferenceController ------┤
                                  ▼
                         共享的引用控制块
                         ├── SharedReferenceCount = 2
                         ├── WeakReferenceCount
                         └── DestroyObject()
```

也就是说，`A` 和 `B` 各自有一个 `Object`，也各自有一个 `FSharedReferencer`，但两个 `FSharedReferencer` 内部指向同一个控制块。

### 为什么不能把计数直接放进 `TSharedPtr`

假设每个 `TSharedPtr` 都直接保存：

```cpp
int32 ReferenceCount;
```

复制以后就会产生两份互不相关的数字：

```cpp
TSharedPtr<FMyObject> A = ...;
TSharedPtr<FMyObject> B = A;
```

如果 `A.ReferenceCount` 和 `B.ReferenceCount` 是两个独立整数，销毁 `A` 时无法可靠地知道 `B` 是否仍然存在。

因此必须把计数放在共享控制块里：

```cpp
A.ReferenceController == B.ReferenceController;
```

所有副本都修改同一份引用计数。

### 创建时发生什么

例如：

```cpp
TSharedPtr<FMyObject> A =
    MakeShared<FMyObject>();
```

概念上发生：

```text
1. 创建 FMyObject
2. 创建引用控制块
3. SharedReferenceCount 初始化为 1
4. A.Object 指向 FMyObject
5. A.SharedReferenceCount 指向控制块
```

此时：

```text
强引用计数 = 1
```

使用 `MakeShared` 时，对象和控制块通常会采用更紧凑的联合分配方式，但逻辑关系不变。

### 复制时发生什么

```cpp
TSharedPtr<FMyObject> B = A;
```

复制构造函数会复制两部分：

```cpp
B.Object = A.Object;
B.SharedReferenceCount = A.SharedReferenceCount;
```

`FSharedReferencer` 的复制构造函数随后执行：

```cpp
ReferenceController->AddSharedReference();
```

结果：

```text
A.Object == B.Object
A.ReferenceController == B.ReferenceController
SharedReferenceCount == 2
```

注意，只复制控制块指针本身还不够，还必须把共享计数加一。

### 销毁或 Reset 时发生什么

当执行：

```cpp
A.Reset();
```

或者 `A` 离开作用域时，`A` 内部的 `FSharedReferencer` 被析构，概念上调用：

```cpp
ReferenceController->ReleaseSharedReference();
```

此时计数从 2 变成 1：

```text
SharedReferenceCount == 1
```

对象不会销毁，因为 `B` 仍然持有它。

之后 `B` 也被销毁：

```text
SharedReferenceCount: 1 → 0
```

当强引用计数变成 0 时，控制块调用：

```cpp
DestroyObject();
```

实际对象被销毁。

### 控制块为什么还有弱引用计数

`TWeakPtr` 也需要访问相同的控制块：

```cpp
TSharedPtr<FMyObject> Shared = MakeShared<FMyObject>();
TWeakPtr<FMyObject> Weak = Shared;
```

关系大致是：

```text
TSharedPtr
└── FSharedReferencer ───┐
                         ├──> 控制块
TWeakPtr                  │    ├── 强引用计数
└── FWeakReferencer ─────┘    └── 弱引用计数
```

当最后一个 `TSharedPtr` 消失时：

```text
强引用计数变成 0
→ 实际对象销毁
```

但是，如果仍有 `TWeakPtr`，控制块不能立刻销毁，因为弱指针还需要通过控制块判断对象是否过期。

只有弱引用计数也变成 0，控制块本身才会销毁：

```text
强引用计数 == 0 → 销毁对象
弱引用计数 == 0 → 销毁控制块
```

因此，“对象的生命周期”和“控制块的生命周期”并不完全相同。

### `FSharedReferencer` 的职责

`FSharedReferencer` 可以看成控制块指针的 RAII 包装器，主要负责：

```text
默认构造       → 控制块指针为空
复制构造       → 共享计数 +1
复制赋值       → 新控制块 +1，旧控制块 -1
移动构造       → 转移控制块指针，不增加计数
析构           → 共享计数 -1
查询计数       → 访问控制块中的计数
```

所以，引用计数的增加和减少并不是主要由 `TSharedPtr` 手动处理，而是封装在 `FSharedReferencer` 的复制、移动和析构行为中。

### 移动为什么不增加计数

```cpp
TSharedPtr<FMyObject> B = MoveTemp(A);
```

移动后：

```text
B.Object = 原来的 A.Object
B.ReferenceController = 原来的控制块
A.Object = nullptr
A.ReferenceController = nullptr
```

拥有者只是从 `A` 转移到了 `B`，有效强引用总数没有变化，因此不需要增加计数。

### `Object` 与控制块管理的对象不一定相同

一般情况下：

```cpp
TSharedPtr<FMyObject> Ptr;
```

`Ptr.Object` 就是控制块负责销毁的对象。

但 UE 支持别名构造：

```cpp
TSharedPtr<FContainer> Container = MakeShared<FContainer>();

TSharedPtr<FElement> ElementPtr(
    Container,
    &Container->Element
);
```

这时：

```text
Container.Object  → FContainer
ElementPtr.Object → FContainer 内部的 Element

但两者共享同一个控制块
```

因此：

```cpp
Container.Get() != ElementPtr.Get();
```

但生命周期由同一个控制块管理。`ElementPtr` 不会单独删除 `Element`，最后一个共享引用消失时销毁的是整个 `FContainer`。

这也是 UE 为什么需要同时保存：

```cpp
ObjectType* Object;
FSharedReferencer SharedReferenceCount;
```

不能只通过控制块取得对象地址。

### 线程安全模式

```cpp
TSharedPtr<FMyObject, ESPMode::ThreadSafe>
```

控制块中的计数类型是：

```cpp
std::atomic<int32>
```

非线程安全模式则使用普通整数：

```cpp
int32
```

需要注意，`ThreadSafe` 保证的是引用计数增减的线程安全，不代表 `FMyObject` 自身的成员访问是线程安全的。

总结来说：

```text
TSharedPtr 本身：
    保存“我指向哪里”
    保存“共享控制块的入口”

FSharedReferencer：
    管理控制块指针
    在复制、赋值、析构时调整强引用

ReferenceController：
    保存真正的强、弱引用计数
    决定何时销毁对象
    决定何时销毁控制块
```

> [$ai-conversation-archive](C:/Users/yongkang.guo//.codex/skills/ai-conversation-archive/SKILL.md)

<details><summary>1 previous message</summary>

> System error
>
> unexpected status 401 Unauthorized: Missing bearer or basic authentication in header, url: https://api.openai.com/v1/responses, cf-ray: a4b4b70c9bb2b341-SJC, request id: req_44c05cc4626d4df1bc02911fd073ecc3
</details>

> [$ai-conversation-archive](C:/Users/yongkang.guo//.codex/skills/ai-conversation-archive/SKILL.md)

<details><summary>1 previous message</summary>

> System error
>
> unexpected status 401 Unauthorized: Missing bearer or basic authentication in header, url: https://api.openai.com/v1/responses, cf-ray: a4b4b8625b357ad3-SJC, request id: req_13219744c481470c84203533070f9fc1
</details>

> [$ai-conversation-archive](C:/Users/yongkang.guo//.codex/skills/ai-conversation-archive/SKILL.md)

<details><summary>1 previous message</summary>

> System error
>
> unexpected status 401 Unauthorized: Missing bearer or basic authentication in header, url: https://api.openai.com/v1/responses, cf-ray: a4b4d30bba5acf15-SJC, request id: req_51b17b1253894a42b2e7109b4c12ef72
</details>

> [$ai-conversation-archive](C:/Users/yongkang.guo//.codex/skills/ai-conversation-archive/SKILL.md)

<details><summary>1 previous message</summary>

> System error
>
> unexpected status 401 Unauthorized: Missing bearer or basic authentication in header, url: https://api.openai.com/v1/responses, cf-ray: a4b4dc143d064c71-SJC, request id: req_13eb6dd60f8e4aa3a7b3d35b3fe39a55
</details>
