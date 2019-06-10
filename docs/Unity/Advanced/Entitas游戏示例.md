# Entitas游戏示例

本篇笔记将介绍如何用 `Entitas` 框架来开发游戏。该框架不光能用于 Unity，你也可以把它拿来做其他的项目。之前我在 ECS 入门篇介绍了 Unity 官方的 ECS 框架，但由于官方框架仍处于试验阶段，因此本篇笔记选用了相对成熟一些的 Entitas 框架。

?> 本章的代码会和 Entitas 三消游戏示例放在一起，各位如果有需要地可以去看看。

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

System 有以下几种接口：

* IExecuteSystem：需要实现 `Execute()` 方法，可以放在 Update 中每帧执行一次。
* ICleanupSystem：需要实现 `Cleanup()` 方法，它会在所有 `Execute()` 执行完之后执行（相当于每帧的末尾）。
* IInitializeSystem：需要实现 `Initialize()` 方法，可以在 Start 中用于初始化。
* ITearDownSystem：可以用于场景/游戏结束后执行。

当实现完系统后，我们可以在一个 MonoBehavior 脚本中调用系统中的方法。如果你用的不是 Unity，那么就需要自己找一个合适的地方执行。一般来说，我们可以把一个持续执行的系统放到 Update 中，当然 FixedUpdate 和 LateUpdate 也可以，具体要看自己的选择。

### Reactive System

响应式系统使用 Collector 来收集特定的 Entity，只有当收集器收集到新的 Entity 时系统才会执行，否则将不作响应。

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

    // 过滤出需要被销毁的Entity
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

在上面的代码中，`GetTrigger` 方法返回了一个收集器，这个收集器会监测所有挂载了 `Destroyed` 组件的实体。我在之前的章节中有提到过，Group 有 `Added`、`Removed`、`AddedOrRemoved` 三个事件可以监测。如果你没有进行指定，那么收集器默认监测的是 `Added` 事件，也就是说当我们把一个 `Destroyed` 组件加到实体上时，这个实体就会被添加到 `Destroyed` 的 Group，并且会被收集器收集到响应式系统中然后触发 `Execute()` 方法。

你可能注意到响应式系统中有一个 `Filter` 方法，这个方法其实是用于过滤的。收集器有一个特性，如果一个实体被某个收集器收集了，那么即使将这个实体复原，它依然会被收集器所收纳。举个例子，我们想收集所有移除了 `Destroyed` 组件的实体，那么只要这个实体被收集了，哪怕你再次给实体挂上 `Destroyed` 组件，它依然会被收集器收集。为了避免这种情况，我们可以在 `Filter` 中进行过滤，检查它们是否包含 `Destroyed` 组件，如果包含的话就舍弃掉。

最后，响应式系统之所以叫做这个名字，是因为它只有在收集器收集到新的 Entity 时才会触发 `Execute()` 方法。如果响应式系统关注的 Group 没有发生变动，那么 `Execute()` 将不会被调用。

## Entitas自动生成代码

---

Jenny 窗口中有一个 `Contexts` 选项，如果你没有改默认设置，那么框架会自动创建 `Game` 和 `Input` 两个目录，它们包含以下文件：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Entitas%E6%B8%B8%E6%88%8F%E7%A4%BA%E4%BE%8B/03.png)

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Entitas%E6%B8%B8%E6%88%8F%E7%A4%BA%E4%BE%8B/04.png)

下面我来介绍一下这五种类型的文件是干什么的：

* Attribute：属性，也就是 C# 中的 Attribute 特性
* ComponentsLookUp：记录组件的总数、名称、类型
* Context：上下文，用于管理 Entity
* Entity：实体，一个上下文对应一种实体
* Matcher：匹配器，用于筛选实体

如果你对上面的文件仍有疑问，那么不要着急，我会在后面的示例中慢慢进行介绍。

## 用Entitas打印HelloWorld

---

按照编程惯例，我们先来打印一个 HelloWorld 试试手。

### 编写组件

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

我们一共有两个上下文：`Game`、`Input`，每当我们编写组件时需要为组件加上对应的 Attribute，以标记该组件属于哪一个 Context。在上面的代码中，`LogComponent` 属于 `Game` 上下文。

我们再次点击 `Generate` 自动生成代码后，`GameComponentsLookUp` 中就会多出以下代码：

