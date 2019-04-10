# Entitas游戏示例

本篇笔记将介绍如何用 Unity 最新出的 ECS 框架来制作游戏，以供各位了解 ECS 结构是如何运作的。

Unity 使用的 ECS 框架叫做 `Entitas`，这个东西不光能用于 Unity，你也可以把它拿来做其他的项目。

## Entitas搭建

---

Entitas 可以在 GitHub 上搜索同名开源项目下载（不要下源码），我使用的版本是 `v1.13`。

将项目解压到 Assets 目录下，菜单栏中会多出一个 `Tools` 项。按照 `Tools -> Jenny -> Preferences` 打开弹窗，然后点击 `Auto Import` 自动生成设置信息：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Entitas%E6%B8%B8%E6%88%8F%E7%A4%BA%E4%BE%8B/01.png)

当然，现在你还不能够进行生成，我们还得做点其他的准备。随便建一个脚本，然后用你的编译器打开它。现在再打开工程目录，找到工程名：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Entitas%E6%B8%B8%E6%88%8F%E7%A4%BA%E4%BE%8B/02.png)

把 Jenny 窗口中的工程名改成你自己的工程名，然后点击 `Generate` 即可自动生成代码。如果你想修改代码的输出目录，那么你可以在 `Target Directory` 中修改。

## Entitas中的基本概念

---

这部分的概念各位可以去看看官方示例，我这里只作简要的介绍。

### Component

组件是数据的基本表现，它可以是空的，也可以是包含多条属性，甚至可以是唯一的（单例）。组件需要继承 `IComponent` 接口，根据实现方式的不同可以分为以下几种。

**标志组件**

标志组件没有任何属性，一般用于对实体做标记，方便过滤。比如某些物体可以移动，那么你可以给它挂个空组件进行标记。

**数据组件**

数据组件中包含多个属性，用于存储数据：

```csharp
public sealed class PositionComponent : IComponent
{
    public int x;
    public int y;
}
```

**引用组件**

引用组件和数据组件其实差不多，不过引用组件包含的是复杂的引用对象：

```csharp
public sealed class ViewComponent : IComponent
{
    public GameObject gameObject;
}
```

引用组件中的属性会指向运行时创建的一些对象，它们并不是持久化数据，因此这类组件不代表数据。

**操作组件**

这种组件也是数据组件的变种，它所包含的属性是一个功能或者操作：

```csharp
public sealed class DelegateComponent : IComponent
{
    public Action action;
}
```

虽然这么写可以把函数、委托、操作存储在组件中，但这种做法会有很多缺点。

**唯一组件**

有些时候你希望某个组件是唯一的，那么你可以使用 C# 的 Attribute 将其标记为 `[Unique]`：

```csharp
[Unique]
public sealed class GameBoardComponent : IComponent
{
    public int columns;
    public int rows;
}
```

由于组件是无行为的，因此唯一组件更像是一个全局变量，并且可以被替换和移除。这种做法比起传统的单例模式要更好用。

### Entity

Entity 只是一个存储 Component 的容器，我们可以创建或者销毁 Entity。不过要注意的是，销毁并不是真正意义上的删除，而是把未使用的 Entity 放入了一个对象池中，以供后续重新启用。这种做法主要是为了减少 GC。

### Context

Context 用于创建、销毁、管理、过滤 Entity，你可以把它看做是工厂模式。为了减少 GC，Context 中内含一个 Entity 对象池，并使用引用计数进行管理。

### Group

上面我提到了 Context 可以管理 Entity，因此当我们需要某些 Entity 时，通常的做法是请求 Context 遍历所有的 Entity。

很显然，如果每次都这么做的话效率是极其低下的，我们需要 Context 为我们提供一个包含特定类型的 Group，这样一来就不必重复地对 Context 进行遍历。举个例子，我们需要查找所有包含 `Position` 和 `Velocity` 组件的 Entity，那么 Context 就会为我们提供这样一个 Group，并且组件的增添与删除都是实时更新的。Context 内部有一个 List 存储所有你请求过的 Group，方便我们重复访问。

