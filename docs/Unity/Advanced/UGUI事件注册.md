# UGUI事件注册

当我们创建 UGUI 的组建时，Unity 会自动生成一个 `EventSystem`。这个组件主要是用来管理 UGUI 的事件系统，我会专门写一篇笔记来介绍它。

## 常见的UI注册方式

---

对于一个 Button 组件，初学者添加事件通常会在 Button 组件的属性面板中找到 `OnClick()` 属性，然后拖拽对应的游戏物体并选择脚本中对应的回调事件。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/UI%E4%BA%8B%E4%BB%B6%E6%B3%A8%E5%86%8C/01.png)

这种做法看似简单，但当 Button 数量过多时会影响 UI 事件的管理，而且很容易操作出错。比较好的做法是把上述操作用代码实现出来：

```
button.onClick.AddListener(TestCallback);
```

### UI事件封装

其实如果你有用过 NGUI，那么你应该接触过 `UIEventListener`。NGUI 将常用的事件都封装了起来，使用时可以非常简单地注册各类事件。那么，UGUI 能不能也封装一下？当然可以，下面给各位看一下[宣雨松](http://www.xuanyusong.com/archives/3325)的实现：

```csharp
using UnityEngine;
using System.Collections;
using UnityEngine.EventSystems;
public class EventTriggerListener : UnityEngine.EventSystems.EventTrigger{
	public delegate void VoidDelegate (GameObject go);
	public VoidDelegate onClick;
	public VoidDelegate onDown;
	public VoidDelegate onEnter;
	public VoidDelegate onExit;
	public VoidDelegate onUp;
	public VoidDelegate onSelect;
	public VoidDelegate onUpdateSelect;
 
	static public EventTriggerListener Get (GameObject go)
	{
		EventTriggerListener listener = go.GetComponent<EventTriggerListener>();
		if (listener == null) listener = go.AddComponent<EventTriggerListener>();
		return listener;
	}
	public override void OnPointerClick(PointerEventData eventData)
    {
        if (onClick != null) onClick(gameObject);
    }
    public override void OnPointerDown(PointerEventData eventData)
    {
        if (onDown != null) onDown(gameObject);
    }
    public override void OnPointerEnter(PointerEventData eventData)
    {
        if (onEnter != null) onEnter(gameObject);
    }
    public override void OnPointerExit(PointerEventData eventData)
    {
        if (onExit != null) onExit(gameObject);
    }
    public override void OnPointerUp(PointerEventData eventData)
    {
        if (onUp != null) onUp(gameObject);
    }
    public override void OnSelect(BaseEventData eventData)
    {
        if (onSelect != null) onSelect(gameObject);
    }
    public override void OnUpdateSelected(BaseEventData eventData)
    {
        if (onUpdateSelect != null) onUpdateSelect(gameObject);
    }
}
```

使用的话很简单：

```csharp
using UnityEngine;
using System.Collections;
using UnityEngine.UI;
using UnityEngine.EventSystems;
using UnityEngine.Events;
public class UIMain : MonoBehaviour {
	Button	button;
	Image image;
	void Start () 
	{
		button = transform.Find("Button").GetComponent<Button>();
		image = transform.Find("Image").GetComponent<Image>();
		EventTriggerListener.Get(button.gameObject).onClick = OnButtonClick;
		EventTriggerListener.Get(image.gameObject).onClick = OnButtonClick;
	}
 
	private void OnButtonClick(GameObject go){
		//在这里监听按钮的点击事件
		if(go == button.gameObject){
			Debug.Log ("DoSomeThings");
		}
	}
}
```

### EventTrigger中的坑

这样看起来是不是就很完美了？不，如果你真正去用的话就会发现上面这么写是不对的。我们先来看一下 `EventTrigger` 到底是个什么东西：

```csharp
public class EventTrigger : MonoBehaviour, IPointerEnterHandler, IPointerExitHandler, IPointerDownHandler, IPointerUpHandler, IPointerClickHandler, IInitializePotentialDragHandler, IBeginDragHandler, IDragHandler, IEndDragHandler, IDropHandler, IScrollHandler, IUpdateSelectedHandler, ISelectHandler, IDeselectHandler, IMoveHandler, ISubmitHandler, ICancelHandler, IEventSystemHandler
{
    // 这里的都是从元数据，没有源代码，看不到具体实现
    public virtual void OnBeginDrag(PointerEventData eventData);
    public virtual void OnCancel(BaseEventData eventData);
    public virtual void OnDeselect(BaseEventData eventData);
    public virtual void OnDrag(PointerEventData eventData);
    public virtual void OnDrop(PointerEventData eventData);
    public virtual void OnEndDrag(PointerEventData eventData);
    public virtual void OnInitializePotentialDrag(PointerEventData eventData);
    public virtual void OnMove(AxisEventData eventData);
    public virtual void OnPointerClick(PointerEventData eventData);
    public virtual void OnPointerDown(PointerEventData eventData);
    public virtual void OnPointerEnter(PointerEventData eventData);
    public virtual void OnPointerExit(PointerEventData eventData);
    public virtual void OnPointerUp(PointerEventData eventData);
    public virtual void OnScroll(PointerEventData eventData);
    public virtual void OnSelect(BaseEventData eventData);
    public virtual void OnSubmit(BaseEventData eventData);
    public virtual void OnUpdateSelected(BaseEventData eventData);
    // 具体部分省略...
}
```

EventTrigger 继承了一堆接口，并以虚方法的形式对它们进行了实现。这些接口能够通过命名来看出其功能，比如点击按下、点击抬起、开始拖拽、结束拖拽等等。如果我们按照上面的那种方式注册事件，那么就会在注册事件时给 UI 添加 EventTriggerListener。由于 EventTriggerListener 继承自 EventTrigger，而 EventTrigger 实现了所有的事件接口，那么就意味着这个 UI 会接收所有的事件（点击、拖拽等等）。

这样会有什么问题呢？假设你在做一个背包界面，背包中有许多可以点击的格子（Button），并且界面可以上下滑动（ScrollRect）。如果你用 EventTriggerListener 为按钮注册事件，那么你会惊奇地发现背包界面无法拖动了！

不，准确的来说是可以拖动的，只不过你必须要在没有格子的地方划动。这种情况就是常见的点击事件与拖拽事件冲突。由于上面这种注册方式导致按钮可以接受所有的事件，而背包界面中的按钮又处于 ScrollRect 之上，那么也就意味着 ScrollRect 无法接收到拖拽事件。一般来说，按钮只需要接受点击事件即可，我们并不需要它接受拖拽事件（除非游戏有需求）。

### 一种治标不治本的解决思路

那么应该怎么解决这个问题呢？第一种做法，再编写一个组件，然后给每个按钮挂上：

```csharp
public class UIDragScrollRect : MonoBehaviour, IBeginDragHandler, IDragHandler, IEndDragHandler
{
    public ScrollRect scrollRect;

    public void OnBeginDrag(PointerEventData eventData)
    {
        if (scrollRect != null) scrollRect.OnBeginDrag(eventData);
    }
    public void OnDrag(PointerEventData eventData)
    {
        if (scrollRect != null) scrollRect.OnDrag(eventData);
    }
    public void OnEndDrag(PointerEventData eventData)
    {
        if (scrollRect != null) scrollRect.OnEndDrag(eventData);
    }
}
```

不光是按钮，只要是出现了某个 UI 遮挡了拖拽的情况，你都可以把这个脚本挂在对应的控件上。如果给发生冲突的按钮挂上了这个脚本，那么当按钮接收到拖拽事件时，就会调用 ScrollRect 的拖拽方法，从而解决了事件冲突。

如果这样能够解决你的问题，那么这篇笔记就到此为止了。我之所以要往后写，是因为这种做法只是治标不治本。举个例子，假如你现在要给背包界面中的 ScrollRect 添加拖拽事件，那么你会发现 ScrollRect 虽然能够拖拽，但是它并不会对事件作出响应。原因很简单，我们上面的做法只是让按钮在接受点击时调用 ScrollRect 的拖拽事件，但是我们为 ScrollRect 添加的事件是无法触发的，除非你能让 ScrollRect 直接接收到事件（比如点在没有按钮的地方）。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/UI%E4%BA%8B%E4%BB%B6%E6%B3%A8%E5%86%8C/02.png)

