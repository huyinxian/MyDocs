# NGUI架构分析

这一章将重点对 NGUI 中几个重要的类进行分析，了解它们的组织结构。

## UITweener

---

先上一张图，我这里列举了常用的几个组件：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E6%9E%B6%E6%9E%84%E5%88%86%E6%9E%90/01.png)

`UITweener` 是 Tween 动画的基类，由它派生出来的组件只是重写了基类的 `OnUpdate`，然后编写了相关的 `Begin` 方法，看懂了基类其他的自然也就懂了。

### amountPerDelta

先来看一段代码：

```csharp
bool mStarted = false;                  // 动画是否开始播放
float mStartTime = 0f;                  // 动画开始
float mDuration = 0f;                   // 动画的时长
float mAmountPerDelta = 1000f;          // 单位时间内动画播放的长度，即播放速度
float mFactor = 0f;                     // 动画的当前进度

/// <summary>
/// Amount advanced per delta time.
/// </summary>

public float amountPerDelta
{
    get
    {
        if (duration == 0f) return 1000f;

        if (mDuration != duration)
        {
            mDuration = duration;
            mAmountPerDelta = Mathf.Abs(1f / duration) * Mathf.Sign(mAmountPerDelta);
        }
        return mAmountPerDelta;
    }
}
```

上面这段代码是对 `mAmountPerDelta` 进行初始化，该属性表示动画的播放速度。

重点说一下这段代码：

```csharp
mAmountPerDelta = Mathf.Abs(1f / duration) * Mathf.Sign(mAmountPerDelta);
```

`Mathf.Sign` 会在传入的值为非负数时返回 1，负数时返回 -1，也就是说 `mAmountPerDelta` 可能会为负值。这样做的原因是因为当动画需要反向播放时，就需要让播放速度为负值。

### Begin

接下来再看看 `Begin` 方法是在干些什么：

```csharp
/// <summary>
/// Starts the tweening operation.
/// </summary>

static public T Begin<T> (GameObject go, float duration, float delay = 0f) where T : UITweener
{
    T comp = go.GetComponent<T>();
#if UNITY_FLASH
    if ((object)comp == null) comp = (T)go.AddComponent<T>();
#else
    // Find the tween with an unset group ID (group ID of 0).
    if (comp != null && comp.tweenGroup != 0)
    {
        comp = null;
        T[] comps = go.GetComponents<T>();
        for (int i = 0, imax = comps.Length; i < imax; ++i)
        {
            comp = comps[i];
            if (comp != null && comp.tweenGroup == 0) break;
            comp = null;
        }
    }

    if (comp == null)
    {
        comp = go.AddComponent<T>();

        if (comp == null)
        {
            Debug.LogError("Unable to add " + typeof(T) + " to " + NGUITools.GetHierarchy(go), go);
            return null;
        }
    }
#endif
    comp.mStarted = false;
    comp.mFactor = 0f;
    comp.duration = duration;
    comp.mDuration = duration;
    comp.delay = delay;
    comp.mAmountPerDelta = duration > 0f ? Mathf.Abs(1f / duration) : 1000f;
    comp.style = Style.Once;
    comp.animationCurve = new AnimationCurve(new Keyframe(0f, 0f, 0f, 1f), new Keyframe(1f, 1f, 1f, 0f));
    comp.eventReceiver = null;
    comp.callWhenFinished = null;
    comp.onFinished.Clear();
    if (comp.mTemp != null) comp.mTemp.Clear();
    comp.enabled = true;
    return comp;
}
```

`Begin` 方法会被 Tweener 派生出来的组件所调用，用于动画各项参数的初始化。

### DoUpdate

`DoUpdate` 主要用于动画的播放，它会根据动画的循环模式（一次、循环、反复）来修改 `mFactor`。之前也说了，`mFactor` 表示动画当前的播放进度。

