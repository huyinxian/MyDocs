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

`value` 代表当前物体的坐标，用于取代旧版中的 `position`。`cachedTransform` 其实就是当前物体的 Transform 组件。至于 `OnUpdate` 主要作用是根据传入的动画进度修改物体的坐标。

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