# UI模块优化技巧

本篇笔记主要记录 NGUI 以及 UGUI 的优化技巧，以帮助各位在各自的项目优化 UI 模块，提高游戏性能。关于这些技巧的由来，我会在 UGUI 源码赏析一章中进行详细地解读。

> 笔记内容会持续更新与修改，记录更新更好的优化技巧。

## 元素更新方式

---

### UI元素如何更新

在 NGUI 中，有 `UIGeometry` 这样一个类：

```csharp
public class UIGeometry
{
	/// <summary>
	/// Widget's vertices (before they get transformed).
	/// </summary>

	public List<Vector3> verts = new List<Vector3>();

	/// <summary>
	/// Widget's texture coordinates for the geometry's vertices.
	/// </summary>

	public List<Vector2> uvs = new List<Vector2>();

	/// <summary>
	/// Array of colors for the geometry's vertices.
	/// </summary>

	public List<Color> cols = new List<Color>();

	// Relative-to-panel vertices, normal, and tangent
	List<Vector3> mRtpVerts = new List<Vector3>();
}
```

UGUI 中也是有类似的：

```csharp
public class VertexHelper : IDisposable
{
    private List<Vector3> m_Positions = ListPool<Vector3>.Get();
    private List<Color> m_Colors = ListPool<Color>.Get();
    // ...
}
```

当我们改变一个 UI 元素，这些 UI 元素最终是要转换成一个 `Mesh`，因此以上的这些数组都是代表 Mesh 上的顶点、颜色、纹理等信息。从底层的实现上来看，这两种框架的处理方式都是把 UI 元素转换成多组顶点属性，并最终转换成 Mesh。

由于我们的 UI 元素需要转换成数组，那么 UI 元素越大，其改动所产生的开销也就越大。也就是说，如果某些 UI 元素经常需要进行变动，那么我们就得想些办法控制这些 UI 元素的顶点数量，让它转换后产生的数组长度尽可能的短。

说完了 UI 元素的基础类，我们再来看看 NGUI 和 UGUI 是怎么对这些 UI 进行更新的。

NGUI 的做法是在 `UIPanel.LateUpdate` 调用 `UIPanel.UpdateWidgets` 进行轮询更新，循环其下所有的 UIWidget 检查其是否进行了更新。如果有，那么就需要进行网格更新，没有的话则直接返回。不过这样一来就会有一个问题，当 UI 元素过多时，轮询的方式会产生开销，从而导致性能问题。

UGUI 采用了另外一种做法。当我们修改 UI 元素时，它会把元素放到队列 `m_LayoutRebuildQueue` 或者 `m_GraphicRebuildQueue` 里，也就是说 UGUI 会根据 UI 元素是进行了布局变化或者渲染变化来分别处理（两种改变都有的话就需要都进行处理）。在渲染前的回调 `Canvas.SendWillRenderCanvas` 中，UGUI 会统一对这些元素进行处理。这种做法的好处在于，如果你的 UI 界面是没有发生变动的，那么 `SendWillRenderCanvas` 是没有开销的，它只会去遍历那些需要重新绘制网格的元素，所以 UGUI 在处理大量静态元素时会更具有优势。

### 缓存机制需要注意的问题

当然，上面写的这些内容不是在比较哪款框架更优秀，而是为了提醒各位在进行优化时需要注意一些点。我们在编写项目时，经常会缓存那些大量出现的元素，将暂时用不到的 UI 隐藏起来，等到下次需要时再进行使用。但问题是，`SetActive` 开销是很明显的，有些时候为了提高性能，我们不得不换一种做法来快速地显示/隐藏 UI 元素。

**UGUI**

对于 UGUI 而言，主要有以下两种：

* 将 `Scale` 设置为 0。
* 将 `CanvasGroup.alpha` 调整为 0。

**NGUI**

对于 NGUI，上述做法是不太合适的。由于 NGUI 是使用轮询的方式进行更新，当激活的 UI 元素越多，轮询的消耗也就越大。所以对于使用了 NGUI 的项目，如果缓存池中的元素过多，那么最好还是调用 `SetActive` 将 UI 设置为未激活状态。如果 UI 元素的数量适中，那么此时就可以用如下方式来操作：

