# MVC开发俄罗斯方块

俄罗斯方块相信各位都玩过，我也就不过多介绍了。这次主要是以俄罗斯方块为例子，让各位简单的了解一下 MVC 框架。

项目源码地址——[GitHub](https://github.com/huyinxian/Tetris)。

## 什么是MVC

---

MVC 全称是 Model View Controller（模型-视图-控制器），是一种将数据、界面、逻辑分离的代码组织方式。

MVC 的各组成部分如下：

* **Model**：负责处理应用程序的数据逻辑。
* **View**：处理界面的显示。
* **Controller**：处理用户交互，负责从视图中读取数据，控制用户输入，并向模型发送数据。

在该框架中，控制器负责与视图和模型交互，但模型与视图之间是没有关联的。基于这一点，当我们需要改进界面和交互时，数据逻辑并不会受到影响。

## 开发准备

---

我们需要事先对图片等素材进行处理，下面是我们将要用到的素材：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/MVC%E5%BC%80%E5%8F%91%E4%BF%84%E7%BD%97%E6%96%AF%E6%96%B9%E5%9D%97/01.png)

既然我们选择了使用 MVC 框架进行开发，那么我们就得先创建 Model、View、Controller 三个对象，然后分别为它们创建一个脚本：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/MVC%E5%BC%80%E5%8F%91%E4%BF%84%E7%BD%97%E6%96%AF%E6%96%B9%E5%9D%97/02.png)

资源文件中有美工准备好的调色盘，我们可以根据喜好来选择这些颜色：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/MVC%E5%BC%80%E5%8F%91%E4%BF%84%E7%BD%97%E6%96%AF%E6%96%B9%E5%9D%97/03.png)

## 创建UI

---

UI 界面的操作我就不多讲了，各位可以根据我上传的项目工程进行对比，我这里只讲几个主要的。

### UI界面

我们一共需要五个界面，开始界面、游戏界面、设置界面、排行榜界面、游戏结束界面。每个界面需要稍微分一下层，便于之后制作界面切换动画。

### 按钮制作

为了方便调节按钮上的图标，可以将它的锚点设置为全方向拉伸，然后调整它与按钮背景的间距即可：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/MVC%E5%BC%80%E5%8F%91%E4%BF%84%E7%BD%97%E6%96%AF%E6%96%B9%E5%9D%97/05.png)

Image 组件中有一个 `Raycast Target` 属性，勾选该属性后，鼠标点击到该物体后将不再穿透到下面的物体。比如你有一个图片，然后有一个按钮覆盖在图片上。当你勾选 `Raycast Target` 时，只有按钮会响应事件，而按钮下的图片则不会响应。由于我们的背景是不响应事件的，为了节约性能，我们把该选项给取消掉吧。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/MVC%E5%BC%80%E5%8F%91%E4%BF%84%E7%BD%97%E6%96%AF%E6%96%B9%E5%9D%97/07.png)

按钮的排列也不难，往按钮层的父对象上挂一个 `Horizontal Layout Group` 组件即可：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/MVC%E5%BC%80%E5%8F%91%E4%BF%84%E7%BD%97%E6%96%AF%E6%96%B9%E5%9D%97/08.png)

?> `Raycast Target` 可以用于暂停界面等。

### 大尺寸背景图

素材中有一张大的圆角矩形，可以用来制作背景图片。为了不让圆角边框失真，可以采取九宫切图的方式来进行制作，需要将图片放大时只需把图片的类型设置为 `Sliced` 即可：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/MVC%E5%BC%80%E5%8F%91%E4%BF%84%E7%BD%97%E6%96%AF%E6%96%B9%E5%9D%97/06.png)

!> 注意，九宫格切图是拖动绿色的线框，不是蓝色的线框。

### 设置界面

就如上面所说的，如果你要制作一个设置界面或者暂停界面，你就应该把 `Raycast Target` 勾选上，防止玩家误触：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/MVC%E5%BC%80%E5%8F%91%E4%BF%84%E7%BD%97%E6%96%AF%E6%96%B9%E5%9D%97/13.png)

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/MVC%E5%BC%80%E5%8F%91%E4%BF%84%E7%BD%97%E6%96%AF%E6%96%B9%E5%9D%97/14.png)

### 游戏地图

素材中有一个 62×62 的小方块，由于我们设置的换算尺寸是 70 像素为 1 米，所以方块的位置可以用米来做单位，而且方块间还会有 8 像素的间隔：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/MVC%E5%BC%80%E5%8F%91%E4%BF%84%E7%BD%97%E6%96%AF%E6%96%B9%E5%9D%97/17.png)

我设置的棋盘原点是左下角，如果你想要让方块以米为单位进行移动，那么你可以选中移动工具，然后按住 `ctrl` 键拖动方块。

