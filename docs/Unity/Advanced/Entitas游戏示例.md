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

把 Jenny 窗口中的工程名改成你自己的工程名，然后点击 `Generate` 即可自动生成代码。

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

## Entitas自动生成代码

---

Jenny 窗口中有一个 `Contexts` 选项，如果你没有改默认设置，那么框架会自动创建 `Game` 和 `Input` 两个目录，它们包含以下文件：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Entitas%E6%B8%B8%E6%88%8F%E7%A4%BA%E4%BE%8B/03.png)

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Entitas%E6%B8%B8%E6%88%8F%E7%A4%BA%E4%BE%8B/04.png)

下面我来介绍一下这五种类型的文件是干什么的：

* Attribute：属性
* ComponentsLookUp：记录组件的总数、名称、类型
* Context：上下文，用于管理 Entity
* Entity：实体
* Matcher：匹配器，用于筛选实体