```csharp
public static class GameComponentsLookup {

    public const int Log = 0;

    public const int TotalComponents = 1;

    public static readonly string[] componentNames = {
        "Log"
    };

    public static readonly System.Type[] componentTypes = {
        typeof(LogComponent)
    };
}
```

`GameComponentsLookup` 会给所有的组件编个号，Entity 会用该索引来创建对应的组件。`GameEntity` 的相关方法如下，大家可以对照着理解一下：

```csharp
public partial class GameEntity {

    public LogComponent log { get { return (LogComponent)GetComponent(GameComponentsLookup.Log); } }
    public bool hasLog { get { return HasComponent(GameComponentsLookup.Log); } }

    public void AddLog(string newMessage) {
        var index = GameComponentsLookup.Log;
        var component = (LogComponent)CreateComponent(index, typeof(LogComponent));
        component.message = newMessage;
        AddComponent(index, component);
    }

    public void ReplaceLog(string newMessage) {
        var index = GameComponentsLookup.Log;
        var component = (LogComponent)CreateComponent(index, typeof(LogComponent));
        component.message = newMessage;
        ReplaceComponent(index, component);
    }

    public void RemoveLog() {
        RemoveComponent(GameComponentsLookup.Log);
    }
}
```

当我们编写了一个新脚本时，要记得重新把代码生成一遍。这多多少少算是 Entitas 的缺陷之一，因为当程序出现报错时，你必须要先把错误解决了才能够进行生成（ToLua 等框架是一样的）。

### 编写系统

有了组件，我们还需要一个响应式系统来处理数据。继承 `ReactiveSystem` 时需要指定该系统监听的实体。这里我们监听的是 `Game` 上下文，因此需要填入 `GameEntity`。

```csharp
using System.Collections.Generic;
using UnityEngine;
using Entitas;
using System;

/// <summary>
/// 打印消息系统
/// </summary>
public class LogSystem : ReactiveSystem<GameEntity>
{
    public LogSystem(Contexts contexts) : base(contexts.game)
    {

    }

    protected override void Execute(List<GameEntity> entities)
    {
        foreach (GameEntity entity in entities)
        {
            Debug.Log(entity.log.message);
        }
    }

    /// <summary>
    /// 过滤器，用于过滤出系统感兴趣的实体
    /// </summary>
    protected override bool Filter(GameEntity entity)
    {
        // 将那些包含了LogComponent的实体过滤出来
        // 注意，编写一个新的Component后要记得重新Generate，因为hasLog方法是框架帮我们自动生成的
        return entity.hasLog;
    }

    /// <summary>
    /// 收集器
    /// </summary>
    protected override ICollector<GameEntity> GetTrigger(IContext<GameEntity> context)
    {
        return context.CreateCollector(GameMatcher.Log);
    }
}
```

接下来我们需要写一个用于初始化的系统，这个系统目前的作用是创建一个附带有 `HelloWorld` 消息的实体：

```csharp
using Entitas;

public class InitSystem : IInitializeSystem
{
    private readonly GameContext gameContext;

    public InitSystem(Contexts contexts)
    {
        gameContext = contexts.game;
    }

    public void Initialize()
    {
        // 创建一个实体
        gameContext.CreateEntity().AddLog("HelloWorld");
    }
}
```

### 编写Feature

系统写完了，之后我们要写一个用于管理所有 System 的 System（你可以把它看做是 SystemManager），它需要继承 `Feature`：

```csharp
public class AddGameSystems : Feature
{
    /// <summary>
    /// 将Game相关的系统添加到框架中
    /// </summary>
	public AddGameSystems(Contexts contexts) : base ("AddGameSystem")
    {
        Add(new LogSystem(contexts));
        Add(new InitSystem(contexts));
    }
}
```

顾名思义，`AddGameSystems` 的功能就是把与 Game 上下文有关的系统全部加入到框架中。如果你之后还写了其他的系统，那么也可以在这里面加上：

### 编写控制器运行程序

最后，我们需要创建一个控制器脚本，用于持续执行系统：

