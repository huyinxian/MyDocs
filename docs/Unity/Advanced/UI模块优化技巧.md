# UI模块优化技巧

本篇笔记主要记录 NGUI 以及 UGUI 的优化技巧，以帮助各位在各自的项目优化 UI 模块，提高游戏性能。

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

一般来说有下面几种常用做法：

* 经常改变的元素少用 `Outline`、`Tiled Sprite`。
* 尽量减少动态的长文本，文字过多会产生许多顶点。

说完了 UI 元素的基础类，我们再来看看 NGUI 和 UGUI 是怎么对这些 UI 进行更新的。

NGUI 的做法是在 `UIPanel.LateUpdate` 调用 `UIPanel.UpdateWidgets` 进行轮询更新，循环其下所有的 UIWidget 检查其是否进行了更新。如果有，那么就需要进行网格更新，没有的话则直接返回。不过这样一来就会有一个问题，当 UI 元素过多时，轮询的方式会产生开销，从而导致性能问题。

UGUI 采用了另外一种做法。当我们修改 UI 元素时，它会把元素放到队列 `m_LayoutRebuildQueue` 或者 `m_GraphicRebuildQueue` 里。在渲染前的回调 `Canvas.SendWillRenderCanvas` 中，UGUI 会统一对这些元素进行处理。这种做法的好处在于，如果你的 UI 界面是没有发生变动的，那么 `SendWillRenderCanvas` 是没有开销的，它只会去遍历那些需要重新绘制网格的元素。

### 缓存机制需要注意的问题

当然，上面写的这些内容不是在比较哪款框架更优秀，而是为了提醒各位在进行优化时需要注意一些点。我们在编写项目时，经常会缓存那些大量出现的元素，将暂时用不到的 UI 隐藏起来，等到下次需要时再进行使用。但问题是，`SetActive` 开销是很明显的，有些时候为了提高性能，我们不得不换一种做法来快速地显示/隐藏 UI 元素。

**UGUI**

对于 UGUI 而言，我们可以将 UI 元素的 `Scale` 设置为 0，或者把 `CanvasGroup.alpha` 调整为 0，让 UI 看起来像是消失了（但实际上还是处于激活状态的）。

**NGUI**

对于 NGUI，上述做法是不太合适的。由于 NGUI 是使用轮询的方式进行更新，当激活的 UI 元素越多，轮询的消耗也就越大。所以对于使用了 NGUI 的项目，如果缓存池中的元素过多，那么最好还是调用 `SetActive` 将 UI 设置为未激活状态。如果 UI 元素的数量适中，那么此时就可以调整 `color.a` 进行快速地显示与隐藏。

有的人可能会问，既然你说要减少 `SetActive` 的调用，那么当 UI 元素过多时，我应该在什么时候进行调用呢？其实，这个问题的答案需要靠我们自己思考，因为不同项目的需求是不一样的。举个例子，我们可以在当前的缓存池外再加上一个二级缓存池，并且规定这个缓存池中的 UI 元素不能够处于激活状态。当游戏从某个频繁修改元素的界面退出时（比如战斗界面），我们就可以把这个界面放入二级缓存池，并将其设置为未激活状态。这种做法的坏处就是在重新打开该界面时，会大量进行 `SetActive` 操作，但相比起轮询造成的大量消耗，这种做法要更好一些。除此之外，我们还可以为 UI 元素加上一个定时器，每隔一段时间将那些长久未使用的 UI 设置为未激活状态，又或者是限制一级缓存池的上限等等。做法其实有很多，需要各位根据实际情况进行处理。

!> 关于快速显示/隐藏，UGUI 与 NGUI 是不一样的。在 UGUI 中，可以调整 UI 的缩放度或者将 `CanvasGroup` 的透明度设置为 0，但在 NGUI 中只能是调整 `color.a`。如果将这二者的做法搞混了，那么 NGUI 和 UGUI 依旧会修改这些元素的网格，造成 DrawCall 的浪费。

## DrawCall合并

---

NGUI 和 UGUI 都有 DrawCall 合并，下面我将介绍一下这两种框架的合并规则，并以此介绍一些优化的手段。

### 合并规则

**NGUI**

NGUI 的合并规则非常简单，`UIPanel` 和 `UIWidget` 都具有 `Depth` 值，合并时就会以 `UIPanel` 为单位进行排序。若相邻的 UI 元素为同一个图集，那么就会合并为一个 DrawCall。举个例子，假设第 1-50 层的 UI 使用了图集 A，第 51 层使用了图集 B，第 52-100 使用了图集 A。根据规则，前 50 层会被合并为一个 DrawCall，而 52-100 层则会合并成另一个 DrawCall，再加上第 51 层总计为 3 个 DrawCall。

