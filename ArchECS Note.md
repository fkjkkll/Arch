# Arch ECS 学习笔记

这份笔是我学习Github开源项目[Arch ECS](https://github.com/genaray/Arch)后的学习代码，通过断点调试 + 与AI交流完成碎片化记录，最终再交由AI整合完成。主要内容包含Archetype ECS结构学习，以及一些项目里用到的技巧总结。

在学习过程中发现了两处问题，一处是BitSet的Any处包含冗余的循环代码；一处是CommandBuffer部分的SparseSet里创建SparseArray时传参错误。目前已经提交了PR。

## 总体结构

Arch 的核心数据关系可以先记成这张图：

```text
World
  ├─ ComponentRegistry              // Type -> ComponentType(Id, ByteSize)
  ├─ GroupToArchetype               // Signature hash -> Archetype
  ├─ Archetypes                     // World 中所有 Archetype 的列表
  ├─ EntityInfoStorage              // Entity.Id -> EntityData
  └─ QueryCache                     // QueryDescription -> Query

Archetype                           // 一种组件组合，比如 Position + Velocity
  ├─ Signature                      // ComponentType[]
  ├─ BitSet                         // Signature 的 bitset 形式
  ├─ _componentIdToArrayIndex        // ComponentType.Id -> Chunk.Components 下标
  └─ Chunks
       ├─ Chunk
       │    ├─ Entity[] Entities
       │    └─ Array[] Components   // 每种组件一个数组，SoA 布局
       └─ Chunk
```

一句话：**World 管所有 Archetype；Archetype 管同一种组件组合的所有实体；Chunk 才是真正连续存储 Entity 和组件数组的地方。**

## ComponentType

`ComponentType` 代表一种组件类型的运行时元信息。

```csharp
public readonly record struct ComponentType
{
    public readonly int Id;
    public readonly int ByteSize;
}
```

- `Id`：组件类型的全局唯一 id，由 `ComponentRegistry` 分配。
- `ByteSize`：组件的大小。值类型用 `Unsafe.SizeOf<T>()`，引用类型用 `IntPtr.Size`。
- `Type` 属性会通过 `ComponentRegistry.Types[Id]` 反查原始 `System.Type`。

这里的 `Id` 很关键：后续 `Signature` hash、`BitSet`、`Chunk` 的组件查找表都依赖它。

## ComponentRegistry

`ComponentRegistry` 是全局组件类型注册表。

```csharp
public static class ComponentRegistry
{
    private static readonly Dictionary<Type, ComponentType> _typeToComponentType;
    private static Type?[] _types;
    public static int Size { get; private set; }
}
```

主路径是：

```text
Component<T>.ComponentType
  -> Component<T> 静态构造
  -> ComponentRegistry.Add<T>()
  -> new ComponentType(Size, SizeOf<T>())
  -> Size++
```

也就是说，组件类型一般是在第一次使用 `Component<T>` 时注册的。

注意：`ComponentRegistry` 里的 `Size` 才是组件类型 id 的分配源。`Component` 非泛型类里的 `internal static int Id` 不是组件类型 id，它主要被生成的 `Component<T0,T1,...>` 组合缓存类使用，目前看更像内部/遗留计数。

## Component / Component<T> / Component<T0,T1>

这几个名字容易混。

```csharp
public static class Component
```

非泛型 `Component` 是工具类，主要用于：

- `GetComponentType(Type type)`：运行时 Type -> ComponentType。
- `GetHashCode(Span<ComponentType>)`：对组件组合算顺序无关的 hash。

```csharp
public static class Component<T>
{
    public static readonly ComponentType ComponentType;
    public static readonly Signature Signature;
}
```

`Component<T>` 是单个组件类型的静态缓存。第一次访问时注册组件，并缓存它自己的单组件 `Signature`。

```csharp
public static class Component<T0, T1>
{
    internal static readonly int Id;
    public static readonly Signature Signature;
    public static readonly int Hash;
}
```

`Component<T0,T1,...>` 是 T4 模板生成出来的「组件组合缓存」。真正有用的是 `Signature` 和 `Hash`，可以避免每次 `Create<T0,T1>` 都重新构造组件组合。

## ArrayRegistry

`ArrayRegistry` 用于按组件类型创建组件数组：

```csharp
public static Array GetArray(ComponentType type, int capacity)
{
    return _createFactories.TryGetValue(type.Id, out Func<int, Array> func)
        ? func(capacity)
        : Array.CreateInstance(type.Type, capacity);
}
```

它被 `Chunk` 和 `SparseArray` 使用。

`ArrayRegistry.Add<T>()` 的意图是注册 `new T[capacity]` 工厂，避免 `Array.CreateInstance` 反射创建数组。但当前主流程里基本没有自动调用它，所以默认多半会走 fallback。它更像一个预留的性能/AOT 优化点。

## Signature

`Signature` 描述一组组件类型，并缓存这组组件的 hash。

```csharp
public struct Signature : IEquatable<Signature>
{
    private int _hashCode;
    internal ComponentType[] ComponentsArray;
}
```

它代表「组件组合」，比如：

```text
Position + Velocity
Position + Velocity + Sprite
Health
```

特点：

- 可以由 `ComponentType`、`ComponentType[]`、`Span<ComponentType>` 隐式转换得到。
- 可以隐式转成 `BitSet`，用于 query 匹配。
- hash 是顺序无关的：`Position + Velocity` 和 `Velocity + Position` 应该得到同一个 hash。
- `Signature.Add` / `Signature.Remove` 会通过 `HashSet<ComponentType>` 合并/删除组件。

## QueryDescription

`QueryDescription` 是用户描述查询条件的结构。

```csharp
public partial struct QueryDescription : IEquatable<QueryDescription>
{
    private int _hashCode;
    public Signature All { get; private set; }
    public Signature Any { get; private set; }
    public Signature None { get; private set; }
    public Signature Exclusive { get; private set; }
}
```

四种条件：

- `All`：必须全部拥有。
- `Any`：至少拥有其中一个。
- `None`：不能拥有其中任何一个。
- `Exclusive`：组件组合必须完全相等，不能多也不能少。

`WithAll / WithAny / WithNone / WithExclusive` 有大量模板生成的泛型重载。当前实现里每次 `WithXXX` 都会立即 `Build()`，也就是重新计算 `QueryDescription` 的 hash：

```csharp
new QueryDescription()
    .WithNone<A>()
    .WithAny<B>()
    .WithAll<C>();
```

这个链式调用会重复 Build 三次，前两次结果马上被覆盖。功能上没问题，但从设计上可以优化成「只把 `_hashCode = -1`，等真正 `GetHashCode()` 时再懒计算」。

## BitSet

`BitSet` 是可扩容的 bit 集合，用于快速表达组件集合。

```csharp
public sealed class BitSet
{
    private const int BitSize = (sizeof(uint) * 8) - 1;        // 31
    private const int IndexSize = 5;                           // log2(32)
    private static readonly int _padding = Vector<uint>.Count; // SIMD 宽度

    private uint[] _bits;
    private int _highestBit;
    private int _max;
}
```

这里每个 `uint` 存 32 个 bit。组件 id 为 `index` 时：

```csharp
var bucket = index >> 5;          // index / 32
var offset = index & 31;          // index % 32
_bits[bucket] |= 1u << offset;
```

`_padding` 不是业务含义上的 padding，而是 SIMD 向量宽度。`_bits` 初始长度和扩容长度都会按 `Vector<uint>.Count` 对齐，方便 `All / Any / None / Exclusive` 里用 `Vector<uint>` 一次处理多个 `uint`。

方法含义：

- `All(other)`：this 的所有 set bit 都必须在 other 中存在。
- `Any(other)`：this 和 other 至少有一个 set bit 相交。
- `None(other)`：this 和 other 没有任何 set bit 相交。
- `Exclusive(other)`：this 和 other 的 set bit 完全相等。

在 Query 中，通常是：

```csharp
_all.All(archetype.BitSet)
_any.Any(archetype.BitSet)
_none.None(archetype.BitSet)
_exclusive.Exclusive(archetype.BitSet)
```

也就是用查询条件去匹配某个 archetype 的组件集合。

## Query

`Query` 是一个缓存过的查询对象。

```csharp
public partial class Query : IEquatable<Query>
{
    private readonly Archetypes _allArchetypes;
    private readonly NetStandardList<Archetype> _matchingArchetypes;
    private int _allArchetypesHashCode;

    private readonly QueryDescription _queryDescription;
    private readonly BitSet _any;
    private readonly BitSet _all;
    private readonly BitSet _none;
    private readonly BitSet _exclusive;
    private readonly bool _isExclusive;
}
```

关键点：Query 不会每次都重新扫描所有 archetype。

`Archetypes` 容器有自己的 hash。`Query.Match()` 会比较：

```csharp
var newArchetypesHashCode = _allArchetypes.GetHashCode();
if (_allArchetypesHashCode == newArchetypesHashCode)
{
    return;
}
```

如果 World 的 archetype 列表没有新增/删除，Query 就复用 `_matchingArchetypes`。如果列表变了，才重新扫描所有 archetype 并筛出匹配项。

所以 Query 的遍历路径是：

```text
QueryDescription
  -> QueryCache 找 Query
  -> Query.Match 筛 Archetype
  -> 遍历匹配 Archetype 的 Chunk
  -> 遍历 Chunk 内实体行
```

## Archetypes

`Archetypes` 是 `World` 中所有 `Archetype` 的容器包装。

```csharp
public class Archetypes : IDisposable
{
    private int _hashCode;
    public NetStandardList<Archetype> Items { get; }
}
```

它比普通 List 多了一个缓存 hash，用来告诉 Query：「World 的 archetype 列表有没有变化」。

注意：它叫 `Archetypes`，不是 ECS 概念中的 archetype 本身。它只是一个列表容器。

## Archetype

`Archetype` 存放同一种组件组合的所有实体。

```csharp
public sealed partial class Archetype
{
    public int BaseChunkSize { get; }
    public int ChunkSize { get; }
    public int EntitiesPerChunk { get; }
    public Signature Signature { get; }
    public BitSet BitSet { get; }

    private readonly int[] _componentIdToArrayIndex;
    public Chunks Chunks { get; internal set; }

    public int Count { get; internal set; }
    public int EntityCount { get; internal set; }
    public int EntityCapacity => ChunkCapacity * EntitiesPerChunk;
}
```

字段含义：

- `BaseChunkSize`：World 提供的基础 chunk 大小，默认注释里对应 16KB。
- `ChunkSize`：当前 archetype 实际 chunk 字节大小，可能是基础大小的倍数。
- `EntitiesPerChunk`：一个 chunk 可以容纳多少个 entity，由组件组合的大小决定。
- `Signature`：这个 archetype 的组件组合。
- `BitSet`：Signature 的 bitset 形式，用于 query 匹配。
- `_componentIdToArrayIndex`：组件 id 到 `Chunk.Components` 下标的查找表。
- `Chunks`：该 archetype 拥有的 chunk 集合。
- `Count`：当前正在使用的 chunk 下标，不是总实体数量。
- `EntityCount`：当前 archetype 里实体总数。
- `EntityCapacity`：当前已分配 chunk 可容纳的实体总数。

`Archetype.Add` 会把新 entity 放进当前 chunk；如果当前 chunk 满了，就移动到下一个 chunk；如果没有预分配 chunk，就创建新 chunk。

## Chunks

`Chunks` 是 `Archetype` 里管理 `Chunk` 数组的容器。

```csharp
public class Chunks
{
    private Arch.LowLevel.Array<Chunk> Items { get; set; }
    public int Count { get; set; }
    public int Capacity { get; private set; }
}
```

注意：

- `Items` 是 `Arch.LowLevel.Array<Chunk>`，底层仍然包着 `Chunk[]`。
- `Count` 是已经放入的 chunk 数量。
- `Capacity` 是逻辑容量，不等于底层数组真实长度。
- 因为 `ArrayPool<T>.Rent(1)` 可能返回长度 16 的数组，所以不能用 `Items.Length` 当逻辑容量。

`EnsureCapacity` 扩容时复制的是外层 `Chunk[]` 容器。`Chunk` 是 struct，所以会复制 struct 值；但 `Chunk` 内部的 `Entity[]`、`Array[] Components` 是引用，复制的是引用，不会深拷贝组件数据。这里这是期望行为。

## Chunk

`Chunk` 是真正存实体和组件数据的地方。

```csharp
public partial struct Chunk
{
    public readonly Entity[] Entities;
    public readonly Array[] Components;
    public readonly int[] ComponentIdToArrayIndex;

    public int Count { get; internal set; }
    public int Capacity { get; }
}
```

布局是 SoA：

```text
Entities:    [E0, E1, E2, ...]

Components:
  [0] Position[]: [P0, P1, P2, ...]
  [1] Velocity[]: [V0, V1, V2, ...]
  [2] Sprite[]:   [S0, S1, S2, ...]
```

同一个 row index 表示同一个实体的各组件：

```text
Entities[5]
Position[5]
Velocity[5]
Sprite[5]
```

`ComponentIdToArrayIndex` 用来把组件 id 映射到 `Components` 下标。例如：

```text
ComponentType.Id(Position) -> 0
ComponentType.Id(Velocity) -> 1
```

这样 `GetArray<T>()` 就能快速定位到对应的组件数组。

## Entity / EntityData / Slot

`Entity` 本身只是一个句柄。

```csharp
public readonly struct Entity
{
    public readonly int Id;
    public readonly byte WorldId;
    public readonly int Version;
}
```

真正的位置存在 `EntityInfoStorage` 中：

```csharp
public record struct Slot
{
    public int Index;       // Entity 在 Chunk 内的 row index
    public int ChunkIndex;  // Chunk 在 Archetype.Chunks 内的 index
}

public struct EntityData
{
    public Archetype Archetype;
    public Slot Slot;
    public int Version;
}
```

所以根据一个 `Entity` 找组件，大体是：

```text
Entity.Id
  -> EntityInfoStorage[Id]
  -> EntityData.Archetype
  -> EntityData.Slot
  -> Archetype.Chunks[Slot.ChunkIndex]
  -> Chunk.Components[componentIndex][Slot.Index]
```

`Version` 用来识别旧句柄，避免 entity 被销毁后 id 复用导致旧 Entity 误操作新实体。

## 创建带组件实体的流程

以批量创建为例：

```csharp
world.Create(size, transform, rotation);
```

核心流程：

```csharp
public void Create<T0, T1>(int amount, in T0? t0Component = default, in T1? t1Component = default)
{
    var archetype = EnsureCapacity<T0, T1>(amount);

    using var entityArray = Pool<Entity>.Rent(amount);
    using var entityDataArray = Pool<EntityData>.Rent(amount);

    var entities = entityArray.AsSpan();
    var entityData = entityDataArray.AsSpan();

    GetOrCreateEntitiesInternal(archetype, entities, entityData, amount);
    archetype.AddAll(entities, amount);

    var firstSlot = entityData[0].Slot;
    var lastSlot = entityData[amount - 1].Slot;
    archetype.SetRange<T0, T1>(in lastSlot, in firstSlot, in t0Component, in t1Component);

    AddEntityData(entities, entityData, amount);
}
```

拆开看：

1. `EnsureCapacity<T0,T1>(amount)`  
   根据 `Component<T0,T1>.Signature` 找到或创建对应 `Archetype`，并预留足够 chunk。

2. `Pool<T>.Rent(amount)`  
   从 `ArrayPool<T>` 租临时数组，包装成 `PooledArray`，用 `using var` 自动归还。

3. `GetOrCreateEntitiesInternal`  
   分配实体 id/version，并准备每个实体对应的 `EntityData`。

4. `archetype.AddAll`  
   把 entity 批量写入目标 chunk 的 `Entities` 数组。

5. `archetype.SetRange`  
   把组件值批量写入对应组件数组。

6. `AddEntityData`  
   把实体定位信息写入 `EntityInfoStorage`。

## World.Capacity 的含义

`World.Capacity` 不是当前实体数量，而是所有 archetype 已分配 chunk 的总实体容量。

大致关系是：

```text
World.Capacity = sum(Archetype.EntityCapacity)
```

所以 `EnsureCapacity` 中会先扣掉当前 archetype 的旧容量，扩容后再加回新容量：

```csharp
var archetype = GetOrCreate(signature);
Capacity -= archetype.EntityCapacity;
archetype.EnsureEntityCapacity(archetype.EntityCount + amount);

var requiredCapacity = Capacity + archetype.EntityCapacity;
EntityInfo.EnsureCapacity(requiredCapacity);
Capacity = requiredCapacity;
```

如果不先扣旧容量，反复 Ensure 同一个 archetype 时，`World.Capacity` 会被重复累计，导致 `EntityInfoStorage` 过度扩容。

## Query 流程

典型查询：

```csharp
var query = new QueryDescription().WithAll<Position, Velocity>();
world.Query(in query, (ref Position pos, ref Velocity vel) =>
{
    pos.X += vel.X;
});
```

大致流程：

```text
QueryDescription
  -> World.QueryCache 查 Query
  -> Query.Match 检查 Archetypes hash 是否变化
  -> 如果变化，重新筛选匹配的 Archetype
  -> 遍历 matching Archetype
  -> 遍历 Archetype.Chunks
  -> 拿到组件数组第一项 ref
  -> Unsafe.Add(ref first, rowIndex) 定位组件
```

核心优化点：

- Query 是按 archetype 筛，不是按 entity 一个个判断组件。
- 同一个 archetype 里实体组件布局相同，所以一旦 archetype 匹配，里面的 chunk 可以连续遍历。
- 组件数组是 SoA，遍历某个组件时内存连续。


## CommandBuffer

`CommandBuffer` 是一个**延迟操作缓冲区**——把对 Entity 的增删改操作先记录在 buffer 里，之后调用 `Playback(world)` 一次性执行。

这在以下场景很有用：
- 遍历 Query 时不能直接修改 World（会破坏迭代器的稳定性），需要延迟修改。
- 多线程 job 里记录操作，主线程统一回放。
- 批量操作后再统一提交，减少中间态。

### 内部数据结构

```csharp
public sealed partial class CommandBuffer : IDisposable
{
    internal PooledList<Entity> Entities;                          // 所有涉及的 Entity
    internal PooledDictionary<int, BufferedEntityInfo> _info;      // Entity.Id -> 索引信息
    internal PooledList<CreateCommand> Creates;                    // 待创建的 Entity
    internal SparseSet Sets;                                       // 待 Set 的组件值
    internal StructuralSparseSet Adds;                             // 待 Add 的组件类型
    internal StructuralSparseSet Removes;                          // 待 Remove 的组件类型
    internal PooledList<int> Destroys;                             // 待 Destroy 的 Entity
}
```

`BufferedEntityInfo` 存储一个 Entity 在 buffer 各数组中的位置：

```csharp
readonly record struct BufferedEntityInfo
{
    int Index;        // Entities 列表中的位置
    int SetIndex;     // SparseSet 中的行号
    int AddIndex;     // StructuralSparseSet 中的行号
    int RemoveIndex;  // StructuralSparseSet 中的行号
}
```

### Register — Entity 注册

对已有 Entity 的首次操作会触发 `Register`：

```csharp
internal void Register(in Entity entity, out BufferedEntityInfo info)
{
    var setIndex   = Sets.Create(in entity);       // SparseSet 分配一行
    var addIndex   = Adds.Create(in entity);       // StructuralSparseSet 分配一行
    var removeIndex = Removes.Create(in entity);   // StructuralSparseSet 分配一行

    info = new BufferedEntityInfo(Size, setIndex, addIndex, removeIndex);
    Entities.Add(entity);
    _info.Add(entity.Id, info);
    Size++;
}
```

三套 SparseSet 都会为这个 Entity 预先分配一个行号，之后对该 Entity 的 Set/Add/Remove 都通过这个行号去定位。

### 负 ID Entity — Create 的特殊处理

`CommandBuffer.Create(types)` 不会立即创建实体，而是返回一个**负 ID 的占位 Entity**：

```csharp
public Entity Create(ComponentType[] types)
{
    var entity = new Entity(-(Size + 1), -1);   // 负 ID，表示 "尚未创建"
    Register(entity, out _);
    Creates.Add(new CreateCommand(Size - 1, types));
    return entity;   // 用户拿着这个占位 entity 去 Set/Add/Remove
}
```

负 ID 的作用：此时实体还不存在于 World 中，不能直接用正 ID。但用户可以用返回的占位 entity 做后续 Set/Add 等操作，这些操作会被记录到 buffer 的行里。

`Playback` 时通过 `Resolve` 把负 ID 映射回真正创建的 Entity：

```csharp
internal Entity Resolve(Entity entity)
{
    var entityIndex = _info[entity.Id].Index;
    return Entities[entityIndex];   // Playback 时 Entities[Index] 已被更新为真实 Entity
}
```

### 各操作的记录方式

**Set** — 记录组件值（SparseSet）：

```text
Sets.Set<T>(info.SetIndex, component)
  -> 确保 Components 中有 T 类型的 SparseArray
  -> 在该 SparseArray 中为 info.SetIndex 这一行写入 component 值
```

SparseArray 内部是一个 `T[]` 数组 + `int[] Entities` 索引表。通过 index 找到组件数组中的位置，直接写入值。

**Add** — 记录组件类型 + 初始值（StructuralSparseSet + SparseSet）：

```text
Adds.Set<T>(info.AddIndex)        // 标记 "要添加 T 类型"
Sets.Set(info.SetIndex, component) // 同时存储初始值（如果有）
```

StructuralSparseSet 只记录「这个 entity 要添加哪些类型」，不存值。值还是走 SparseSet。

**Remove** — 仅记录组件类型（StructuralSparseSet）：

```text
Removes.Set<T>(info.RemoveIndex)  // 标记 "要移除 T 类型"
```

**Destroy** — 仅记录 entity 在 buffer 内的索引：

```text
Destroys.Add(info.Index)
```

### Playback 回放顺序

```csharp
public void Playback(World world, bool dispose = true)
```

执行顺序是**严格有序**的：

```
1. Create  — 把 Creates 中的 entity 逐个 world.Create(types)，并更新 Entities 数组
2. Add     — 对每个 entity，收集它要添加的组件类型，调用 world.AddRange
3. Set     — 对每个 entity，遍历 SparseSet 中该 entity 的所有组件，Array.Copy 到 chunk
4. Remove  — 对每个 entity，收集它要移除的组件类型，调用 world.RemoveRange
5. Destroy — 对 Destroys 中的每个 entity 执行 world.Destroy
6. Clear   — 如果 dispose=true，清空所有 buffer
```

这个顺序保证了：先创建实体 → 再添加/设置组件 → 再移除组件 → 最后销毁。同一个 entity 如果在 buffer 中被 Destroy 又被 Add，顺序仍然保持在最后一步才销毁。

### Set 的具体过程

Set 是 Playback 中最复杂的步骤：

```text
遍历 Sets.Entities（每个 entity 一行）
  -> Resolve 负 ID 为真实 Entity
  -> 从 EntityInfoStorage 找到 entity 所在的 Archetype 和 Chunk
  -> 遍历 Sets.Used（所有涉及到的组件类型 ID）
      -> 如果该 entity 在这个组件类型的 SparseArray 中有数据
      -> Array.Copy(sparseArray[i], chunkArray, 1)  把值拷进 chunk
```

关键：Set 是直接 `Array.Copy` 到 chunk 的组件数组里，不走 World.Set 的完整路径（不移 archetype）。所以 Set 只能修改 entity 已有的组件，不能添加新类型。

### SparseSet vs StructuralSparseSet

| 特性 | SparseSet (`Sets`) | StructuralSparseSet (`Adds` / `Removes`) |
|------|--------------------|-------------------------------------------|
| 存什么 | 组件**值** | 组件**类型**（有没有这回事） |
| 内部数组 | `SparseArray`（含 `T[]` + `Entities` 索引） | `StructuralSparseArray`（仅 `Entities` 索引，无值数组） |
| 用途 | Set 操作的值暂存 | Add/Remove 的类型标记 |
| `Set<T>` 行为 | 写入 `T` 类型的组件值 | 仅标记该 index 关联了类型 T |

### 线程安全

所有公开方法都有 `lock (this)`，内部 SparseArray 操作也有自己的 `lock`。所以 **多线程可以向同一个 CommandBuffer 写入操作**。但 `Playback` 必须在主线程调用。

### 使用模式

```csharp
var commandBuffer = new CommandBuffer();

// 1. 操作已有 entity
commandBuffer.Set(entity, new Position { X = 10, Y = 10 });
commandBuffer.Add<Velocity>(entity);
commandBuffer.Remove<Health>(entity);
commandBuffer.Destroy(entity);

// 2. 创建新 entity
var placeholder = commandBuffer.Create([typeof(Position), typeof(Velocity)]);
commandBuffer.Set(placeholder, new Position { X = 5, Y = 5 });

// 3. 回放
commandBuffer.Playback(world);
```

也可以在 `Playback(world, dispose: false)` 后复用 buffer，但通常用 `using` 释放。


## 使用技巧

- 如果对同一个实体多次访问组件，可以先拿 `EntityData` 和 `Chunk`，避免每次重复从 `EntityInfoStorage` 查找。
- 热路径里尽量用泛型 API，例如 `Get<T>`、`Query<T0,T1>`，少走 `object` / `Type` 反射路径。
- 批量创建优先用 `Create(amount, ...)`，比循环单个 `Create` 更少扩容和查找。
- QueryDescription 可以缓存起来复用，不必每帧重复 new。
- 高性能遍历可以关注 InlineQuery / job 相关 API，减少 lambda/委托开销。

## 容易误解的点

- `ArrayPool<T>.Rent(n)` 返回的是「至少 n 长度」的数组，真实长度可能更大。
- `Chunks.Capacity` 是逻辑容量，不是 `ArrayPool` 返回数组的真实长度。
- `Chunk[]` 扩容复制是浅拷贝：Chunk struct 被复制，内部数组引用不会深拷贝。
- `Span<T>` 作为参数通常不用 `ref`；除非方法要修改调用者手里的 Span 变量本身。
- `Component.Id` 不是组件类型 id；组件类型 id 来自 `ComponentRegistry.Size`。
- `ArrayRegistry.Add<T>()` 当前主流程基本没接上；`ArrayRegistry.GetArray` 才是实际使用点。
- `QueryDescription.WithXXX` 当前每次都会 Build，链式调用会重复计算 hash，但功能正确。
