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

* 经常改变的元素少用 `Outline`、`Tiled Sprite`，这些组件会产生大量的顶点。
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

DrawCall 的合并是一个老生常谈的问题，在图形渲染基础的笔记中我也介绍了减少 DrawCall 的必要性。下面我将介绍一下 NGUI 和 UGUI 的合并规则，并以此介绍一些优化的手段。

> 合并 DrawCall 是为了减少那些多余的调用，并不是说 DrawCall 越少就越好。

### 合并规则

**NGUI**

NGUI 中具有 `Depth` 值，合并时就会以 `UIPanel` 为单位，对每个 Panel 下的 `UIWidget` 进行排序。若相邻的 UI 元素为同一个图集，那么就会合并为一个 DrawCall。举个例子，假设第 1-50 层的 UI 使用了图集 A，第 51 层使用了图集 B，第 52-100 使用了图集 A。根据规则，前 50 层会被合并为一个 DrawCall，而 52-100 层则会合并成另一个 DrawCall，再加上第 51 层总计为 3 个 DrawCall。

NGUI 中的 UI 元素的遮挡顺序与 z 坐标无关，主要由渲染顺序决定。对于某些 DrawCall 较高的界面，可以通过调整图集以及层级顺序，将界面的 DrawCall 降到一个很低的值。如果各位对于刚刚的内容存在一些疑惑，可以去看一下 `UIPanel.FillAllDrawCalls` 中的代码，里面包含了 NGUI 的合并逻辑。

**UGUI**

相比于可以看得到源码的 NGUI，UGUI 内部会自动对 DrawCall 进行调整。UGUI 中是以 `Canvas` 为单位进行合并，但它并没有像 NGUI 那样设计一个 `Depth` 控制 DrawCall 的合并，它的合并顺序主要取决于 Hierarchy 视图中的节点顺序，排在越下面的元素渲染后就会越靠前。下面将用一个简单的例子来说明 UGUI 的合并规则。

如下图所示，游戏中存在左右两个堆叠的界面，每种颜色都代表一种不同的图集，此时的 DrawCall 为 4。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/UI%E6%A8%A1%E5%9D%97%E4%BC%98%E5%8C%96%E6%8A%80%E5%B7%A7/01.png)

现在我们在右边的最底层再添加一个界面，此时的 DrawCall 为 9。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/UI%E6%A8%A1%E5%9D%97%E4%BC%98%E5%8C%96%E6%8A%80%E5%B7%A7/02.png)

如果我们让左边的最底层也添加一个相同的界面，那么此时的 DrawCall 为 5。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/UI%E6%A8%A1%E5%9D%97%E4%BC%98%E5%8C%96%E6%8A%80%E5%B7%A7/03.png)

相信各位会对第二张图产生疑问，明明我只是在右边加了一个界面，为什么会导致左右两边的 DrawCall 不进行合并呢？其实在 UGUI 中，如果一个 UI 元素没有遮挡任何的 UI，那么我们可以认为这个 UI 元素处于第 0 层（或者说最底层）；如果 UI 元素遮挡了其它的 UI，并且该 UI 使用了其它的图集无法进行合并，那么可以认为这个 UI 元素的层级是其底下的 UI 层级加一。UGUI 正是通过这样堆叠的方式来设置 UI 的渲染层级，相同层级且使用了同一图集的 UI 将会进行合并，因此你可以看到第一张图中的 DrawCall 为 4。

但问题是，如果我们像图二那样在某个界面底下又加了一层，那么就相当于把它上面所有的 UI 的层级都往上抬了一层，从而导致相同的 UI 元素不处于同一层而无法进行合并。为了证明这一点，我们可以如图三那样在左边的界面底部也加一个相同的界面，让左右两边的各个 UI 层级相等，这样就能够让 UGUI 顺利地进行 DrawCall 合并。

从这个简单的例子可以看出，UGUI 的合并规则具有很大限制，某些简单的 UI 可能会产生大量的 DrawCall。在 UGUI 中，进行重叠检测和分层合并是非常有必要的，并不是说划分出一个新的 `Canvas` 就会产生大量的 DrawCall（比如第二张图中对左右两部分分别建立 Canvas 不会产生新的 DrawCall）。

还需要注意的一点就是 UGUI 的 `Mask` 组件。该组件的实现方式是在模板缓冲中划出一片区域判断元素的显示与否，其本身就会占据一个 DrawCall。最关键的是，该组件还会导致原本可以的 DrawCall 被打断，造成性能浪费（比如往某个背包中的 Item 上加 Mask，那么这个 Item 就很可能无法与其他 Item 合并 DrawCall。