```csharp
/// <summary>
/// Update the tweening factor and call the virtual update function.
/// </summary>

protected void DoUpdate ()
{
    float delta = ignoreTimeScale && !useFixedUpdate ? Time.unscaledDeltaTime : Time.deltaTime;
    float time = ignoreTimeScale && !useFixedUpdate ? Time.unscaledTime : Time.time;

    if (!mStarted)
    {
        delta = 0;
        mStarted = true;
        mStartTime = time + delay;
    }

    if (time < mStartTime) return;

    // Advance the sampling factor
    mFactor += (duration == 0f) ? 1f : amountPerDelta * delta * timeScale;

    // Loop style simply resets the play factor after it exceeds 1.
    if (style == Style.Loop)
    {
        if (mFactor > 1f)
        {
            mFactor -= Mathf.Floor(mFactor);
        }
    }
    else if (style == Style.PingPong)
    {
        // Ping-pong style reverses the direction
        if (mFactor > 1f)
        {
            mFactor = 1f - (mFactor - Mathf.Floor(mFactor));
            mAmountPerDelta = -mAmountPerDelta;
        }
        else if (mFactor < 0f)
        {
            mFactor = -mFactor;
            mFactor -= Mathf.Floor(mFactor);
            mAmountPerDelta = -mAmountPerDelta;
        }
    }

    // If the factor goes out of range and this is a one-time tweening operation, disable the script
    if ((style == Style.Once) && (duration == 0f || mFactor > 1f || mFactor < 0f))
    {
        mFactor = Mathf.Clamp01(mFactor);
        Sample(mFactor, true);
        enabled = false;

        if (current != this)
        {
            UITweener before = current;
            current = this;

            if (onFinished != null)
            {
                mTemp = onFinished;
                onFinished = new List<EventDelegate>();

                // Notify the listener delegates
                EventDelegate.Execute(mTemp);

                // Re-add the previous persistent delegates
                for (int i = 0; i < mTemp.Count; ++i)
                {
                    EventDelegate ed = mTemp[i];
                    if (ed != null && !ed.oneShot) EventDelegate.Add(onFinished, ed, ed.oneShot);
                }
                mTemp = null;
            }

            // Deprecated legacy functionality support
            if (eventReceiver != null && !string.IsNullOrEmpty(callWhenFinished))
                eventReceiver.SendMessage(callWhenFinished, this, SendMessageOptions.DontRequireReceiver);

            current = before;
        }
    }
    else Sample(mFactor, false);
}
```

在获取了动画的进度后，要怎么修改动画的状态呢？代码调用了 `Sample` 方法，这个方法会根据 `mFactor` 的值来修改动画的状态。

### Sample

Tweener 创建了一个 `AnimationCurve`：

```csharp
/// <summary>
/// Optional curve to apply to the tween's time factor value.
/// </summary>

[HideInInspector]
public AnimationCurve animationCurve = new AnimationCurve(new Keyframe(0f, 0f, 0f, 1f), new Keyframe(1f, 1f, 1f, 0f));
```

动画曲线最主要的作用是让补间动画呈曲线变化，符合现实中的物理特性。另外，动画曲线虽然可以手动调节，但是也有许多约定俗成的函数可供我们使用：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E6%9E%B6%E6%9E%84%E5%88%86%E6%9E%90/02.jpg)

上面的这些函数叫做**缓冲函数**。

再来看 `Sample` 的代码：

```csharp
/// <summary>
/// Sample the tween at the specified factor.
/// </summary>

public void Sample (float factor, bool isFinished)
{
    // Calculate the sampling value
    float val = Mathf.Clamp01(factor);

    if (method == Method.EaseIn)
    {
        val = 1f - Mathf.Sin(0.5f * Mathf.PI * (1f - val));
        if (steeperCurves) val *= val;
    }
    else if (method == Method.EaseOut)
    {
        val = Mathf.Sin(0.5f * Mathf.PI * val);

        if (steeperCurves)
        {
            val = 1f - val;
            val = 1f - val * val;
        }
    }
    else if (method == Method.EaseInOut)
    {
        const float pi2 = Mathf.PI * 2f;
        val = val - Mathf.Sin(val * pi2) / pi2;

        if (steeperCurves)
        {
            val = val * 2f - 1f;
            float sign = Mathf.Sign(val);
            val = 1f - Mathf.Abs(val);
            val = 1f - val * val;
            val = sign * val * 0.5f + 0.5f;
        }
    }
    else if (method == Method.BounceIn)
    {
        val = BounceLogic(val);
    }
    else if (method == Method.BounceOut)
    {
        val = 1f - BounceLogic(1f - val);
    }

    // Call the virtual update
    OnUpdate((animationCurve != null) ? animationCurve.Evaluate(val) : val, isFinished);
}
```

