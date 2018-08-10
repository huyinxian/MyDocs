# Unity生命周期详解

说真的，没事多看看文档，有些内容文档里面写的是真的很全面：[生命周期](http://docs.unity3d.com/Manual/ExecutionOrder.html)。

下面这张图全面的描绘了Unity的整个生命周期：

![](http://obkyr9y96.bkt.clouddn.com/monobehaviour_flowchart.svg)

!> 注意，生命周期的每一阶段都是需要所有物体执行完毕后，才会进行下一步。例如当所有物体的 `Awake()` 执行完后，才会继续执行 `OnEnable()`。

## 初始化

---

这一部分主要是用于脚本的初始化。

### Awake

当脚本实例被载入时，`Awake()` 将会被调用，你可以在这里进行初始化操作。虽然每一个脚本的 `Awake()` 方法是随机顺序执行的，但你依然可以使用诸如 `FindWithTag()` 这样的方法来搜索对象。

!> `Awake()` 方法只会调用一次，如果你希望脚本被激活后执行某些操作，请使用 `OnEnable()`。

### OnEnable

当对象变为激活状态时（active = true），就会调用这个方法。一般来说，添加委托或事件可以放在 `OnEnable()`，取消委托或事件可以放在 `OnDisable()`。

!> `OnEnable()` 不能够用于协同程序。

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
// 当碰撞体与触发器开始接触时
void OnTriggerEnter(Collider other);

// 当碰撞体与触发器持续接触时，该方法每帧都会调用
void OnTriggerStay(Collider other);

// 当碰撞体脱离触发器时
void OnTriggerExit(Collider other);
```

?> 2D 模式下会有另外的调用方法。

### OnCollisionXXX

碰撞检测与触发检测其实是类似的：

```csharp
// 开始碰撞
void OnCollisionEnter(Collision collisionInfo);

// 碰撞持续发生
void OnCollisionStay(Collision collisionInfo);

// 结束碰撞
void OnCollisionExit(Collision collisionInfo);
```

?> 2D 模式下会有另外的调用方法

### yield WaitForFixedUpdate

如果你对于协程不了解，那么请先看协程那一章。`WaitForFixedUpdate` 表示它会等待 `FixedUpdate()` 执行后再运行。

## 输入事件

---

这一部分只有 `OnMouseXXX` 形式的方法，主要处理鼠标的输入事件。

## 游戏逻辑

---

游戏逻辑（Game Logic）是整个流程中最常用的部分，像是我们经常用的 `Update()` 也在这里面。

### Update

该方法每帧都会调用，可以用于处理游戏的主逻辑。不过需要注意的是，`Update()` 的调用频率是每帧一次，所以它的时间间隔会随着帧率的变化而变化。如果你不希望敌人的移动速度时快时慢，那么你可以使用 `Time.deltaTime` 来抵消这部分误差。

### yield null

`yield return null` 是用的比较多的。各位在学习的时候可能有一个误区，以为返回的值是用于等待几帧后执行。事实上，不管 yield 返回的值是多少，它都是等待一帧执行。

### yield WaitForSeconds

这个也是用的很多的一种，`yield return new WaitForSeconds(float time)` 的参数可以设置任意的数值，表示等待几秒之后再执行。不过请注意，不管你设置的值有多么小，哪怕设置的是 0，它也会在 `yield return null` 之后执行。

可能有人会有问，`WaitForSeconds` 真的能按时执行吗？这当然是不行的，如果你想要让某个方法等待 `0.02s` 后执行，那么具体的执行时间肯定是大于或等于 `0.02s`，至于其中的误差就是由当前的帧数所决定的。

说到底，协程终究是单线程，它不过是每帧检测运行条件，如果满足的话就执行，不满足就等到下一帧再检测。如果当前的帧间隔为 `0.02s`，哪怕你设置的值再小，至少也要等待 `0.02s` 后才会去执行剩余的代码。

### yield WWW

这个表示等待下载完成后再执行后续代码

### yield StartCoroutine

对于嵌套调用协程，需等待这个协程运行完毕后，才能执行后续代码。这个问题我在协程一章中有讲过。

### LateUpdate

`LateUpdate()` 一般可用于摄像机的移动。比如你在 `Update()` 中处理人物的移动，那么 `LateUpdate()` 中就可以进行相机的移动，保证视角是跟随人物的。

## 场景渲染

---

这一部分用于处理场景的渲染，我简单的介绍一下各个方法：

* **OnPreCull**：在相机剔除场景前调用。主要的作用是决定相机可以看到哪些物体，对于看不到的就进行剔除。
* **OnBecameVisible/OnBecameInvisible**：这两个方法分别是在对象变得可见/不可见时调用。
* **OnWillRenderObject**：如果对象是可见的，那么为每一个相机调用一次该方法。
* **OnPreRender**：在相机开始渲染场景前调用。
* **OnRenderObject**：在完成所有常规场景渲染后调用。可以使用 GL 类或者 `Graphics.DrawMeshNow` 来绘制自定义几何体。
* **OnPostRender**：在相机渲染完场景后调用。
* **OnRenderImage**：在场景渲染完毕后调用，用于对图像进行后处理。
* **OnGUI**：用于绘制 GUI，会在每帧进行擦除与重绘。一般用于测试。
* **OnDrawGizmos**：用于在场景视图中绘制 Gizmos。

## 一帧结束后的处理

---

当完整的一帧渲染完毕后，就会继续执行 `WaitForEndOfFrame` 的后续代码。这个也是属于用的比较多的一种。

## 游戏暂停

---

当检测到游戏暂停时，`OnApplicationPause()` 会在当前帧结束后被调用。

## 游戏结束

---

主要有三个方法：

* **OnApplicationQuit**：在退出游戏之前，将在所有的游戏对象上调用该方法。如果是在编辑器中，那么就是当用户点下停止按钮时调用。
* **OnDisable**：当对象变为未激活状态（active = false），就会调用这个方法。脚本被卸载时会调用 `OnDisable()`，而脚本载入时会调用 `OnEnable()`。
* **OnDestroy**：当继承自 MonoBehaviour 的类被销毁时，这个方法就会被调用。不过前提是游戏物体处于激活状态。