?> 单就 DrawCall 合并而言，NGUI 比 UGUI 具有更大的优势。

### 调试工具

NGUI 中可以使用其自带的 `DrawCall Tool` 进行调试优化，主要的方式就是调整 UI 元素的 `Depth` 以减少 DrawCall。相比之下，UGUI 就只能使用 `Frame Debugger` 进行查看，并且我们能获取到的信息非常的少（你并不知道某些元素为什么没有进行合并）。所以对于 UGUI 来说，就必须按照其合并的原理来对界面进行评估，其优化难度要比 NGUI 大得多。

### 对界面制作的影响

由于 UGUI 本身的合并规则存在很大的缺陷，所以它很有可能会影响到界面的制作。比如，UGUI 在做重叠判断时是按照 UI 的包围盒计算的（平行于 XY 轴），也就是说即使你的图片只是一条很长很细的斜线，它依旧会遮挡住大片的区域。除了重叠问题，如果你想在 UGUI 做动态的遮挡，那么也是比较难以实现的，因为 UGUI 的元素遮挡顺序就等同于 `Hierarchy` 中的顺序。另外，如果你的 UI 是 3D 的，那么在 3D UI 的旋转过程中难免又会产生重叠，从而导致 DrawCall 无法合并。

相比于令人头疼的 UGUI，在 NGUI 中你只需要手动调整 UI 元素的深度就可以把 DrawCall 降下来，远没有 UGUI 那么复杂。所以，单从 DrawCall 控制这一点来说，NGUI 要比 UGUI 好很多，特别是对于复杂界面的制作来说更是如此。

!> 对于 UGUI 而言，重叠是一个需要重视的问题，因为它是实实在在会影响 UI 性能的重要因素。

## 网格更新

---

在进行 UI 渲染前，会对网格进行更新重建。可以说，绝大部分的 UI 开销都是在这里。

### 更新机制

**NGUI**

在 NGUI 中，触发网格重建时分为两种方式，第一种是使用 `UIPanel.FillDrawCall` 更新单个 DrawCall，第二种是使用 `UIPanel.FillAllDrawCalls` 更新所有 DrawCall。由于一个 Panel 中会涉及到多个 DrawCall，一个 DrawCall 就对应了一个 Mesh。因此，我们要想办法让 NGUI 尽量调用第一种方法进行更新，否则将很容易让 UI 绘制出现峰值。

**UGUI**

使用 UGUI 需要分清楚**网格更新**与**网格重建**这两个概念。更新是指 UI 元素的某些属性发生变化需要重新绘制，比如修改 UI 颜色就是在修改顶点颜色，所以更新其实就是在修改顶点属性（UIVertex）。至于重建操作，指的是当 UI 元素发生变动时需要进行网格重建。

网格更新的标志性函数是 `Canvas.SendWillRenderCanvases`，即对 UI 元素进行重绘。在 UGUI 中，UI 元素的更新会被分成两种，第一种是 `RectTransform` 发生了变化（比如修改了 `Size`、`Anchor`、`Pivot` 会对 `UIVertext.position` 产生影响），第二种是渲染元素发生了变化（比如修改了 Image 和 Text 的颜色）。只是单纯的修改 UI 元素位置并不会引起网格更新。

网格重建的标志性函数就是 `Canvas.BuildBatch`，任意 Canvas 中的 UI 元素发生变动时都会引发整个 Canvas 的网格重建（哪怕只有一个元素改变了）。这里的变动指的是影响 UI 元素外观的改动，包括修改 `SpriteRenderer` 的图片、位置、缩放、文本等等。

总而言之，UGUI 以 Canvas 为单位进行网格更新和重建。任意 UI 元素的变动会引发重建，而 UI 元素的顶点属性变化还会引起更新。更新总是伴随着重建，所以它的开销较大。

在 Unity 5.2 版本之后，网格合并的操作更多的是放到了子线程中，因而我们还需要关注其他的几个方法：

* WaitingForJob
* PutGeometryJobFence
* BatchRenderer.Flush（开启多线程渲染之后）

### NGUI与UGUI如何进行网格合并

说了这么多，那么 UGUI 和 NGUI 在更新网格时有什么不同呢？请看下图：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/UI%E6%A8%A1%E5%9D%97%E4%BC%98%E5%8C%96%E6%8A%80%E5%B7%A7/04.png)