?> Group 中的数据都是最新的，可以放心使用。

### Mathcer

既然说到了 Group，那么就不得不提一下 Matcher。Matcher 是一个匹配器，用于描述我们所感兴趣的实体。比如我们想要查找包含 `Position` 和 `Velocity` 组件的 Entity，那么我们可以这样进行描述：

```csharp
context.GetGroup(GameMatcher.AllOf(GameMatcher.Position, GameMatcher.Velocity));
```

这样一来，Context 就会返回对应类型的 Group，并将之记录下来。下次当你使用相同的 Matcher 来请求 Group 时，Context 会直接返回 List 中存储的 Group。

Matcher 的使用方法很简单，比如 `context.GetGroup(GameMatcher.Position)` 会返回所有包含 `Position` 组件的 Entity。如果你需要更复杂的用法，可以使用 `AllOf`、`AnyOf`、`NoneOf`。AllOf 表示其所有列出来的组件都必须包含在实体中，AnyOf 则只需要包含其中之一，而 NoneOf 则是不能够包含列出来的组件。

`NoneOf` 并不能够独立使用，比如 `context.GetGroup(GameMatcher.NoneOf(GameMatcher.Position))`。这样做可能会创建一个非常大的集合。NoneOf 只能够与 AllOf、AnyOf 结合使用：

```csharp
context.GetGroup(GameMatcher.AllOf(GameMatcher.Position, GameMatcher.Velocity).NoneOf(GameMatcher.NotMoveable));
```

这样我们就能够得到一个拥有 `Position`、`Velocity` 组件但不包含 `NotMoveable` 组件的 Entity Group。

### Collector

Collector 相当于 Group 的观察者，用于收集特殊的实体。

```csharp
context.CreateCollector(GameMatcher.Log.Removed());
```

在上面的代码中，我们创建了一个 Collector 用于收集所有删除了 `LogComponent` 组件的 Entity。在观察者模式中，观察者需要订阅它感兴趣的事件，那么 Group 中可以订阅的事件一共有三种：

* Added
* Removed
* AddedOrRemoved

Collector 主要用于 Reactive System，它的更多功能我将放到后面讲解。

### Index

如果我们想获取所有拥有 `Position` 组件的实体时，我们可以申请一个 Group。但假如你只想获取某个特定的实体时，就有必须要用 Index（索引）进行标记：

```csharp
using Entitas;
using Entitas.CodeGeneration.Attributes;

[Game]
public sealed class PositionComponent : IComponent
{
    [EntityIndex]
    public IntVector2 value;
}
```

要使用 Index，你首先得给对应的变量加上属性 `[EntityIndex]`，这个属性会告诉 ECS 代码生成器，让它在对应的 Context 中（上述代码对应的 Context 是 Game）创建一个能让用户根据 `IntVector2` 来获取 Entity 的 API。这类方法的代码通常如下，你可以自己在项目中查看：

```csharp
// 下面的代码是ECS自动生成的
foreach (var e in _contexts.game.GetEntitiesWithPosition(
                    new IntVector2(input.x, input.y)
                  ).Where(e => e.isInteractive)) {
    e.isDestroyed = true;
}
```

Index 的实现方式为哈希表，key 为 Index，value 为 Entity。内置的 Index 共有两种，`EntityIndex` 对应一组 Entity，而 `PrimaryEntityIndex` 只对应一个 Entity。

### System

System 是我们定义行为的地方，实现一个 System 需要我们继承多个接口。`ISystem` 是最基础的接口，它是空的，与 `IComponent` 类似，都是用于标记。

如果我们需要 System 持续执行，那么可以继承 `IExecuteSystem` 接口并实现 `Execute()` 方法；如果我们需要持续清理逻辑，那么可以继承 `ICleanupSystem` 接口并实现 `Cleanup()` 方法，该方法会在所有 `Execute()` 执行后再执行。