```csharp
using UnityEngine;
using Entitas;

public class GameController : MonoBehaviour
{
    private Systems systems;

    private void Start()
    {
        var contexts = Contexts.sharedInstance;
        systems = new Feature("Systems").Add(new AddGameSystems(contexts));
        // 初始化放在Start中，只执行一次
        systems.Initialize();
    }

    private void Update()
    {
        // 这两个方法是每帧调用
        // 首先执行所有的Excute方法，然后再执行所有的Cleanup方法
        systems.Execute();
        systems.Cleanup();
    }
}
```

之前在编写 System 时，我每次都会写一个构造方法，参数为 `Contexts`。Contexts 会定义所有的上下文，比如 `contexts.game` 就可以获取 Game 上下文。Contexts 使用了单例模式，你可以用 `Contexts.sharedInstance` 来获取。

### 小结

ECS 框架的使用步骤基本如下：

* 编写组件，并用代码生成器自动生成相关方法。
* 编写相关系统，包含构造方法、过滤器、收集器、执行方法。
* 编写系统管理器（Feature），用于系统的实例化。
* 编写控制器，调用 Feature，并每帧执行系统的相关方法。

HelloWorld 示例中我使用的是响应式系统，它所关注的事件是 `Added`，也就是说它只在新的实体加入时才会被触发。由于我只创建了一个实体，因此它只会打印一句 `HelloWorld`。

!> 注意，每次编写了新的组件后都要让框架重新生成一遍代码，否则你无法使用组件相关的 API。

## 使用Entitas与Unity的UI交互

---

上面我介绍了 Entitas 各个模块的基本用法，接下来就再提高点难度，做一个与 Unity 有更多交互的示例。要注意的是，为了让各个示例能够清晰地分隔开，最好给示例代码都加上相应的命名空间。ECS 在自动生成变量名时会把命名空间作为前缀，以区分同名模块。

### 基本功能演示

这个示例的功能很简单，首先点击左键时会在点击位置生成一张图片，点击右键时所有图片会旋转朝向点击位置并匀速移动至该位置。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Entitas%E6%B8%B8%E6%88%8F%E7%A4%BA%E4%BE%8B/06.png)

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Entitas%E6%B8%B8%E6%88%8F%E7%A4%BA%E4%BE%8B/07.png)

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Entitas%E6%B8%B8%E6%88%8F%E7%A4%BA%E4%BE%8B/08.png)

需要注意的是，鼠标的点击位置不能直接拿来做图片的位置，必须要进行屏幕坐标到世界坐标的转换。为了保证转换后位置是正确的，主相机的投影模式要改成正交，因为透视模式进行坐标转换得到的坐标永远是屏幕中心点（透视模式下相机的视野是一个视锥体，不明白的话可以自己试试）。

### 结构划分

由于篇幅原因，这里就不贴详细的代码了，只讲解一下主要的实现思路。该示例的功能并不复杂，因此我们依旧只需要 Game、Input 这两个上下文就可以了。

Game 上下文所需组件如下：

* MoveCompleteComponent：标志组件，用于标记物体是否允许移动。
* MoveComponent：记录目标位置。
* PositionComponent：记录物体的位置。
* SpriteComponent：记录图片的名称。
* ViewComponent：记录物体的 Transform 组件。

Input 上下文所需组件如下：

* MouseComponent：唯一组件，记录按钮以及按钮触发的事件。按钮分为左键和右键，事件分为按下和抬起。

组件介绍完了，接下来就需要编写对应的系统来处理实体了。示例所需系统如下：

* AddViewSystem：响应式，为实体添加 View 组件。
* CreaterSystem：响应式，当左键按下时创建一张图片。
* DirectionSystem：响应式，当有物体需要移动状态时改变该物体的朝向。
* MouseSystem：实现 IExecuteSystem 接口，当鼠标点击发生时修改 inputContext 中的 Mouse 组件。
* MoveSystem：响应式，当有物体需要移动状态时为其调用 DoTween 组件进行移动。
* PositionSystem：响应式，当实体的 Position 组件变动时修改物体的位置。
* RenderSpriteSystem：响应式，根据图片名称修改物体的图片。
* StartMoveSystem：响应式，当右键按下时为所有图片添加移动组件（进入移动状态）。

!> 向 Feature 中添加系统时一定要注意先后顺序，否则极有可能出现程序错误。

### 左键点击监听