如上图，如果你在有按钮的位置进行拖拽，虽然可以拖动 ScrollRect，但是只能够响应 Click 事件。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/UI%E4%BA%8B%E4%BB%B6%E6%B3%A8%E5%86%8C/03.png)

上图中，我们在没有按钮的位置进行了拖拽，所以可以响应 Drag 事件。

如果你不需要为 ScrollRect 编写 OnDrag、OnBeginDrag、OnEndDrag 等方法，那么你可以尝试使用这种思路来解决，不过这种做法终归是有隐患的。

### 分离式做法

其实我们没必要把所有的事件都封装在一起，比如一个按钮，它常用的也就是按下、抬起、开始点击、点击、结束点击等事件。我们可以把这些接口分开进行封装。首先是点击型 UI：

```csharp
using UnityEngine;
using System.Collections;
using UnityEngine.EventSystems;

/// <summary>
/// 注册点击型UI的事件
/// </summary>
public class UIPointerEvent : MonoBehaviour,
    IPointerClickHandler,
    IPointerDownHandler,
    IPointerEnterHandler,
    IPointerExitHandler,
    IPointerUpHandler,
    ISelectHandler,
    IDeselectHandler,
    IUpdateSelectedHandler
{
    // 注意，如果直接继承EventTrigger，由于EventTrigger实现了所有接口，会导致其接管所有事件，进而引起UI事件屏蔽。正确的做法是只继承对应的接口
    // 举例，在制作背包界面时，Button在ScrollRect的上面，那么EventTrigger就会让Button把OnClick和OnDrag全部接受
    // 由于Button没有处理OnDrag，因此导致背包界面只能够点击而不能够拖拽，只有点击按钮以外的部分才能够拖动
    // 如果继承的是对应的接口，那么Button将无法接受OnDrag，因此可以进行拖动
    public delegate void VoidDelegate(GameObject go);
    public VoidDelegate onClick;
    public VoidDelegate onDown;
    public VoidDelegate onEnter;
    public VoidDelegate onExit;
    public VoidDelegate onUp;
    public VoidDelegate onSelect;
    public VoidDelegate onDeselect;
    public VoidDelegate onUpdateSelectd;

    static public UIPointerEvent Get(GameObject go)
    {
        UIPointerEvent listener = go.GetComponent<UIPointerEvent>();
        if (listener == null) listener = go.AddComponent<UIPointerEvent>();
        return listener;
    }

    public void OnPointerClick(PointerEventData eventData)
    {
        if (onClick != null) onClick(gameObject);
    }
    public void OnPointerDown(PointerEventData eventData)
    {
        if (onDown != null) onDown(gameObject);
    }
    public void OnPointerEnter(PointerEventData eventData)
    {
        if (onEnter != null) onEnter(gameObject);
    }
    public void OnPointerExit(PointerEventData eventData)
    {
        if (onExit != null) onExit(gameObject);
    }
    public void OnPointerUp(PointerEventData eventData)
    {
        if (onUp != null) onUp(gameObject);
    }
    public void OnSelect(BaseEventData eventData)
    {
        if (onSelect != null) onSelect(gameObject);
    }
    public void OnDeselect(BaseEventData eventData)
    {
        if (onDeselect != null) onDeselect(gameObject);
    }
    public void OnUpdateSelected(BaseEventData eventData)
    {
        if (onUpdateSelectd != null) onUpdateSelectd(gameObject);
    }
}
```