NGUI 在更新网格时是按照图集进行划分的，相同图集的元素会被合并到一个 Mesh 中。在上图中，血条和文本分别对应一个 DrawCall，在更新网格时并不会相互进行影响。

反观 UGUI，它的网格的划分是依据 Canvas 进行的，同一个 Canvas 下的元素会被划分进一个 Mesh，因此血条和文本被合并成了一个 DrawCall。不过这样一来，当某个元素发生改变时，该界面对应的 Mesh 就需要进行更新。

### 对界面制作的影响

了解到 NGUI 和 UGUI 的网格更新机制后，我们就需要在制作界面时注意以下几点：

* UGUI：拆分 Canvas。
* NGUI：控制 FillAllDrawCalls；拆分 UIPanel。

当你修改 UGUI 中某个 Canvas 的元素时，UGUI 会将整个 Canvas 的网格进行重建，因此一定要注意对 Canvas 的划分，否则某些简单但频繁地变动将影响整个界面的性能表现。虽说 Canvas 的增多将会导致 DrawCall 的增加，但对于复杂界面而言，划分 Canvas 有助于我们对界面进行显示/隐藏以及减少网格重建（比起多几个 DrawCall，网格重建的消耗要高得多）。至于如何进行 Canvas 的划分，可以参考下面的降低界面更新消耗的技巧。

在 NGUI 里，我们可以通过一些巧妙的方式控制 Mesh 的更新，将影响范围控制在修改的元素所在的 Mesh 中。当然，这种做法的容错率其实比较低，很容易就会引发整个 Panel 的重建，所以不管对于 UGUI 还是 NGUI 都需要先对 Canvas 和 UIPanel 进行拆分，以提高容错率。

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

UGUI 中首先要注意的一点就是让 UI 的 z 坐标为 0。如果 UI 的 z 坐标不为 0，那么 Unity 就会认为当前的 UI 具备深度，在合并 DrawCall 时就会从 Hierarchy 中寻找与当前节点相邻的节点，判断它们之间是否属于同一图集。如果相邻节点不属于同一图集，那么就无法进行合并。事实上，我们在制作 UI 元素时，其实并不需要调整 z 值来实现遮挡关系，因此我们理应保证所有的 UI 元素的 z 坐标都为 0。

另外一个比较常见的问题是在隐藏元素时用了错误的方法，比如把 Sprite 的图片设置为空、将 `color.a` 设置为 0 等等。有些用惯了 NGUI 的开发者可能习惯将元素移动到屏幕外，但这种做法在 UGUI 中是无法降低 DrawCall 的。

当然，我们在之前的内容中也提到了 UGUI 中最严重的一个问题：Hierarchy 节点的穿插和重叠。举个例子，如果我们要给背包界面的每个图标上添加一个红色的气泡提示，而红色气泡由于其位置的关系刚好又重叠在了旁边的图标上，那么此时就会产生大量无法合并的 DrawCall（具体的原因各位可以回顾一下上面讲过的 UGUI DrawCall 合并规则），比如下图这个看起来是 2 个 DrawCall 的界面可能会产生十几个 DrawCall。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/UI%E6%A8%A1%E5%9D%97%E4%BC%98%E5%8C%96%E6%8A%80%E5%B7%A7/05.png)

这个问题确实是非常令人头疼的，因为如果你的游戏需求就是要让气泡往旁边偏一点，那么你就没有办法避免图标的重叠。要解决这个问题，基本上就只能够调整图标与红点的节点层次，让图标和红点分别处于两个组别中，这样 UGUI 就可以对相邻节点进行合并。不过说是这么说，真正去做的话其实是非常麻烦的，因为你必须要想办法处理图标与红点的对应关系，所以最好还是缩小 Item 的尺寸避免重叠的发生。

UGUI 中还有一个与图集有关的隐蔽问题。在使用 UGUI 的图集合并时，会根据根据原始图片的压缩格式以及 Alpha 通道进行分类。如果你打入图集的图片的压缩格式不统一，或者一部分图片有 Alpha 通道而另一部分没有，那么它们在合并图集时会被划分进不同的图集中。也就是说，哪怕你给所有图片标记的 Tag 都一样，它们也很有可能会被分进四个不同的图集中。

**NGUI**