`Sample` 方法的最后调用了 `OnUpdate` 方法，该方法会在派生子类重写。当动画曲线不存在时，会传入动画结束的标识；当动画曲线存在时，会传入之前计算出的 `val`（其实就是 mFactor），然后在 `OnUpdate` 中改变物体的位置。

顺带一提，`Method` 是一个枚举体，里面包含的是几个缓冲函数：

```csharp
[DoNotObfuscateNGUI] public enum Method
{
    Linear,
    EaseIn,
    EaseOut,
    EaseInOut,
    BounceIn,
    BounceOut,
}
```

### 派生子类

由于每一个派生类都是差不多的，这里就以 TweenPosition 组件为例。

首先最主要的是重写了 `OnUpdate`：

```csharp
/// <summary>
/// Tween's current value.
/// </summary>

public Vector3 value
{
    get
    {
        return worldSpace ? cachedTransform.position : cachedTransform.localPosition;
    }
    set
    {
        if (mRect == null || !mRect.isAnchored || worldSpace)
        {
            if (worldSpace) cachedTransform.position = value;
            else cachedTransform.localPosition = value;
        }
        else
        {
            value -= cachedTransform.localPosition;
            NGUIMath.MoveRect(mRect, value.x, value.y);
        }
    }
}

/// <summary>
/// Tween the value.
/// </summary>

protected override void OnUpdate (float factor, bool isFinished) { value = from * (1f - factor) + to * factor; }
```

重点说一下 `value`。在 TweenPosition 中，`value` 代表当前物体的坐标。但如果是在其他的动画组件中，它就有可能是缩放值、透明度等等。但不管是哪一种，`OnUpdate` 都可以使用下面这行代码来改变 `value`：

```csharp
value = from * (1f - factor) + to * factor;
```

当然，具体情况具体分析，OnUpdate 有时也要处理其他的参数。

之后再来看看子类中的 `Begin` 方法：

```csharp
/// <summary>
/// Start the tweening operation.
/// </summary>

static public TweenPosition Begin (GameObject go, float duration, Vector3 pos)
{
    TweenPosition comp = UITweener.Begin<TweenPosition>(go, duration);
    comp.from = comp.value;
    comp.to = pos;

    if (duration <= 0f)
    {
        comp.Sample(1f, true);
        comp.enabled = false;
    }
    return comp;
}
```

如果 `duration` 小于等于 0 时，就相当于瞬移了。

## UIRoot

---

只要接触过 NGUI 插件，就一定会知道 `UIRoot` 有多重要。在我们开发 UI 界面时，会以 UIRoot 为基础来创建一个 UI 树。

### 缩放方式

如果不明白它的作用，可以看一看属性面板：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E6%9E%B6%E6%9E%84%E5%88%86%E6%9E%90/03.png)

其实之前在基础篇的时候我已经介绍过了，UIRoot 用于控制整个 UI 树的缩放，一共包含有三种方式：

```csharp
[DoNotObfuscateNGUI] public enum Scaling
{
    Flexible,
    Constrained,
    ConstrainedOnMobiles,
}
```

对于不同的缩放方式，UIRoot 会依据不同的值来进行缩放，比如 Flexible 会依据 `minimumHeight`、`maximumHeight` 等等。

?> 如果对于缩放方式有疑问，请去看 NGUI 基础篇。

### ConstrainedOnMobiles

在该模式下，游戏会根据其所处的平台自动选择缩放方式。具体实现如下：

```csharp
/// <summary>
/// Active scaling type, based on platform.
/// </summary>

public Scaling activeScaling
{
    get
    {
        Scaling scaling = scalingStyle;

        if (scaling == Scaling.ConstrainedOnMobiles)
#if UNITY_EDITOR || UNITY_IPHONE || UNITY_ANDROID || UNITY_WP8 || UNITY_WP_8_1 || UNITY_BLACKBERRY
            return Scaling.Constrained;
#else
            return Scaling.Flexible;
#endif
        return scaling;
    }
}
```

也就是说，如果是在编辑器、iphone、安卓、WP8、WP8_1、黑莓等平台下，游戏会选择 `Constrained` 缩放模式，而其他情况下会选择 `Flexible` 模式。

### Flexible与Constrained

之前我曾粗浅地介绍过这两种模式，但其具体实现方式还是需要我们仔细研究：