* 将 `color.a` 设置为 0。
* 将 UI 移出屏幕外。

如果项目中使用的 UI 元素确实有点多，必须要进行 `SetActive`，那么可以考虑在当前的缓存池外再加上一个二级缓存池，并且规定这个缓存池中的 UI 元素不能够处于激活状态。当游戏从某个频繁修改元素的界面退出时（比如战斗界面），我们就可以把这个界面放入二级缓存池，并将其设置为未激活状态。这种做法的坏处就是在重新打开该界面时，会大量进行 `SetActive` 操作，但相比起轮询造成的大量消耗，这种做法要更好一些。除此之外，我们还可以为 UI 元素加上一个定时器，每隔一段时间将那些长久未使用的 UI 设置为未激活状态，又或者是限制一级缓存池的上限等等。做法其实有很多，需要各位根据实际情况进行处理。

!> 关于快速显示/隐藏，UGUI 与 NGUI 是不一样的。如果将这二者的做法搞混了，那么 NGUI 和 UGUI 依旧会绘制这些元素的网格，造成 DrawCall 的浪费。另外，有些项目可能习惯建一个隐藏的空节点，把需要隐藏的 UI 移动到空节点下，但问题是 `SetParent` 所带来的消耗也不小，最多只是代码写起来稍微方便点。

## DrawCall如何进行合并

---

DrawCall 的合并是一个老生常谈的问题，在图形渲染基础的笔记中我也介绍了减少 DrawCall 的必要性。下面我将介绍一下 NGUI 和 UGUI 的合并规则，并以此介绍一些优化的手段。

> 合并 DrawCall 是为了减少那些多余的调用，并不是说 DrawCall 越少就越好。

### 合并规则

**NGUI**

NGUI 中具有 `Depth` 值，合并时就会以 `UIPanel` 为单位，对每个 Panel 下的 `UIWidget` 进行排序。若相邻的 UI 元素为同一个图集，那么就会合并为一个 DrawCall。举个例子，假设第 1-50 层的 UI 使用了图集 A，第 51 层使用了图集 B，第 52-100 使用了图集 A。根据规则，前 50 层会被合并为一个 DrawCall，而 52-100 层则会合并成另一个 DrawCall，再加上第 51 层总计为 3 个 DrawCall。

NGUI 中的 UI 元素的遮挡顺序与 z 坐标无关，主要由渲染顺序决定。对于某些 DrawCall 较高的界面，可以通过调整图集以及层级顺序，将界面的 DrawCall 降到一个很低的值。如果各位对于刚刚的内容存在一些疑惑，可以去看一下 `UIPanel.FillAllDrawCalls` 中的代码，里面包含了 NGUI 的合并逻辑。

**UGUI**

相比于可以看得到源码的 NGUI，UGUI 内部会自动对 DrawCall 进行调整。UGUI 中是以 `Canvas` 为单位进行合并，但它并没有像 NGUI 那公开出一个 `Depth` 值用于控制 DrawCall 的合并，它的合并顺序主要取决 UGUI 计算出来的 Depth（这个过程是自动的），并且根据 Depth、材质 ID、纹理 ID、UI 层级（在 Hierarchy 视图中的顺序）排成一个列表。UGUI 最终会对这个渲染列表进行合批操作，**在列表中位置相邻且材质、Shader、纹理相同的 UI 即可进行合批**。

如下图所示，游戏中存在左右两个堆叠的界面，每种颜色都代表一种不同的图集，此时的 DrawCall 为 4。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/UI%E6%A8%A1%E5%9D%97%E4%BC%98%E5%8C%96%E6%8A%80%E5%B7%A7/01.png)

现在我们在右边的最底层再添加一个界面，此时的 DrawCall 为 9。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/UI%E6%A8%A1%E5%9D%97%E4%BC%98%E5%8C%96%E6%8A%80%E5%B7%A7/02.png)

