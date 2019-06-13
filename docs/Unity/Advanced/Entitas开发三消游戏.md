# Entitas开发三消游戏

本章将开发一个相对复杂些的游戏示例。项目代码的整体难度不大，基本看下注释都能够看懂，所以我只会讲解主要功能模块的设计思路。如果各位对于 ECS 的概念不是很了解，那么建议先看看我之前写的入门笔记。

游戏源码：[CandyCrushByECS](https://github.com/huyinxian/CandyCrushByECS)。这个 Demo 包含了之前的两个演示示例以及三消游戏，我在文件夹中已经用序号进行了区分。

## 功能性需求

---

我们先来看一下三消游戏需要哪些基础功能：

* 棋盘上的珠子生成
* 珠子按照一定的规则进行交换并消除
* 消除完毕后需要产生新的珠子下落以填补空位
* 珠子成链时如果符合某种形状，将会在消除之后生成一个特殊的珠子，该珠子具有特殊的消除效果

明确了游戏所应具备的功能后，接下来就是将这些功能拆分成一个个系统以及组件。

## 系统设计与实现

---

Entitas 框架中的主要逻辑都集中在各个系统中，所以如何规划各个系统的功能就成为了最主要的问题。各位在编写逻辑时，应尽量保证一个系统只专注于处理一小部分的组件（如果是类似于三消游戏这种小项目，通常一个系统只处理一种组件），这样能够降低系统之间的耦合性，提高程序的可复用性。

### 棋盘生成功能

棋盘的生成包括三部分，包括规定棋盘的尺寸、生成珠子、生成障碍物。由于这部分功能涉及到与 Unity 进行交互，因此这部分逻辑需要单独写成一个服务类（Service）以供 Entitas 进行调用。

?> 在编写 Entitas 内部代码时，尽量避免直接调用 Unity 的接口，应该使用组件来保存对象的索引或者调用中间服务类来执行逻辑。

### 点击功能

三消游戏的点击功能是指玩家拖动珠子的操作。商业项目为了实现良好的游戏效果，手势的拖拽逻辑通常是非常复杂的。当然，这里我们不做那么多考量，我们只需要实现点击交换和拖拽交换两种逻辑即可。

这一部分的逻辑对应的代码是 `ClickSystem` 和 `DragSystem`。如果你仔细看的话，你就会发现拖拽交换的底层调用的其实是点击交换，也就是说它们的表现效果是差不多的。各位若是想要实现那种能够用手势操控珠子移动的效果，那么你可以修改 DragSystem 的逻辑。

### 珠子移动效果

为了让珠子具备良好的移动以及消除效果，项目使用了 DoTween 作为缓冲动画插件。DoTween 动画统一提供了名为 `OnComplete` 的回调函数接口，当珠子的移动动画结束时，`MoveCompleteSystem` 将会捕捉到该实体对象并进行消除逻辑的判断。

### 珠子交换功能

珠子交换存在两种结果，第一种是符合交换规则，进行珠子消除；第二种是不符合交换规则，需要把这两个珠子的位置还原。这一部分的逻辑可以写成一个响应式系统，该系统主要监控那些进行了移动的珠子，判断它们移动之后是否符合横排或者竖排包含三个以上相同元素的情况。代码部分对应的是 `GetSameColorSystem`：

```csharp
private void GetSame(GameEntity entity, out List<IEntity> leftSameColorItems, out List<IEntity> rightSameColorItems,
out List<IEntity> upSameColorItems, out List<IEntity> downSameColorItems)
{
    string colorName = entity.gameLoadPrefab.path;
    CustomVector2 pos = entity.gameItemIndex.index;
    leftSameColorItems = new List<IEntity>();
    rightSameColorItems = new List<IEntity>();
    upSameColorItems = new List<IEntity>();
    downSameColorItems = new List<IEntity>();

    // 左
    for (int i = (int)pos.x - 1; i >= 0; i--)
    {
        if (!AddSameColorItem(colorName, i, pos.y, leftSameColorItems)) { break; }
    }

    // 右
    for (int i = (int)pos.x + 1; i < gameContext.gameBoard.columns; i++)
    {
        if (!AddSameColorItem(colorName, i, pos.y, rightSameColorItems)) { break; }
    }

    // 上
    for (int i = (int)pos.y + 1; i < gameContext.gameBoard.rows; i++)
    {
        if (!AddSameColorItem(colorName, pos.x, i, upSameColorItems)) { break; }
    }

    // 下
    for (int i = (int)pos.y - 1; i >= 0; i--)
    {
        if (!AddSameColorItem(colorName, pos.x, i, downSameColorItems)) { break; }
    }
}

private bool AddSameColorItem(string colorName, float x, float y, List<IEntity> sameColorItems)
{
    GameEntity entity;
    if (CheckSameColorItem(colorName, x, y, out entity))
    {
        sameColorItems.Add(entity);
        return true;
    }
    else
    {
        return false;
    }
}

private bool CheckSameColorItem(string colorName, float x, float y, out GameEntity entity)
{
    var set = gameContext.GetEntitiesWithGameItemIndex(new CustomVector2(x, y));
    entity = null;

    if (set.Count == 1)
    {
        entity = set.SingleEntity();
        if (!entity.isGameMovable) { return false; }

        return entity.gameLoadPrefab.path == colorName;
    }

    return false;
}
```

逻辑很简单，该系统会以当前的位置上的珠子为中心，计算出上下左右四个方向上与它相邻且颜色相同的珠子的个数。计算完毕后，`CheckSameColorSystem` 会检查当前珠子是否满足消除条件，并进行下一步处理：

```csharp
// 判断当前珠子是否满足横向或竖向上存在3个以上相同元素
// 满足则进行消除，不满足则将该珠子的位置进行还原
private bool HasCondition(GameEntity entity)
{
    int left = entity.gameSameColorItems.leftSameColorItems.Count;
    int right = entity.gameSameColorItems.rightSameColorItems.Count;
    int up = entity.gameSameColorItems.upSameColorItems.Count;
    int down = entity.gameSameColorItems.downSameColorItems.Count;

    return left + right >= 2 || up + down >= 2;
}
```

### 消除逻辑

为了提高游戏性，当相同颜色的珠子所形成的链满足一定的形状时，将会产生特殊的珠子。不同珠子的效果如下：

* 五个相同珠子排成一线进行消除时，所生成的特殊珠子将消除棋盘上所有与该珠子相同颜色的珠子。
* 横向和竖向上珠子个数超过三个时，所生成的特殊珠子将消除 3X3 范围内的所有珠子。
* 四个相同珠子排成一行/列进行消除时，所生成的特殊珠子将消除当前一行/列上的所有的珠子。

形状的判断逻辑对应 `CheckFormationSystem`。由于之前已经计算出了当前珠子在四个方向上相邻且颜色相同的珠子的个数，因此，代码也十分的简单：

```csharp
/// <summary>
/// 检测珠子成链的形状
/// </summary>
private bool CheckFormation(GameEntity entity, SpecialEliminateEffect effect)
{
    int left = entity.gameSameColorItems.leftSameColorItems.Count;
    int right = entity.gameSameColorItems.rightSameColorItems.Count;
    int up = entity.gameSameColorItems.upSameColorItems.Count;
    int down = entity.gameSameColorItems.downSameColorItems.Count;

    bool result;

    switch (effect)
    {
        case SpecialEliminateEffect.EliminateSameColor:
            result = left + right >= 4 || up + down >= 4;
            break;
        case SpecialEliminateEffect.EliminateHorizontal:
            result = left + right == 3;
            break;
        case SpecialEliminateEffect.EliminateVertical:
            result = up + down == 3;
            break;
        case SpecialEliminateEffect.Explode:
            result = left + right >= 2 && up + down >= 2;
            break;
        default:
            result = false;
            break;
    }

    return result;
}
```

判断出形状后，就需要交由不同的消除响应式系统来处理消除效果。上述四种特殊效果分别对应系统：`EliminateSameColorSystem`、`ExplodeSystem`、`EliminateHorizontalSystem`、`EliminateVerticalSystem`。

## Entitas框架常见问题

---

### 标志组件的更新问题

标志组件在值相同的情况下是不会响应的：

```csharp
public bool isGameEliminate {
    get { return HasComponent(GameComponentsLookup.GameEliminate); }
    set {
        // 如果值相同则下面的代码将不会执行
        if (value != isGameEliminate) {
            var index = GameComponentsLookup.GameEliminate;
            if (value) {
                var componentPool = GetComponentPool(index);
                var component = componentPool.Count > 0
                        ? componentPool.Pop()
                        : gameEliminateComponent;

                AddComponent(index, component);
            } else {
                RemoveComponent(index);
            }
        }
    }
}
```

### 监听事件的注册与系统的调用顺序

如果我们需要对某个组件进行监听，通常的声明方式如下：

```csharp
/// <summary>
/// 消除组件
/// </summary>
[Game, Event(EventTarget.Self)]
public class DestroyComponent : IComponent
{

}
```

框架会自动生成一个 `Events` 文件夹，该文件夹内会生成对应的 `Feature`、`System`、`Interface` 用来处理事件的监听和响应。如果需要使用对应的事件，那么只需要让类继承对应的接口并实现监听方法即可：

```csharp
public class View : MonoBehaviour, IGameDestroyListener
{
    // 实现监听方法
    public virtual void OnGameDestroy(GameEntity entity)
    {
        // 与实体解绑
        gameObject.Unlink();
    }
}
```

当然，如果你要让监听事件系统正常运作，你还需要执行该系统。它可以在 `Events` 文件夹下找到，通常是以 `XXXEventSystems` 进行命名：

```csharp
private Systems CreateSystems(Contexts contexts)
{
    // 注意，如果使用了Entitas的Event，那么框架会自动生成一个EventSystems，只有将它调用之后Event才会响应
    // 如果对于EventSystems内部的加载顺序有要求，那么就需要给对应组件的Event设置Priority
    return new Feature("Systems")
        .Add(new GameFeature(contexts))
        .Add(new GameEventSystems(contexts))
        .Add(new InputFeature(contexts));
}
```

`Event` 标签一共有三个参数，分别是事件的作用目标（自身或者所有监听了该事件的对象）、监听的事件类型（添加或者删除组件）、事件的执行优先级：

```csharp
/// <summary>
/// 棋盘元素的位置
/// 该事件的优先级要比LoadPrefab事件低，应该先实例化物体再改变其位置
/// </summary>
[Game, Event(EventTarget.Self, Entitas.CodeGeneration.Attributes.EventType.Added, 1)]
public class ItemIndexComponent : IComponent
{
    [EntityIndex]
    public CustomVector2 index;
}
```

由于系统的调用是有先后顺序的，如果你对监听事件系统的调用顺序有要求，那么你就需要设置对应组件的优先级，比如要等所有的初始化操作完毕后才能够改变元素的位置：

```csharp
public sealed class GameEventSystems : Feature {
    // 优先级数字越大，执行顺序越靠后
    public GameEventSystems(Contexts contexts) {
        Add(new GameDestroyEventSystem(contexts)); // priority: 0
        Add(new GameAnyLoadPrefabEventSystem(contexts)); // priority: 0
        Add(new GameLoadSpriteEventSystem(contexts)); // priority: 0
        Add(new GameItemIndexEventSystem(contexts)); // priority: 1
    }
}
```

!> 如果某个系统或者监听事件没有响应，那么你需要考虑系统的调用顺序是否存在问题。

### Group遍历问题

直接遍历 Group 会导致死循环，必须新建一个 List 来进行遍历。

### 响应式系统与接口的选择

响应式系统多用于处理框架内部逻辑，比如监听某个组件的修改。接口则多用于与 Unity 进行交互，因为 MonoBehavior 脚本可以继承相关的系统接口，进而作为一个 Entitas 与 Untiy 交互的中间层。

### 实体数组的访问方式

如果我们需要访问指定的实体，那么需要给组件加上 `EntityIndex` 标签，然后使用类似于 `GetEntitiesWithGameItemIndex(pos)` 的方法进行访问。要注意的是，该方法返回的是一个 `HashSet`，如果你确定你获取到的容器中有且只有一个实体，那么你可以使用框架的扩展方法 `SingleEntity` 来获取到对应的实体对象。

同样的，我们在编写响应式系统时，`Execute` 方法提供给我们的也是一个 `List` 容器。如果你能够确保该容器只包含一个对象，那么也可以用上述方法进行获取，否则还是建议各位使用如下方式进行安全访问：

```csharp
protected override void Execute(List<GameEntity> entities)
{
    foreach (var entity in entities)
    {
        // ...具体的逻辑
    }
}
```