由于我们需要监听鼠标的点击，因此 MouseSystem 不能写成响应式，而是应该每帧都进行调用。输入类的设备通常都是唯一的，例如鼠标、键盘等等，因此我们需要把 Mouse 组件写成单例组件。单例组件并不会挂载在某个实体上，所以如果要对单例组件进行修改，可以直接在对应的上下文中进行修改：

```csharp
public void Execute()
{
    if (Input.GetMouseButtonDown(0))
    {
        // 更改数据要用Replace，直接赋值无法让框架响应
        // inputContext.interactionExampleMouse.mouseButton = MouseButton.Left;
        inputContext.ReplaceInteractionExampleMouse(MouseButton.Left, MouseEvent.Down);
    }
    if (Input.GetMouseButtonDown(1))
    {
        inputContext.ReplaceInteractionExampleMouse(MouseButton.Right, MouseEvent.Down);
    }
}
```

当鼠标点击事件发生时（即 Mouse 组件发生了更改），各类响应式系统就会开始运作，比如创建一个新的实体，然后给实体挂上各种组件等等。具体的逻辑就不细讲了。

顺带一提，我们在修改组件的数据时要使用框架提供的 API，如果直接进行修改的话是无法让框架响应的。另外，`Replace` 同样也会给实体添加组件，因此会被收集器给收集起来。可以看一下它的代码：

```csharp
public void ReplaceInteractionExampleMouse(MouseButton newMouseButton, MouseEvent newMouseEvent) {
    var entity = interactionExampleMouseEntity;
    if (entity == null) {
        entity = SetInteractionExampleMouse(newMouseButton, newMouseEvent);
    } else {
        entity.ReplaceInteractionExampleMouse(newMouseButton, newMouseEvent);
    }
}
```

逻辑很简单，如果当前实体不存在，那么就新创建一个实体；如果实体存在，那么就更新实体的组件。内部的实现如下：

```csharp
public void AddInteractionExampleMouse(MouseButton newMouseButton, MouseEvent newMouseEvent) {
    var index = InputComponentsLookup.InteractionExampleMouse;
    var component = (InteractionExample.MouseComponent)CreateComponent(index, typeof(InteractionExample.MouseComponent));
    component.mouseButton = newMouseButton;
    component.mouseEvent = newMouseEvent;
    AddComponent(index, component);
}

public void ReplaceInteractionExampleMouse(MouseButton newMouseButton, MouseEvent newMouseEvent) {
    var index = InputComponentsLookup.InteractionExampleMouse;
    var component = (InteractionExample.MouseComponent)CreateComponent(index, typeof(InteractionExample.MouseComponent));
    component.mouseButton = newMouseButton;
    component.mouseEvent = newMouseEvent;
    ReplaceComponent(index, component);
}
```

可以看到，`Add` 和 `Replace` 其实都是新建了一个组件，它们的区别在于内部的实现逻辑是不一样的。

### 右键点击监听

当右键按下时，我们需要获取到当前已经存在的实体，然后为它们添加 Move 组件。这一操作同样也是一个讯号，诸如 MoveSystem、DirectionSystem、PositionSystem 等系统就会随之响应，让图片进行旋转并移动至目标点。

### 如何让Unity与Entitas相互链接

在上述示例中，如果我们想在 Entitas 中访问 Unity 的接口（比如 GameObject），那么通常是在 Entitas 中保存对象的索引。反过来，如果我们需要在 Unity 中使用 Entitas，那么就必须要通过某种方式获取到 Entity 的索引。

为了方便开发者实现上述需求，框架在 `Entitas.Unity` 的命名空间中提供了一个扩展方法，它可以将 GameObject 与 Entity 进行链接：

```csharp
using UnityEngine;

namespace Entitas.Unity
{
    public static class EntityLinkExtension
    {
        public static EntityLink GetEntityLink(this GameObject gameObject);
        public static EntityLink Link(this GameObject gameObject, IEntity entity);
        public static void Unlink(this GameObject gameObject);
    }
}
```

使用的话很简单：

```csharp
protected override void Execute(List<IMultiViewEntity> entities)
{
    foreach (IMultiViewEntity entity in entities)
    {
        string name = entity.contextInfo.name;
        GameObject go = new GameObject(name + "View");
        go.transform.SetParent(parentMap[name]);
        entity.AddMultiReactiveView(go.transform);
        // 让GameObject与Entity产生链接
        go.Link(entity);
    }
}
```

