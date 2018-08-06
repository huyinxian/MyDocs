# 脚本生命周期详解

说真的，没事多看看文档，有些内容文档里面写的是真的很全面：[生命周期](http://docs.unity3d.com/Manual/ExecutionOrder.html)。

下面这张图全面的描绘了脚本的整个生命周期：

![](http://obkyr9y96.bkt.clouddn.com/monobehaviour_flowchart.svg)

!> 注意，生命周期的每一阶段都是需要所有物体执行完毕后，才会进行下一步。例如当所有物体的 `Awake()` 执行完后，才会继续执行 `OnEnable()`。

## 初始化

---

这一部分主要是用于脚本的初始化。

### Awake

当脚本实例被载入时，`Awake()` 将会被调用，你可以在这里进行初始化操作。虽然每一个脚本的 `Awake()` 方法是随机顺序执行的，但你依然可以使用诸如 `FindWithTag()` 这样的方法来搜索对象。

### OnEnable

当对象变为激活状态时，就会调用这个方法。一般来说，假如你需要变动某个脚本的 `enabled` 属性，而你又希望脚本被激活后执行一些操作，那么你就可以使用 `OnEnable()` 方法。

!> `Awake()` 方法只会调用一次，如果你希望脚本被激活后执行某些操作，请使用 `OnEnable()`。另外，该方法不能够用于协同程序。

### Start

该方法总是在 `Awake()` 之后执行，并在 `Update()` 被调用前执行。同样的，`Start()` 也只会执行一次，不过由于 `Awake()` 已经执行过了，所以你可以放心地对其他的对象进行操作。

## 物理周期

---

这一部分用于处理物理相关的逻辑，循环的时间是固定的。

!> 虽然物理周期的循环时间是固定的，但如果游戏的帧数太低，那么物理周期可能会每帧执行两次以上。

### FixedUpdate

该方法就如同它的名字，更新的时间是固定的（默认为0.2s，可以修改），一般来说可以把物理相关的操作放在这里。但事实上，这仅仅是针对于帧数高的情况，如果游戏的帧间隔大于 0.2s，那么更新时间显然也不固定了。

### OnTriggerXXX

主要有这么三个方法：

```csharp
// 当触发开始时
void OnTriggerEnter(Collider other);

// 当碰撞体处在触发区域中时
void OnTriggerStay(Collider other);

// 当碰撞体离开触发区域时
void OnTriggerExit(Collider other);
```

?> 2D 模式下会有另外的调用方法。

### yield return WaitForFixedUpdate

