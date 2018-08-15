## 界面切换

---

教各位一个简单的界面切换。先创建两个界面，一个是开始界面，另一个则是选项界面。为了方便管理，最好是将两个界面的元素分别用一个不可见的 Widget 包装好。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80%E6%A1%88%E4%BE%8B/01.png)

之后分别为两个界面添加 `Tween Position` 动画。在最开始时要将动画脚本禁用：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80%E6%A1%88%E4%BE%8B/02.png)

写一个脚本，用于控制动画的播放：

```csharp
public class GameSetting : MonoBehaviour {

    public TweenPosition welcomeLayerTween;
    public TweenPosition optionLayerTween;

    /// <summary>
    /// 选项按钮响应事件
    /// </summary>
    public void OnOptionButtonClick() {
        welcomeLayerTween.PlayForward();
        optionLayerTween.PlayForward();
    }

    public void OnCompleteButtonClick() {
        welcomeLayerTween.PlayReverse();
        optionLayerTween.PlayReverse();
    }
}
```

随便找一个对象挂上脚本，然后分别在选项按钮以及完成按钮上设置响应事件：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80%E6%A1%88%E4%BE%8B/03.png)

别忘了，脚本需要获取两个 UI 层的动画组件：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80%E6%A1%88%E4%BE%8B/04.png)

效果如下：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80%E6%A1%88%E4%BE%8B/05.png)

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80%E6%A1%88%E4%BE%8B/06.png)

当玩家点击选项按钮时，界面就会切换到选项页面；当玩家设置完毕后，界面就会切换回开始界面。

!> 必须要将各类 UI 控件分别包含到各自的界面中。另外，要记得设置按钮的响应事件。