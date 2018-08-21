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

## 技能冷却

---

当游戏角色在释放技能后，技能图标会进入冷却状态。那么现在就教大家来做一个简单的技能冷却效果。

创建一个精灵，把它作为我们的技能图标。然后为它创建一个纯黑色的子精灵，将它的模式调整为 `Filled`，填充方式设置为 `Vertical`。为了能看清背后的技能图标，要调整一下透明度：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80%E6%A1%88%E4%BE%8B/07.png)

编写一个脚本，用于控制黑色遮罩的填充值：

```csharp
public class Skill : MonoBehaviour {

    public UISprite skillA;
    
    private bool isReleased;

    void Start() {
        isReleased = false;
    }

    void Update() {
        if (Input.GetKey(KeyCode.A) && isReleased == false) {
            isReleased = true;
            skillA.fillAmount = 1f;
        }

        if (isReleased == true) {
            skillA.fillAmount -= Time.deltaTime / 2;

            if (skillA.fillAmount < 0.05f) {
                skillA.fillAmount = 0f;
                isReleased = false;
            }
        }
    }
}
```

将黑色遮罩赋值给脚本：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80%E6%A1%88%E4%BE%8B/08.png)

那么当我们按下 A 键时，技能就会进入冷却，并且在冷却完成前不可以再次释放技能：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80%E6%A1%88%E4%BE%8B/09.png)

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80%E6%A1%88%E4%BE%8B/10.png)

## 简易聊天室

---

创建一个底框，添加 `UIDragObject` 组件。然后添加聊天框背景以及缩放按钮：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80%E6%A1%88%E4%BE%8B/11.png)

用于缩放的组件是 `UIDragResize`：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80%E6%A1%88%E4%BE%8B/12.png)

当然，由于我们还没有设置锚点，所以缩放之后 UI 就会变形：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80%E6%A1%88%E4%BE%8B/13.png)

设置锚点之后再看：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80%E6%A1%88%E4%BE%8B/14.png)

接下来，为聊天框区域添加 `UITextList` 组件，然后再添加滚动条以及输入框。这样，基本的聊天框就搭建完毕了：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80%E6%A1%88%E4%BE%8B/15.png)

当然，这还没完，你得为输入框做一些改动。首先更改下面的设置：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80%E6%A1%88%E4%BE%8B/16.png)

这个 `RemoveFocus` 会在你输入完成后取消光标，如果你想继续输入的话就得重新点一下输入框，非常麻烦。为了输入的流畅性，建议把这个方法给取消掉。

接下来为输入框写一个脚本，处理输入的内容：

```csharp
public class ChangeInputContent : MonoBehaviour {

    [HideInInspector] public UIInput input;
    public UITextList textList;

    private string[] names = new string[] {
        "Cappuccino",
        "huyinxian",
        "大哥",
        "管理"
    };

    void Start() {
        input = this.GetComponent<UIInput>();
    }
    
    public void OnChatInput() {
        string chatMessage = input.value;
        string name = names[Random.Range(0, 4)];
        textList.Add(name + "：" + chatMessage);
        input.value = "";
    }
}
```

一般来说，聊天室会取到用户的 ID，然后在每段文字的前面加上用户名。由于我们这个只是测试用例，所以就随便弄了几个名字。

为相关的属性赋值：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80%E6%A1%88%E4%BE%8B/17.png)

效果如下：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80%E6%A1%88%E4%BE%8B/18.png)

## 简易背包

---

教各位做一个很简单的背包系统。首先弄个背景，然后摆上一堆小格子（建议做成预制体）。随后选择一张你喜欢的图片，把它当做物品（最好也做成预制体）。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80%E6%A1%88%E4%BE%8B/19.png)

拖动物品需要用到 `UIDragDropItem` 组件，这个组件与 `UIDragObject` 的差别就在于它可以设置松开鼠标后的操作。比如我们想要让物体始终处于当前格子的中心点，那么就可以这么写：

```csharp
public class BackpackItem : UIDragDropItem {
    protected override void OnDragDropRelease(GameObject surface) {
        base.OnDragDropRelease(surface);

        this.transform.parent = surface.transform;
        this.transform.localPosition = Vector3.zero;
    }
}
```

这里写的脚本是继承了 UIDragDropItem，并且重写了其中的 `OnDragDropRelease` 方法。该方法含有一个 `surface` 参数，包含了与物品碰撞的游戏物体。

如果不懂的话可以看看效果，当你拖动物品至另一个格子时，物品就会自动设置到这个格子的中心点。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80%E6%A1%88%E4%BE%8B/20.png)

光能够拖动还不够，假如另一个格子上已经存在了物品，那么就应该将这两个物品进行交换：

```csharp
public class BackpackItem : UIDragDropItem {
    protected override void OnDragDropRelease(GameObject surface) {
        base.OnDragDropRelease(surface);

        // 如果物品拖动的位置是空格，那么将物体的父对象设置为这个空格
        if (surface.tag == "Cell") {
            this.transform.parent = surface.transform;
            this.transform.localPosition = Vector3.zero;

        // 如果物品拖动的位置已经有物品了，那么交换这两个物品
        } else if (surface.tag == "Item") {
            Transform parent = surface.transform.parent;
            surface.transform.parent = this.transform.parent;
            surface.transform.localPosition = Vector3.zero;

            this.transform.parent = parent;
            this.transform.localPosition = Vector3.zero;
        }
    }
}
```

如果你分不清的话，可以给物品做上标识。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80%E6%A1%88%E4%BE%8B/21.png)

?> 记得要给每个格子加上 Box Collider。