```csharp
/// <summary>
/// UI Root's active height, based on the size of the screen.
/// </summary>

public int activeHeight
{
    get
    {
        Scaling scaling = activeScaling;

        if (scaling == Scaling.Flexible)
        {
            Vector2 screen = NGUITools.screenSize;
            float aspect = screen.x / screen.y;

            if (screen.y < minimumHeight)
            {
                screen.y = minimumHeight;
                screen.x = screen.y * aspect;
            }
            else if (screen.y > maximumHeight)
            {
                screen.y = maximumHeight;
                screen.x = screen.y * aspect;
            }

            // Portrait mode uses the maximum of width or height to shrink the UI
            int height = Mathf.RoundToInt((shrinkPortraitUI && screen.y > screen.x) ? screen.y / aspect : screen.y);

            // Adjust the final value by the DPI setting
            return adjustByDPI ? NGUIMath.AdjustByDPI(height) : height;
        }
        else
        {
            Constraint cons = constraint;
            if (cons == Constraint.FitHeight)
                return manualHeight;

            Vector2 screen = NGUITools.screenSize;
            float aspect = screen.x / screen.y;
            float initialAspect = (float)manualWidth / manualHeight;

            switch (cons)
            {
                case Constraint.FitWidth:
                {
                    return Mathf.RoundToInt(manualWidth / aspect);
                }
                case Constraint.Fit:
                {
                    return (initialAspect > aspect) ?
                        Mathf.RoundToInt(manualWidth / aspect) :
                        manualHeight;
                }
                case Constraint.Fill:
                {
                    return (initialAspect < aspect) ?
                        Mathf.RoundToInt(manualWidth / aspect) :
                        manualHeight;
                }
            }
            return manualHeight;
        }
    }
}
```

首先，如果想要获取 UI 的实际高度，就得先通过 `NGUITools` 获取实际屏幕尺寸，然后算出屏幕的宽高比 `aspect`。之前我也讲过，`Flexible` 模式需要设置最小和最大高度，如果实际屏幕高度超出这个范围，那么屏幕高度就会被限制在边界值，然后根据 `aspect` 算出实际屏幕宽度。

在计算 `height` 时，如果游戏是处于竖屏模式，且屏幕高度大于宽度，那么实际高度就会等于 `screen.y / aspect`；如果不是，那么就直接使用 `screen.y`。

如果游戏用的是 `Constrained` 模式，那么 UI 界面会进行缩放以保持最佳的宽高比。在不同的分辨率下，UI 的大小看起来都会是一样的。

### 缩放实现

UIRoot 的缩放是如何实现的呢？看下 `UpdateScale` 方法就知道了：

```csharp
/// <summary>
/// Immediately update the root's scale. Call this function after changing the min/max/manual height values.
/// </summary>

public void UpdateScale (bool updateAnchors = true)
{
    if (mTrans != null)
    {
        float calcActiveHeight = activeHeight;

        if (calcActiveHeight > 0f)
        {
            float size = 2f / calcActiveHeight;

            Vector3 ls = mTrans.localScale;

            if (!(Mathf.Abs(ls.x - size) <= float.Epsilon) ||
                !(Mathf.Abs(ls.y - size) <= float.Epsilon) ||
                !(Mathf.Abs(ls.z - size) <= float.Epsilon))
            {
                mTrans.localScale = new Vector3(size, size, size);
                if (updateAnchors) BroadcastMessage("UpdateAnchors", SendMessageOptions.DontRequireReceiver);
            }
        }
    }
}
```

`activeHeight` 就是上面讲过的实际高度，游戏根据这个值来确定 UI 的缩放。可能有人会好奇，`2f / calcActiveHeight` 是怎么来的？这就要说到摄像机了。

### Camera

先来看一下 UIRoot 下的 Camera：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E6%9E%B6%E6%9E%84%E5%88%86%E6%9E%90/04.png)

Camera 的默认尺寸是 `1`，投影方式为正交投影。注意，这个 `size` 值表示视窗的一半高度，也就是说视野的高度其实是 `2`。

这样一来，我想你应该知道 `2f` 是怎么回事了。

### 影响UI大小的因素

其实能够影响 UI 大小的不只是缩放方式，像是锚点、相机都可以影响。下一次我将分析一下 UICamera 组件。