NGUI 造成 DrawCall 比较高的原因其实很少，也很好进行优化。第一个需要注意的点就是 UITexture，因为一个 UITexture 就会占据一个 DrawCall，除非是纹理相同并且 Depth 相邻时才会合并为一个 DrawCall。

除此之外，NGUI 造成 DrawCall 高的原因还有 Depth 的穿插以及 UI 隐藏方式错误的问题，这些都在之前的内容有提到过，这里就不再重复了。

## 降低界面更新开销

---

降低更新开销可以显著地提高游戏表现，并且还可以同时降低渲染开销。

### 动静分离

动静分离可以说是降低网格重建最有效的方式。还是以背包界面为例，现在的游戏需求是点击某个 Item 时，该 Item 上的图标会播放展示动画。一般来说，UGUI 滚动列表里面的 Item 都是处于同一个 Canvas 下的。如果我们不做任何修改，那么当 Item 播放展示动画时，整个 Canvas 就会发生网格重建，非常影响性能。

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

### 降低更新频率

这一点其实很好理解。比如我们在制作游戏中的地图雷达时，经常需要对玩家的位置信息进行更新，这种时候就可以根据具体的需求来选择优化的手段。对于 MMORPG 这种大型多人联机的游戏而言，玩家位置的信息其实没必要实时进行同步，你可以每隔一定时间更新一次，或者是当玩家移动的距离较短时不进行更新。

至于像其他更新频率很高的界面也可以使用这种技巧。以聊天界面为例，为了不让聊天列表进行频繁刷新，你可以在本地开启一个定时器，每隔一段时间刷新一次聊天消息。除此之外，你还可以为聊天消息列表设置一个数量上限，当消息过多时直接删除掉最旧的消息。当然，关于滚动列表的优化其实有很多可以探讨的内容，这里就不再展开讨论了，我会额外写一篇笔记介绍具体的优化方式。

### 避免敏感操作

**NGUI**

NGUI 中较为敏感的操作就是 `FillAllDrawCalls`，它会重新绘制整个界面，而不是针对某个 DrawCall 进行网格重建。造成这种情况出现的主要原因还是 UI 元素穿插，比如在添加/删除元素时穿插了其它的 DrawCall，或者是添加/删除的元素自成了一个 DrawCall。在 NGUI 中，如果在 Panel 中新加入了一个 DrawCall，那么就会导致整个界面的重建（这里主要是因为 NGUI 为了安全考虑，会直接重建所有 DrawCall，而不是针对某几个 DrawCall 进行重建）。最为典型的例子就是技能冷却，一般来说冷却的遮罩不会属于技能图标所在的图集，那么当技能冷却后需要显示遮罩时，就很有可能会引发上述现象。

为了避免这种情况发生，我们可以尝试让插入的元素能够合并到现有的 DrawCall 中，或者是预留位置，让元素进行单纯的显示和隐藏，而不是直接添加/删除元素。注意，这里的隐藏并不是用的 `color.a`，因为 NGUI 会直接把这类元素的面片去除掉。比较好的做法是用 `scale`，这样 NGUI 就不会把元素的 DrawCall 去除掉，其显示/隐藏时也只会更新它所在的 DrawCall。

?> 有人可能会觉得上述内容与之前讲的降低 NGUI DrawCall 的方式有一定的冲突。其实这主要是因为 `FillAllDrawCalls` 会导致整个界面的重建，我们应该首先避免界面更新产生的开销，然后再去讨论如何降低 NGUI 的 DrawCall。

**UGUI**

由于 UGUI 每次都是对整个界面进行重绘，不存在更新单个 DrawCall 的情况，所以我们就不需要关注上面提到的问题。UGUI 中的敏感操作是对 UI 元素的 `position` 赋值，因为每次赋值时 UGUI 并不会判断这个值有没有发生变动，它会直接调用 `Canvas.BuildBatch` 进行网格重建。为了避免无效赋值的情况，我们在每次改变 UI 元素的位置前都需要先判断数值是否发生了改变。

?> NGUI 中会对赋值是否发生变化进行判断。

### 其他优化选项

**NGUI**

UIPanel 上有两个选项会影响网格的更新，分别是 `static` 和 `Visible`。当你确定 UIPanel 下的元素不会发生位置移动时，那么你就可以勾选这个选项 `static`（当然适用范围并不大）。其次，如果你能够保证 UI 元素处于屏幕可见范围，那么你可以勾选 `Visible` 以避免重新计算其 Mesh 的包围盒（该选项主要针对大量网格更新）。