完成链接后，框架会生成一个叫做 `EntityLink` 的 MonoBehavior 脚本，Unity 部分的代码可以通过该脚本获取到它所链接的 Entity。如下图所示：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Entitas%E6%B8%B8%E6%88%8F%E7%A4%BA%E4%BE%8B/10.png)

## 多上下文响应系统

---

在之前演示的示例中，我们所编写的响应式系统全都只用于处理某一个上下文中的实体，比如 `GameEntity` 或者 `InputEntity`。但在某些时候，组件可能属于多个上下文（即公用组件），如果按照上面所讲的编写步骤可能就无法满足我们的需求。

下面是一个公用组件的示例，这两个组件从属于三种上下文：

```csharp
using UnityEngine;
using Entitas;
using Entitas.CodeGeneration.Attributes;

namespace MultiReactive
{
    /// <summary>
    /// 多上下文公用的销毁组件
    /// </summary>
    [Game, Input, UI]
    public class DestroyedComponent : IComponent
    {

    }

    [Game, Input, UI]
    public class ViewComponent : IComponent
    {
        public Transform transform;
    }
}
```

这种时候应该怎么办呢？没关系，Entitas 框架提供了相关接口供我们使用，你可以在 `Generated -> Components -> Interface` 下找到它们：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Entitas%E6%B8%B8%E6%88%8F%E7%A4%BA%E4%BE%8B/09.png)

当框架检测到某个组件属于多个上下文时，就会自动为其生成一个接口类。至于使用的话可以按照下面的方式：

```csharp
using System.Collections.Generic;
using UnityEngine;
using Entitas;
using MultiReactive;
using System;

namespace MultiReactive
{
    /// <summary>
    /// 通用销毁系统
    /// </summary>
    public class MultiDestroySystem : MultiReactiveSystem<IMultiDestroyEntity, Contexts>
    {
        public MultiDestroySystem(Contexts contexts) : base(contexts)
        {

        }

        protected override void Execute(List<IMultiDestroyEntity> entities)
        {
            foreach (IMultiDestroyEntity entity in entities)
            {
                if (entity.hasMultiReactiveView)
                {
                    // 如果实体拥有View组件，那么就需要进行额外的销毁操作
                    GameObject.Destroy(entity.multiReactiveView.transform.gameObject);
                }
                Debug.Log("Destroy in " + entity.contextInfo.name);
            }
        }

        protected override bool Filter(IMultiDestroyEntity entity)
        {
            return entity.isMultiReactiveDestroyed;
        }

        protected override ICollector[] GetTrigger(Contexts contexts)
        {
            return new ICollector[]
            {
                contexts.game.CreateCollector(GameMatcher.MultiReactiveDestroyed),
                contexts.input.CreateCollector(InputMatcher.MultiReactiveDestroyed),
                contexts.uI.CreateCollector(UIMatcher.MultiReactiveDestroyed)
            };
        }
    }

    // 声明一个特殊的Entity接口，系统内部处理的就是这个Entity
    public interface IMultiDestroyEntity : IEntity, IMultiReactiveDestroyedEntity, IMultiReactiveViewEntity { }
}

// 让对应的Entity继承接口
public partial class GameEntity : IMultiDestroyEntity { }
public partial class InputEntity : IMultiDestroyEntity { }
public partial class UIEntity : IMultiDestroyEntity { }
```

可以看到，我们在系统的内部声明一个特殊的 `IMultiDestroyEntity`，该接口继承了 `IEntity`, `IMultiReactiveDestroyedEntity`, `IMultiReactiveViewEntity`，因此它可以进行如下调用：

```csharp
entity.isMultiReactiveDestroyed;
entity.multiReactiveView;
```

值得注意的是，Entitas 中使用大量的 `partial` 关键字，这种分布式编程的结构能够让我们轻松地对原有的类进行扩展：

```csharp
// 让组件所处的所有上下文继承自定义的接口
// 这样GameEntity、InputEntity、UIEntity都能够使用Destroy和View这两个组件
public partial class GameEntity : IMultiDestroyEntity { }
public partial class InputEntity : IMultiDestroyEntity { }
public partial class UIEntity : IMultiDestroyEntity { }
```