如果我们让左边的最底层也添加一个相同的界面，那么此时的 DrawCall 为 5。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/UI%E6%A8%A1%E5%9D%97%E4%BC%98%E5%8C%96%E6%8A%80%E5%B7%A7/03.png)

相信各位会对第二张图产生疑问，明明我只是在右边加了一个界面，为什么会导致左右两边的 DrawCall 不进行合并呢？其实在 UGUI 中，如果一个 UI 元素没有遮挡任何的 UI，那么我们可以认为这个 UI 元素处于第 0 层（或者说最底层）；如果 UI 元素遮挡了其它的 UI，并且该 UI 使用了其它的图集无法进行合并，那么可以认为这个 UI 元素的层级是其底下最高的 UI 层级加一。UGUI 正是通过这样堆叠的方式来设置 UI 的渲染层级，相同层级且使用了同一图集的 UI 将会进行合并，因此你可以看到第一张图中的 DrawCall 为 4。

但问题是，如果我们像图二那样在某个界面底下又加了一层，那么就相当于把它上面所有的 UI 的层级都往上抬了一层，从而导致左右两边相同的 UI 元素不处于同一层而无法进行合并。为了证明这一点，我们可以如图三那样在左边的界面底部也加一个相同的界面，让左右两边的各个 UI 层级相等，这样就能够让 UGUI 顺利地进行 DrawCall 合并。

从这个简单的例子可以看出，UGUI 的合并规则具有很大限制，某些简单的 UI 可能会产生大量的 DrawCall。在 UGUI 中，进行**重叠检测**和**分层合并**是非常有必要的，并不是说划分出一个新的 `Canvas` 就会产生额外的 DrawCall（比如第二张图中对左右两部分分别建立 Canvas 不会产生新的 DrawCall）。

还需要注意的一点就是 UGUI 的 `Mask` 组件，它会占据两个 DrawCall。该组件的实现方式是在底层模板缓冲根据 Image 传进来的 Alpha 值来进行区域裁剪，并在子 UI 元素绘制完毕后结束掉裁剪的计算。这种做法会导致 Mask 下的 UI 进行单独的合批计算，使得原本看似可以进行合并的 DrawCall 被 Mask 强行打断。

总结来说，UGUI 的合并规则可以用下面这张图概括：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/UI%E6%A8%A1%E5%9D%97%E4%BC%98%E5%8C%96%E6%8A%80%E5%B7%A7/UGUI%E5%90%88%E6%89%B9%E8%A7%84%E5%88%99.png)

在 Depth 的计算方法中，由于需要对所有 UI 元素进行遍历并将它们与已经计算过 Depth 的元素进行相交判断，因此 UGUI 使用了分组计算包围盒矩形的方法加速了计算。具体的来说，就是以 16 个元素为一组计算 Group Rect，判断相交时首先需要判断当前元素是否与 Group 相交，然后才会去挨个与 Group 中的元素进行计算。所以，当 UI 元素过多且层级过于复杂时，Batch 速度会严重下降。

?> NGUI 的 DrawCall 之所以要比 UGUI 好控制，其实很大程度上取决于 Depth 的计算方式。NGUI 的 Depth 可以手动进行调节，因此很少存在 DrawCall 被打断的情况。相比而言，UGUI 的 Depth 是自动进行计算的，我们很难进行手动的调整。

### 调试工具

NGUI 中可以使用其自带的 `DrawCall Tool` 进行调试优化，主要的方式就是调整 UI 元素的 `Depth` 以减少 DrawCall。UGUI 可以用 `Profiler` 和 `Frame Debugger` 来进行查看：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/UI%E6%A8%A1%E5%9D%97%E4%BC%98%E5%8C%96%E6%8A%80%E5%B7%A7/06.png)

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/UI%E6%A8%A1%E5%9D%97%E4%BC%98%E5%8C%96%E6%8A%80%E5%B7%A7/07.png)

高版本 Unity 下的 Profiler 有一个 UI 模块的检测，可以清晰地查看有哪些 Mesh 生成了，并且还能看到合批被中断的原因。Frame Debugger 中则是可以查看 DrawCall 的渲染顺序，同时还能在游戏视图中看到每一步 DrawCall 的绘制过程。