然后是拖拽型 UI：

```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.EventSystems;

/// <summary>
/// 注册拖拽型UI的事件
/// </summary>
public class UIDragEvent : MonoBehaviour, IBeginDragHandler, IDragHandler, IEndDragHandler
{
    public delegate void VoidDelegate(GameObject go);
    public VoidDelegate onBeginDrag;
    public VoidDelegate onDrag;
    public VoidDelegate onEndDrag;

    static public UIDragEvent Get(GameObject go)
    {
        UIDragEvent listener = go.GetComponent<UIDragEvent>();
        if (listener == null) listener = go.AddComponent<UIDragEvent>();
        return listener;
    }

    public void OnBeginDrag(PointerEventData eventData)
    {
        if (onBeginDrag != null) onBeginDrag(gameObject);
    }
    public void OnDrag(PointerEventData eventData)
    {
        if (onDrag != null) onDrag(gameObject);
    }
    public void OnEndDrag(PointerEventData eventData)
    {
        if (onEndDrag != null) onEndDrag(gameObject);
    }
}
```

如果你想要添加长按等逻辑，那么你可以在对应的事件中进行修改，用起来非常灵活。

### 不是很推荐的解决方法

之所以继承 EventTrigger 会造成 ScrollRect 和 Button 冲突，是因为 EventTrigger 和 ScrollRect 都实现了相同的接口。我们来看一下 ScrollRect 的实现：

```csharp
public class ScrollRect : UIBehaviour, IInitializePotentialDragHandler, IBeginDragHandler, IEndDragHandler, IDragHandler, IScrollHandler, ICanvasElement, ILayoutElement, ILayoutGroup, IEventSystemHandler, ILayoutController
{
    // 具体实现看不到，省略
    public virtual void OnBeginDrag(PointerEventData eventData);
    public virtual void OnDrag(PointerEventData eventData);
    public virtual void OnEndDrag(PointerEventData eventData);
    public virtual void OnInitializePotentialDrag(PointerEventData eventData);
    public virtual void OnScroll(PointerEventData data);
    // ...
}
```

看到了吗，ScrollRect 的拖拽也是通过这几个接口实现的，所以才会导致事件冲突。我在开头介绍了 Button 最常见的一种注册方法，那就是使用 Button 自己封装的 OnClick 事件：

```csharp
button.onClick.AddListener(TestCallback);
```

如果 Button 使用这种方式进行监听，那么就不会与 ScrollRect 冲突了。不过呢，当你要对按下、抬起、长按等操作进行识别时，你其实还是要去继承对应接口，也就是说最好用的方法依旧是分离式做法。