接下来，我们还需要设置相机的位置。根据计算，棋盘的中心点为 (4.5, 9.6)：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/MVC%E5%BC%80%E5%8F%91%E4%BF%84%E7%BD%97%E6%96%AF%E6%96%B9%E5%9D%97/18.png)

另外，可以根据你的喜好来调节相机的 size，让棋盘处在视野中。

## UI界面的切换

---

开始界面如下：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/MVC%E5%BC%80%E5%8F%91%E4%BF%84%E7%BD%97%E6%96%AF%E6%96%B9%E5%9D%97/28.png)

游戏界面如下：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/MVC%E5%BC%80%E5%8F%91%E4%BF%84%E7%BD%97%E6%96%AF%E6%96%B9%E5%9D%97/25.png)

UI 的移进移出其实很好解决，只需要使用 DoTween 插件来改变位置即可。但棋盘的放大和缩小如何解决呢？很简单，改变相机的 size 即可。

_CameraManager.cs_

```csharp
/// <summary>
/// 放大相机视野
/// </summary>
public void ZoomIn()
{
    mainCamera.DOOrthoSize(14.5f, 0.5f);
}

/// <summary>
/// 缩小相机视野
/// </summary>
public void ZoomOut()
{
    mainCamera.DOOrthoSize(20.0f, 0.5f);
}
```

相机视野的调整放到了 CameraManager 中，它在代码结构中属于控制层。另外，视野的大小可以根据需要自行调整，不必按照我的数值来。

UI 层的移动也很简单：

_view.cs_

```csharp
public void ShowMenuLayer()
{
    logoLabel.gameObject.SetActive(true);
    logoLabel.DOAnchorPosY(-190.4f, 0.5f);

    menuLayer.gameObject.SetActive(true);
    menuLayer.DOAnchorPosY(80f, 0.5f);
}

public void HideMenuLayer()
{
    logoLabel.DOAnchorPosY(192.2f, 0.5f)
        .OnComplete(delegate { logoLabel.gameObject.SetActive(false); });

    menuLayer.DOAnchorPosY(-80f, 0.5f)
        .OnComplete(delegate { menuLayer.gameObject.SetActive(false); });
}

public void ShowGameLayer(int score = 0, int highScore = 0)
{
    UpdateGameLayer(score, highScore);

    gameLayer.gameObject.SetActive(true);
    gameLayer.DOAnchorPosY(-158.2f, 0.5f);
}

public void HideGameLayer()
{
    gameLayer.DOAnchorPosY(160f, 0.5f)
        .OnComplete(delegate { gameLayer.gameObject.SetActive(false); });
}
```

UI 层的移动写在了视图中。这里解释一下 `OnComplete` 方法，该方法会在 Tween 动画执行完毕后，执行代理中的方法。以 `HideGameLayer` 为例，当游戏界面的 UI 移出屏幕后，游戏界面就会被设置为未激活状态。

!> 这里其实有一个 BUG，如果玩家快速地切换界面，那么 UI 将会永远地隐藏。建议是不要将 UI 界面设置为未激活。

## 制作方块

---

各位在制作方块时，最好把小矩形制作成一个预制体。比如我把独立的小矩形做成了预制体 `Block`，然后用 `Block` 拼出方块 `Shape - 1`：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/MVC%E5%BC%80%E5%8F%91%E4%BF%84%E7%BD%97%E6%96%AF%E6%96%B9%E5%9D%97/19.png)

另外，由于我们的方块需要旋转，所以还得在 Shape 下创建一个空物体 `Pivot` 用于标识方块的旋转中心。至于 `Pivot` 的具体位置，我建议是以靠近方块中心的小矩形为旋转中心：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/MVC%E5%BC%80%E5%8F%91%E4%BF%84%E7%BD%97%E6%96%AF%E6%96%B9%E5%9D%97/21.png)

由于田字形方块不需要旋转，所以可以直接把 `Pivot` 设置为方块的中心点：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/MVC%E5%BC%80%E5%8F%91%E4%BF%84%E7%BD%97%E6%96%AF%E6%96%B9%E5%9D%97/20.png)

所有方块一览：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/MVC%E5%BC%80%E5%8F%91%E4%BF%84%E7%BD%97%E6%96%AF%E6%96%B9%E5%9D%97/31.png)

另外，为了让方块的颜色是随机的，我们可以预先存储几种颜色。这里我就直接从调色盘上取颜色了：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/MVC%E5%BC%80%E5%8F%91%E4%BF%84%E7%BD%97%E6%96%AF%E6%96%B9%E5%9D%97/32.png)

?> Color 数组的初始值为 (0, 0, 0, 0)，所以在提取了颜色后，还得把颜色的透明度拉满。

## FSM有限状态机