### 对界面制作的影响

由于 UGUI 本身的合并规则存在很大的缺陷，所以它很有可能会影响到界面的制作。比如，UGUI 在做重叠判断时是按照 Mesh 的包围盒大小计算的（平行于 XY 轴），也就是说即使你的图片只是一条很长很细的斜线，它依旧会遮挡住大片的区域。除了重叠问题，如果你想在 UGUI 做动态的遮挡（比如点击某个 Item 可以放大遮住其它的 Item），那么也是比较难以实现的，因为 UGUI 的元素遮挡顺序就等同于 `Hierarchy` 中的顺序。另外，如果你的 UI 是 3D 的，那么在 3D UI 的旋转过程中难免又会产生重叠。这主要是由于 3D UI 在计算遮挡关系时是先投影到 2D 平面上然后再进行遮挡计算，因此在旋转时会产生大量的重叠，从而导致许多 DrawCall 无法合并。

相比于令人头疼的 UGUI，在 NGUI 中你只需要手动调整 UI 元素的深度就可以把 DrawCall 降下来，远没有 UGUI 那么复杂。所以，单从 DrawCall 控制这一点来说，NGUI 要比 UGUI 好很多，特别是对于复杂界面的制作来说更是如此。

!> 对于 UGUI 而言，重叠是一个需要重视的问题，因为它是实实在在会影响 UI 性能的重要因素。

## 网格重建

---

在进行 UI 渲染前，会对网格进行更新重建。相比起增加 DrawCall 带来的开销，网格的更新重建才是 UI 性能问题的关键点。

### NGUI与UGUI如何进行网格合并

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/UI%E6%A8%A1%E5%9D%97%E4%BC%98%E5%8C%96%E6%8A%80%E5%B7%A7/04.png)

NGUI 在更新网格时是按照图集进行划分的，相同图集的元素会被合并到一个 Mesh 中。在上图中，血条和文本分别对应一个 DrawCall，在更新网格时并不会相互进行影响。

反观 UGUI，它的网格的划分是依据 Canvas 进行的，同一个 Canvas 下的元素会被划分进一个 Mesh，因此血条和文本被合并成了一个 DrawCall。不过这样一来，当某个元素发生改变时，该界面对应的 Mesh 就需要进行更新。

### 重建机制

**NGUI**

在 NGUI 中，`UIPanel.LateUpdate` 的更新分为两种方式，第一种是使用 `UIPanel.FillDrawCall` 更新单个 DrawCall，第二种是使用 `UIPanel.FillAllDrawCalls` 更新所有 DrawCall。由于一个 Panel 中会涉及到多个 DrawCall，一个 DrawCall 就对应了一个 Mesh。因此，我们要想办法让 NGUI 尽量调用第一种方法进行更新，否则将很容易让 UI 绘制出现峰值。

**UGUI**

了解 UGUI 的网格重建需要理解 `Rebuild` 与 `Rebatch` 这两个过程。前者是指 UI 的 `Layout` 和 `Graphic` 组件发生改变时需要重新计算网格，后者则是指 Canvas 中的 UI 发生修改时需要重新绘制整个界面的网格，并发送给 GPU 进行渲染。

Rebatch 的标志性函数就是 `Canvas.BuildBatch`，计算 Batch 时需要按照深度进行排序，测试它们是否有重叠以及共同的材质等等。Canvas 会缓存上一次 Batch 的结果，直到 Canvas 发生修改时才会进行 Rebatch。

在 Unity 5.2 版本之后，网格合并的操作放到了子线程中，因而我们还需要关注其他的几个方法：

* WaitingForJob
* PutGeometryJobFence
* BatchRenderer.Flush（开启多线程渲染之后）

`Canvas.BuildBatch` 放到子线程后，我们一般来说是看不到这个方法带来的开销。但是请注意，这两者并不是完全的并行关系，Unity 需要等待合并返回的结果，所以当网格重建过于频繁时仍然会带来性能问题。