其它的接口也有，比如 `IInitializeSystem` 可以用于初始化，`ITearDownSystem` 可以用于场景/游戏结束后执行。我们可以根据需要继承对应的接口。

当实现完系统后，我们可以在一个 MonoBehavior 脚本中调用系统中的方法。如果你用的不是 Unity，那么就需要自己找一个合适的地方执行。一般来说，我们可以把一个持续执行的系统放到 Update 中，当然 FixedUpdate 和 LateUpdate 也可以，具体要看自己的选择。

### Reactive System

响应式系统只会在我们需要处理 Entity 时才会被调用，它会使用 Collector 来收集特定的 Entity。

```csharp
using System.Collections.Generic;
using Entitas;

public sealed class DestroySystem : ReactiveSystem<GameEntity>
{
    public DestroySystem(Contexts contexts) : base(contexts.game){}

    // 监测那些挂载了Destroyed组件的Entity
    protected override ICollector<GameEntity> GetTrigger(IContext<GameEntity> context)
    {
        return context.CreateCollector(GameMatcher.Destroyed);
    }

    // 筛选出需要被销毁的Entity
    protected override bool Filter(GameEntity entity)
    {
        return entity.isDestroyed;
    }

    protected override void Execute(List<GameEntity> entities)
    {
        // 将筛选完的Entity删除
        foreach (var e in entities)
        {
            e.Destroy();
        }
    }
}
```

在上面的代码中，`GetTrigger` 方法返回了一个收集器，这个收集器会监测所有挂载了 `Destroyed` 组件的实体。我在之前的章节中有提到过，Group 有 `Added`、`Removed`、`AddedOrRemoved` 三个事件可以监测。如果你没有进行指定，那么收集器默认监测的是 `Added` 事件，也就是说当我们把一个 `Destroyed` 组件加到实体上时，这个实体就会被添加到 `Destroyed` 的 Group 中，并且被对应的收集器收集到响应式系统中。

你可能注意到响应式系统中有一个 `Filter` 方法，这个方法其实是用于过滤的。收集器有一个特性，如果一个实体被某个收集器收集了，那么即使将这个实体复原，它依然会被收集器所收纳。举个例子，我们想收集所有移除了 `Destroyed` 组件的实体，那么只要这个实体被收集了，哪怕你再次给实体挂上 `Destroyed` 组件，它依然会被收集器收集。为了避免这种情况，我们可以在 `Filter` 中进行过滤，检查它们是否包含 `Destroyed` 组件，如果包含的话就舍弃掉。

最后，响应式系统之所以叫做这个名字，是因为它只有在收集器收集到新的 Entity 时才会触发 `Execute` 方法，否则该方法不会被调用。

## Entitas自动生成代码

---

Jenny 窗口中有一个 `Contexts` 选项，如果你没有改默认设置，那么框架会自动创建 `Game` 和 `Input` 两个目录，它们包含以下文件：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Entitas%E6%B8%B8%E6%88%8F%E7%A4%BA%E4%BE%8B/03.png)

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Entitas%E6%B8%B8%E6%88%8F%E7%A4%BA%E4%BE%8B/04.png)

下面我来介绍一下这五种类型的文件是干什么的：

* Attribute：属性，也就是 C# 中的 Attribute 特性
* ComponentsLookUp：记录组件的总数、名称、类型
* Context：上下文，用于管理 Entity
* Entity：实体
* Matcher：匹配器，用于筛选实体

如果你对上面的文件仍有疑问，那么不要着急，我会在后面的示例中慢慢进行介绍。按照编程惯例，我们先来打印一个 HelloWorld 试试手。

首先创建一个 `LogComponent` 脚本：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Entitas%E6%B8%B8%E6%88%8F%E7%A4%BA%E4%BE%8B/05.png)

然后编写以下代码：

```csharp
using Entitas;

/// <summary>
/// 打印消息的组件
/// </summary>
[Game]
public class LogComponent : IComponent
{
    /// <summary>
    /// 需要打印的信息
    /// </summary>
    public string message;
}
```

有了组件，我们还需要一个系统来处理数据：