---

游戏总共有四个状态：开始菜单状态、游戏状态、暂停状态、结束状态。不过由于我们制作了界面的切换效果，最终用到的状态只有开始菜单状态和游戏状态，这个算是设计上的小瑕疵。

既然我们用了开发框架，那么游戏状态的切换自然也应该正式一点。相比较于使用 switch 语句进行切换，FSM 有限状态机要严谨的多。

代码的话不需要我们来写，Unity Wiki 上面已经有现成的代码了，可以直接去复制：[Finite State Machine](http://wiki.unity3d.com/index.php/Finite_State_Machine)。我们只需要创建一个新的脚本 `FSMSystem.cs`，然后把网站上对应的代码复制进去即可。

当然，为了让状态机适应我们的游戏，我们还得为它做一些更改，各位可以对照我上传的代码，看看哪里有不同。我会在设计模式部分详细地介绍状态机。

## 游戏进程的控制

---

代码部分我不会细讲，各位可以自行查看源码。控制层我写了四个脚本，作用如下：

* **CameraManager**：相机控制器，用于相机的缩放。
* **Controller**：控制器，主要用于数据的交互。
* **GameManager**：游戏控制器，用于控制游戏进程、方块的生成、改变分数 UI 等。
* **Shape**：方块脚本，用于控制方块的移动、方块的碰撞判定、方块消除等。

当玩家点击暂停按钮时，游戏状态会切换至开始菜单状态，然后执行相关方法：

_PlayState.cs_

```csharp
public override void DoBeforeLeaving()
{
    ctrl.view.HideGameLayer();
    ctrl.view.ShowRestartButton();
    ctrl.gameManager.PauseGame();
}

public void OnPauseButtonClick()
{
    ctrl.audioManager.PlayCursor();
    fsm.PerformTransition(Transition.PauseButtonClick);
}
```

当状态机从 A 状态变换到 B 状态时，会执行 A 状态的 `DoBeforeLeaving` 方法和 B 状态的 `DoBeforeEntering` 方法。简单点说，就是离开时执行的方法和进入时执行的方法。这样，游戏就能够自由地在开始菜单和游戏界面间切换：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/MVC%E5%BC%80%E5%8F%91%E4%BF%84%E7%BD%97%E6%96%AF%E6%96%B9%E5%9D%97/33.png)

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/MVC%E5%BC%80%E5%8F%91%E4%BF%84%E7%BD%97%E6%96%AF%E6%96%B9%E5%9D%97/34.png)

## 方块的碰撞判定

---

方块如果要停止运动，那就得碰到底边或者碰到其它的方块。其次，方块在左右移动时，还不能够超出地图。

地图信息存储在模型层，所以碰撞检测也就写在了这里：

_Model.cs_

```csharp
/// <summary>
/// 判断方块的位置是否存在其它的方块
/// </summary>
/// <param name="t"></param>
/// <returns></returns>
public bool IsValidMapPosition(Transform t)
{
    foreach (Transform child in t)
    {
        if (child.tag != "Block") { continue; }

        // 这里扩展了一个Vector2的Round方法，用于对x和y取整
        // 图形在旋转时可能会出现坐标减少的情况，例如4->3.99999
        Vector2 pos = child.position.Round();

        // 判断方块是否超出了地图边界（左、右、下边界）
        if (IsInsideMap(pos) == false) { return false; }

        // 判断当前方块是否与其它方块重叠
        if (map[(int)pos.x, (int)pos.y] != null) { return false; }
    }

    return true;
}

/// <summary>
/// 判断当前方块是否超出边界
/// </summary>
/// <param name="pos"></param>
bool IsInsideMap(Vector2 pos)
{
    return pos.x >= 0 && pos.x < MAX_COLUMNS && pos.y >= 0;
}
```

注释已经写得很清楚了，因为坐标是用浮点数存储的，在旋转时可能会出现小数，所以得对旋转后的坐标进行取整。由于 `Mathf.Round` 只针对 `Vector3`，所以我们得写一个扩展方法：

_Vector3Extension.cs_

```csharp
public static class Vector3Extension
{
    public static Vector2 Round(this Vector3 v)
    {
        int x = Mathf.RoundToInt(v.x);
        int y = Mathf.RoundToInt(v.y);

        return new Vector2(x, y);
    }
}
```

顺带一提，可能会有人对变量的定义存在疑问：

```csharp
public const int NORMAL_ROWS = 20;          // 地图的正常行数为20行
public const int MAX_ROWS = 22;             // 最大行数为22行（包括方块生成的位置）
public const int MAX_COLUMNS = 10;
```

地图总共有 20 行，不过由于方块生成时是处于 21-22 行，最大行数应该是 22 行。

## 方块的消除

---