Rebuild 的标志性函数是 `Canvas.SendWillRenderCanvases`，即对 UI 元素进行更新。当 UI 发生修改时，该元素会被设置为 `Dirty`，并且会根据变化的类型将它放到 `m_LayoutRebuildQueue` 和 `m_GraphicRebuildQueue` 中。Rebuild 的过程是在 `CanvasUpdateRegistry` 类中执行的，这里的细节就不具体讲了，有兴趣的可以去看 UGUI 源码赏析一章。

LayoutRebuild 引发的更新主要是因为元素位置（或者布局）发生了改变，比如常用的 `HorizontalLayoutGroup`、节点层次结构发生改变（添加/删除等）等等。由于 Layout 每次都会计算其子元素的大小和位置，所以这类布局组件能少用就少用，或者自行编写一种布局组件。

GraphicRebuild 引发的更新主要是元素的本身产生了改变，例如大小、旋转、图片更改、文本变动等等。

最后在做一个小结：

* Rebuild 是当 Graphic 发生变化时，重新计算自身或者被它所影响的其它子节点的 Mesh。
* Rebatch 则是根据层级、遮挡关系、材质等因素，对修改过后的 Canvas 进行 DrawCall 合并，并送至 GPU 进行渲染。

?> 注意，Canvas 的网格是从 CanvasRender 组件获取的，如果存在嵌套 Canvas 的情况，那么子 Canvas 的网格并不会被包括进去，也就是说一次 Canvas 的 Batch 只会影响其子节点，不会影响其子 Canvas。反过来也是如此。

### 对界面制作的影响

了解到 NGUI 和 UGUI 的网格更新机制后，我们就需要在制作界面时注意以下几点：

* UGUI：拆分 Canvas；
* NGUI：控制 FillAllDrawCalls；拆分 UIPanel。

当你修改 UGUI 中某个 Canvas 的元素时，UGUI 会将整个 Canvas 的网格进行重建，因此一定要注意对 Canvas 的划分，否则某些简单但频繁地变动将影响整个界面的性能表现。虽说 Canvas 的增多将会导致 DrawCall 的增加，但对于复杂界面而言，划分 Canvas 有助于我们对界面进行显示/隐藏以及减少网格重建（比起多几个 DrawCall，网格重建的消耗要高得多）。至于如何进行 Canvas 的划分，可以参考后面的降低界面更新消耗的技巧。

在 NGUI 里，我们可以通过一些巧妙的方式控制 Mesh 的更新，将影响范围控制在修改的元素所在的 Mesh 中。当然，这种做法的容错率其实比较低，很容易就会引发整个 Panel 的重建，所以不管对于 UGUI 还是 NGUI 都需要先对 Canvas 和 UIPanel 进行拆分，以提高容错率。

## NGUI与UGUI的优劣比较

---

在介绍完两种主流框架的 DrawCall 控制以及网格更新机制后，我们可以做一个小结来对比两种框架的优势与缺点：

* 功能界面的 DrawCall 控制：NGUI > UGUI
* 功能界面的网格更新控制：NGUI > UGUI
* 动态 HUD 界面的网格更新控制：NGUI << UGUI
* 堆内存控制：NGUI << UGUI

对于前两点，由于 UGUI 的 DrawCall 合并规则有所缺陷，以及 UGUI 在更新网格时会直接选择将整个 Canvas 重建，因此 NGUI 略占优势。当然 UGUI 并不是一无是处。由于 NGUI 每次绘制时会轮询 Panel 下的所有处于激活的 Widget，所以 UGUI 在绘制大量静态元素时具有更明显的优势。另外，UGUI 的网格合并是在原生代码的 C++ 代码上执行的，在运行速度与堆内存控制上占据绝对优势。

## 降低渲染开销

---

降低 UI 的渲染开销主要就是降低 DrawCall，这一小节将以之前讲到的 DrawCall 合并规则为基础，提出一些优化方式。

### 分析渲染开销

在探讨如何降低开销之前，我们先来看看与渲染开销相关的一些函数。