对于某些 DrawCall 较高的界面，可以通过调整图集以及层级顺序，将界面的 DrawCall 降到一个很低的值。

**UGUI**

相比于可以看得到源码的 NGUI，UGUI 内部会自动对 DrawCall 进行调整。下面将用一个简单的例子来说明 UGUI 的合并规则。

如下图所示，游戏中存在左右两个堆叠的界面，每种颜色都代表一种不同的图集，此时的 DrawCall 为 4。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/UI%E6%A8%A1%E5%9D%97%E4%BC%98%E5%8C%96%E6%8A%80%E5%B7%A7/01.png)

现在我们在右边的最底层再添加一个界面，此时的 DrawCall 为 9。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/UI%E6%A8%A1%E5%9D%97%E4%BC%98%E5%8C%96%E6%8A%80%E5%B7%A7/02.png)

如果我们让左边的最底层也添加一个相同的界面，那么此时的 DrawCall 为 5。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/UI%E6%A8%A1%E5%9D%97%E4%BC%98%E5%8C%96%E6%8A%80%E5%B7%A7/03.png)

相信各位会对第二张图产生疑问，明明我只是在右边加了一个界面，为什么会导致左右两边的 DrawCall 不进行合并呢？其实在 UGUI 中，如果一个 UI 元素没有遮挡任何的 UI，那么我们可以认为这个 UI 元素处于第 0 层（或者说最底层）；如果 UI 元素遮挡了其它的 UI，并且该 UI 使用了其它的图集无法进行合并，那么可以认为这个 UI 元素的层级是其底下的 UI 层级加一。UGUI 正是通过这样堆叠的方式来设置 UI 的渲染层级，相同层级且使用了同一图集的 UI 将会进行合并，因此你可以看到第一张图中的 DrawCall 为 4。

但问题是，如果我们像图二那样在某个界面底下又加了一层，那么就相当于把它上面所有的 UI 的层级都往上抬了一层，从而导致相同的 UI 元素不处于同一层而无法进行合并。为了证明这一点，我们可以如图三那样在左边的界面底部也加一个相同的界面，让左右两边的各个 UI 层级相等，这样就能够让 UGUI 顺利地进行 DrawCall 合并。

从这个简单的例子可以看出，UGUI 的合并规则具有很大限制，某些简单的 UI 可能会产生大量的 DrawCall。在 UGUI 中，进行重叠检测和分层合并是非常有必要的，并不是说划分出一个新的 `Canvas` 就会产生大量的 DrawCall（比如第二张图中对左右两部分分别建立 Canvas 不会产生新的 DrawCall）。

### 调试工具

NGUI 中可以使用其自带的 `DrawCall Tool` 进行调试优化，主要的方式就是调整 UI 元素的 `Depth` 以减少 DrawCall。相比之下，UGUI 就只能使用 `Frame Debugger` 进行查看，并且我们能获取到的信息非常的少（你并不知道某些元素为什么没有进行合并）。所以对于 UGUI 来说，就必须按照其合并的原理来对界面进行评估，其优化难度要比 NGUI 大得多。

### 对界面制作的影响

由于 UGUI 本身的合并规则存在很大的缺陷，所以它很有可能会影响到界面的制作。比如，UGUI 在做重叠判断时是按照 UI 的包围盒计算的（平行于 XY 轴），也就是说即使你的图片只是一条很长很细的斜线，它依旧会遮挡住大片的区域。除了重叠问题，如果你想在 UGUI 做动态的遮挡，那么也是比较难以实现的，因为 UGUI 的元素遮挡顺序就等同于 `Hierarchy` 中的顺序。另外，如果你的 UI 是 3D 的，那么在 3D UI 的旋转过程中难免又会产生重叠，从而导致 DrawCall 无法合并。

相比于令人头疼的 UGUI，在 NGUI 中你只需要手动调整 UI 元素的深度就可以把 DrawCall 降下来，远没有 UGUI 那么复杂。所以，单从 DrawCall 控制这一点来说，NGUI 要比 UGUI 好很多，特别是对于复杂界面的制作来说更是如此。

!> 对于 UGUI 而言，重叠是一个需要重视的问题，因为它是实实在在会影响 UI 性能的重要因素。