方块的消除是在方块落地后判定的，由于需要检测当前的地图信息，所以得写在模型层中：

_Model.cs_

```csharp
/// <summary>
/// 当方块落地时，在map中标识方块的位置，并检查是否有能被消除的行
/// </summary>
/// <param name="t"></param>
/// <returns>返回布尔值，表示是否有行被销毁</returns>
public bool PlaceShape(Transform t)
{
    foreach (Transform child in t)
    {
        if (child.tag != "Block") { continue; }

        Vector2 pos = child.position.Round();
        map[(int)pos.x, (int)pos.y] = child;
    }

    return CheckMap();
}

/// <summary>
/// 检查地图是否存在可消除的行
/// </summary>
bool CheckMap()
{
    int count = 0;

    for (int i = 0; i < MAX_ROWS; i++)
    {
        // 当i行满了之后，需要删除i行，然后将它上面的行全都往下移
        // 当上一行下移成为i行后，继续判断i行是否已满
        if (CheckRowFull(i))
        {
            count++;
            DeleteRow(i);
            MoveDownAllRows(i + 1);
            i--;
        }
    }

    if (count > 0)
    {
        score += count * 100;
        if (score > highScore)
        {
            highScore = score;
        }

        isDataUpdate = true;

        return true;
    }
    else
    {
        return false;
    }
}

/// <summary>
/// 判断当前行是否已经满了
/// </summary>
bool CheckRowFull(int row)
{
    for (int i = 0; i < MAX_COLUMNS; i++)
    {
        if (map[i, row] == null) { return false; }
    }

    return true;
}

/// <summary>
/// 消除当前行
/// </summary>
/// <param name="row"></param>
void DeleteRow(int row)
{
    for (int i = 0; i < MAX_COLUMNS; i++)
    {
        Destroy(map[i, row].gameObject);
        map[i, row] = null;
    }
}

/// <summary>
/// 将传入行之上的所有行往下移动
/// </summary>
/// <param name="row"></param>
void MoveDownAllRows(int row)
{
    for (int i = row; i < MAX_ROWS; i++)
    {
        MoveDownRow(i);
    }
}

/// <summary>
/// 将传入的行往下移动一行
/// </summary>
/// <param name="row"></param>
void MoveDownRow(int row)
{
    for (int i = 0; i < MAX_COLUMNS; i++)
    {
        if (map[i, row] != null)
        {
            map[i, row - 1] = map[i, row];
            map[i, row] = null;
            map[i, row - 1].position += new Vector3(0, -1, 0);
        }
    }
}
```

方法虽然比较多，但这是为了方便起见，把功能拆分成了几个小方法，逻辑还是很简单的。当方块落地后，只需要调用 `PlaceShape` 方法就能够判断是否有可以消除的行。

## 游戏结束

---

游戏结束的判定比较简单，只要有方块超出地图，那么游戏就结束了：

_Model.cs_

```csharp
/// <summary>
/// 当方块超出地图时，游戏结束，保存分数
/// </summary>
/// <returns></returns>
public bool IsGameOver()
{
    for (int i = NORMAL_ROWS; i < MAX_ROWS; i++)
    {
        for (int j = 0; j < MAX_COLUMNS; j++)
        {
            if (map[j, i] != null)
            {
                numberOfGames++;
                SaveData();

                return true;
            }
        }
    }

    return false;
}
```

可能有人会问，方块生成的时候不是处在地图外面吗？那游戏岂不是直接就结束了？当然不是，`IsGameOver` 方法只会在上一个方块停止运动后调用，假如没有方块超出地图，那么下一个方块才会生成。这样，游戏结束的判定就没有问题了。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/MVC%E5%BC%80%E5%8F%91%E4%BF%84%E7%BD%97%E6%96%AF%E6%96%B9%E5%9D%97/39.png)

## UI的一些问题

---

这里讲一个小知识，当我们打开设置界面时，如果我们想要退出界面，其实不需要额外弄一个按钮，只需要给 `SettingLayer` 挂上一个 `Button` 组件即可。把按钮的动画设置为 `None`，然后为它写一个响应事件：

```csharp
public void OnSettingLayerClick()
{
    ctrl.audioManager.PlayCursor();
    settingLayer.gameObject.SetActive(false);
}
```

这样，当你点击设置界面的空白处时，界面就会自动关闭了。

## 总结

---

这篇笔记只是较为简略地介绍了一下如何用 MVC 开发游戏，具体的实现方式还是需要各位根据源码来仔细研究。当然，项目中也有一些不规范的地方，游戏的设计上也有几个小 BUG，各位可以在学习完毕后自行修改，权当做练习。

?> 对于 MVC 框架，模型层与视图层没有直接的交互，请牢记这一点。如果想要访问数据或视图的话，需要通过控制层间接访问。