在 Unity4.x 中，NGUI 渲染所调用的函数主要有 `Mesh.DrawVBO` 和 `Mesh.CreateVBO`，这两个函数都是属于 `Mesh.Renderer`，因此也包括进了一部分场景物体的渲染（不过一般来说不会很多，因为场景物体的网格不会像 UI 这样频繁进行变动）。UGUI 中同样使用的是上述的两个方法，不过这两个方法属于 `RenderForwardAlpha.Render`，因此基本上可以确认这个方法下的开销全都是由 UGUI 带来的。

如果版本处于 Unity5.3 以上，NGUI 的渲染函数基本没有变化，而 UGUI 则是将渲染放在了 `BatchRenderer.Add` 下的 `CanvasBatchIntermediateRenderer.RenderSubBatch` 中，渲染的耗时就体现得更为明确。

?> 对 UI 界面的分析可以帮助我们定位到问题的发生位置，从而找到解决的思路。

### 合并DrawCall

渲染开销的一个比较明显的指标就是 DrawCall，降低 DrawCall 的数量能够有效地提高游戏的表现性能。当然，并不是说 DrawCall 越低越好，而是应该让 DrawCall 处于一个较为合理的范围。

**UGUI**

UGUI 中首先要注意的一点就是让 UI 的 z 坐标为 0。如果 UI 的 z 坐标不为 0，那么 Unity 就会认为当前的 UI 具备深度。事实上，我们在制作 UI 元素时，其实并不需要调整 z 值来实现遮挡关系，因此我们理应保证所有的 UI 元素的 z 坐标都为 0。

另外一个比较常见的问题是在隐藏元素时用了错误的方法，比如把 Sprite 的图片设置为空、将 `color.a` 设置为 0 等等。有些用惯了 NGUI 的开发者可能习惯将元素移动到屏幕外，但这种做法在 UGUI 中是无法降低 DrawCall 的。

当然，我们在之前的内容中也提到了 UGUI 中最严重的一个问题：Hierarchy 节点的**穿插**和**重叠**。举个例子，如果我们要给背包界面的每个图标上添加一个红色的气泡提示，而红色气泡由于其位置的关系刚好又重叠在了旁边的图标上，那么此时就会产生大量无法合并的 DrawCall（具体的原因各位可以回顾一下上面讲过的 UGUI DrawCall 合并规则），比如下图这个看起来是 2 个 DrawCall 的界面可能会产生十几个 DrawCall。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/UI%E6%A8%A1%E5%9D%97%E4%BC%98%E5%8C%96%E6%8A%80%E5%B7%A7/05.png)

这个问题确实是非常令人头疼的，因为如果你的游戏需求就是要让气泡往旁边偏一点，那么你就没有办法避免图标的重叠。要解决这个问题，基本上就只能够调整图标与红点的节点层次以避免红点与图标的穿插，让图标和红点分别处于两个组别中。不过说是这么说，真正去做的话其实是非常麻烦的，因为项目一般的做法都是以 Item 为单位进行管理，强行将 Item 进行拆分只会带来管理上的不便，所以最好还是缩小 Item 的尺寸避免重叠的发生。

UGUI 中还有一个与图集有关的隐蔽问题。在使用 UGUI 的图集合并时，会根据根据原始图片的压缩格式以及 Alpha 通道进行分类。如果你打入图集的图片的压缩格式不统一，或者一部分图片有 Alpha 通道而另一部分没有，那么它们在合并图集时会被划分进不同的图集中。也就是说，哪怕你给所有图片标记的 Tag 都一样，它们也很有可能会被分进四个不同的图集中，造成图集不一致而无法合并 DrawCall。

**NGUI**

NGUI 造成 DrawCall 比较高的原因其实很少，也很好进行优化。第一个需要注意的点就是 UITexture，因为一个 UITexture 就会占据一个 DrawCall，除非是纹理相同并且 Depth 相邻时才会合并为一个 DrawCall。

除此之外，NGUI 造成 DrawCall 高的原因还有 Depth 的穿插以及 UI 隐藏方式错误的问题，这些都在之前的内容有提到过，这里就不再重复了。

### 减少OverDraw

所谓的 `OverDraw`，指的是半透明元素被多次进行绘制的问题。由于 Canvas 中的所有几何体都是在透明队列中进行绘制的，所以 UGUI 制作的界面始终伴随着透明度混合。由于关闭了深度测试和写入，所以当界面 UI 重叠时，就会出现同一个像素被多次绘制的问题。

显卡有一个叫做 `Fill Rate` 的参数，它代表了显卡每帧每秒能够绘制的像素数量。如果一帧当中有某个像素被多次进行了绘制，那么该像素占用的资源也就越多。

减少 `OverDraw` 的方法如下：

* 减少 UI 层叠。
* 遮挡场景时，关闭场景相机。
* 不要用一张不可见的 Image 来阻挡事件，可以自己写一个组件来进行事件遮挡。
* 如果 Image 使用了九宫切图，那么可以尝试取消勾选 `Fill Center`。

?> 在 Unity 的场景视图中可以切换至 `OverDraw` 选项来进行大致的评估，颜色越亮的地方代表多次绘制的问题越严重。

## 降低界面更新开销

---

降低更新开销可以显著地提高游戏表现，并且还可以同时降低渲染开销。

### 动静分离

由于单个元素的变动会引发整个界面的网格重建，因此动静分离可以说是降低网格重建最有效的方式。以 UGUI 为例，现在的游戏需求是点击某个 Item 时，该 Item 上的图标会播放展示动画。一般来说，UGUI 滚动列表里面的 Item 都是处于同一个 Canvas 下的。如果我们不做任何修改，那么当 Item 播放展示动画时，整个 Canvas 就会发生网格重建，非常影响性能。

比较好的解决方案是直接给选中的 Item 挂上一个新的 Canvas，然后在取消选中时销毁掉 Canvas。这样一来，动态元素就不会影响到整个 Canvas。

```csharp
public void DettachCanvas(bool isSelected)
{
	Canvas canvas = transform.FindChild("icon").gameObject.GetComponent<Canvas>();
	if (isSelected && canvas == null)
	{
		canvas = transform.FindChild("icon").gameObject.AddComponent<Canvas>();
		canvas.overrideSorting = true;
		canvas.sortingOrder = 10;
	}

	if (!isSelected) { Destroy(canvas); }
}
```

UGUI 的 Canvas 会对每次 Rebatch 的结果进行缓存，而如果 Canvas 中的某个 Graphic 进行更改导致自身 Mesh 的重建，那么就会引发整个 Canvas 的 Rebatch。所以，动静分离其实优化的是 Rebatch 这一部分。

当然，你可能会说加个 Canvas 或者 UIPanel 会增加 DrawCall 的量，但问题是 DrawCall 并不是越少越好，我们得看实际的开销。相比较于多了几个 DrawCall 带来的消耗，由 UI 元素的修改引发的 DrawCall 重建才是性能问题的关键。

### 降低更新频率

这一点其实很好理解。比如我们在制作游戏中的地图雷达时，经常需要对玩家的位置信息进行更新，这种时候就可以根据具体的需求来选择优化的手段。对于 MMORPG 这种大型多人联机的游戏而言，玩家位置的信息其实没必要实时进行同步，你可以每隔一定时间更新一次，或者是当玩家移动的距离较短时不进行更新。

至于像其他更新频率很高的界面也可以使用这种技巧。以聊天界面为例，为了不让聊天列表进行频繁刷新，你可以在本地开启一个定时器，每隔一段时间刷新一次聊天消息。除此之外，你还可以为聊天消息列表设置一个数量上限，当消息过多时直接删除掉最旧的消息。当然，关于滚动列表的优化其实有很多可以探讨的内容，这里就不再展开讨论了，我会额外写一篇笔记介绍具体的优化方式。

### 避免敏感操作

**NGUI**

NGUI 中较为敏感的操作就是 `FillAllDrawCalls`，它会重新绘制整个界面，而不是针对某个 DrawCall 进行网格重建。以技能界面为例，当我们点击某个技能时，通常都会显示一个技能冷却的遮罩图标，那么此时就极有可能会发生 `FillAllDrawCalls`。

事实上，我们在显示/隐藏遮罩时通常都是调用 `SetActive`，而遮罩图标一般都是单独放在一个通用的图集内，因此它在显示/隐藏时就产生 DrawCall 的增加/减少。对于 NGUI 来说，当添加/删除元素时穿插了其它的 DrawCall，或者是添加/删除的元素自成了一个 DrawCall，那么 NGUI 就会对整个界面进行重建，从而导致渲染开销的峰值。

为了避免这种情况发生，我们可以尝试让插入的元素能够合并到现有的 DrawCall 中，或者是预留位置让元素进行单纯的显示和隐藏，而不是直接添加/删除元素。注意，这里的隐藏并不是用的 `color.a`，因为 NGUI 会直接把这类元素的面片去除掉。比较好的做法是让 `scale` 等于 0，或者是让 `alpha` 接近于 0，这样 UI 元素看起来就像是隐藏了，但实际上 NGUI 还是会继续对它进行渲染。

?> 有人可能会觉得上述内容与之前讲的降低 NGUI DrawCall 的方式有一定的冲突。其实这主要是因为 `FillAllDrawCalls` 会导致整个界面的重建，我们应该首先避免界面更新产生的开销，然后再去讨论如何降低 NGUI 的 DrawCall。

**UGUI**

由于 UGUI 每次都是对整个界面进行重绘，不存在更新单个 DrawCall 的情况，所以我们就不需要关注上面提到的问题。UGUI 中的敏感操作是对 UI 元素的 `position` 赋值，因为每次赋值时 UGUI 并不会判断这个值有没有发生变动，它会直接调用 `Canvas.BuildBatch` 进行网格重建。为了避免无效赋值的情况，我们在每次改变 UI 元素的位置前都需要先判断数值是否发生了改变。

?> NGUI 中会对赋值是否发生变化进行判断。

### 其他优化操作

对于 UI 元素来说，如果它的顶点数越多，更改时产生的消耗也就会越大，所以我们应该尽量减少动态 UI 元素的顶点数。一般的做法有以下几种：

* 经常改变的元素少用 `Outline`，该组件的实现方式是将原本的网格复制多份，然后调整位置以实现描边效果（可以适当用阴影替代）。
* 避免使用 `Tiled Sprite`，该选项会产生大量顶点。
* 尽量减少动态的长文本，文字过多会产生许多顶点。

除开上述几点，在 NGUI 和 UGUI 中也有不同的优化选项。

**NGUI**

UIPanel 上有两个选项会影响网格的更新，分别是 `static` 和 `Visible`。当你确定 UIPanel 下的元素不会发生位置移动时，那么你就可以勾选这个选项 `static`（当然适用范围并不大）。其次，如果你能够保证 UI 元素处于屏幕可见范围，那么你可以勾选 `Visible` 以避免重新计算其 Mesh 的包围盒（该选项主要针对大量网格更新）。

**UGUI**

影响 UGUI 网格更新的因素有很多，我们可以一个个看：

* 尽量减少 UI 的数量和避免使用复杂的层级，这样做可以减少深度排序的时间。
* 慎用 UI 元素的 enable 和 disable，它们会触发 Rebuild。可以用其他的隐藏方式进行替代，比如 CanvasGroup、Scale。
* 尽量不要使用 Text 的 `Best Fit` 选项，它虽然可以自动调节字体大小从而避免超框，但 UGUI 会将该组件用到的所有字号全部都保存在 Atlas 中，导致占用空间变大。
* Canvas 如果开启了 `Pixel Perfect` 选项，那么当 UI 的位置发生变化时，会导致 Layout Rebuild。比如在 ScrollRect 滚动时，会导致 Canvas.SendWillRenderCanvas 的消耗变高。
* 对于不需要进行交互的 UI 组件，可以将它们的 `Raycast Target` 选项关掉，从而减轻点击事件带来的消耗。
* 如果打开的 UI 界面覆盖了全屏，那么可以把主相机关掉，只留一个 UI 相机。这样做的主要原因是 UI 界面没有参与游戏物体的深度剔除，所以哪怕游戏中的 UI 界面挡住了所有的物体，Unity 也依旧会对它们进行渲染。