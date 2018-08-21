# UGUI基础案例

> 本文是有关于官方GUI的基础知识

UGUI 是官方的 GUI 实现，这次的话我会用一个小案例来讲解 UI 组件的用法，知识相对会基础一些，如果有什么不懂的属性就需要各位自行查文档了。

?> 在操作 UI 界面时，建议把编辑器的显示模式改成`2D`。

## 创建开始界面

---------

### 创建游戏菜单

UGUI 基于`Canvas`（画布），所有的 UI 组件都排布在画布上。新建画布，创建如下图所示的按钮：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/UGUI-Introductory/01.png)

简单的介绍一下四个组件：

* `Rect Transform`：没什么好说的，无非就是`Transform`的变种。
* `Canvas Renderer`：画布渲染相关，具体信息请查阅文档。
* `Image`：用于设置按钮的背景图片。
* `Button`：处理按钮响应事件，设置按钮响应时的各类样式。

`Button`对象下还有一个`Text`，可以修改按钮的名称。

再创建两个按钮，分别是`Save`、`Load`。之后再创建一个`Toggle`（切换选项）：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/UGUI-Introductory/03.png)

`Toggle`可以改的样式也挺多的，这里我就只改个名字，表示该选项用于控制声音的开关。

接下来再创建一个`Slider`，用于控制音量的大小：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/UGUI-Introductory/04.png)

稍微说一下，`Fill`代表是前置背景，我这里将它设置为了红色，用于标识音量的当前大小。

### 创建游戏说明

游戏说明是一大段文本，不过鉴于文本的长度，我们必须要把它做成可拖动式。创建两个`Image`组成背景，然后新建一个`Text`，适当地修改它的大小：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/UGUI-Introductory/05.png)

随便粘贴一大段文字进去，保证文本长度超出文本框的长度。为了让文本框变成可拖拽显示，我们需要新建一个`Image`，并将它的大小改成与文本框一样：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/UGUI-Introductory/06.png)

为`Image`添加`Scroll Rect`组件，将`Text`赋值给`Content`属性。另外我们得把`Horizontal`禁用，因为文本框只需要上下拖动：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/UGUI-Introductory/07.png)

不过当你运行游戏后，你发现文本框还是不能够上下拖动。解决方法很简单，修改文本框的大小，让它能够将所有文本显示出来：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/UGUI-Introductory/08.png)

虽然这么做确实可以拖动文本，但是这上下多出来的部分能不能够隐藏起来呢？当然可以，为`Image`添加`Mask`组件就能够做到这一点（如果你觉得白色的背景很难看，请把`Show Mask Graphic`禁用）：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/UGUI-Introductory/09.png)

当然，为了体贴用户，我们得为游戏说明添加一个滚动条。创建一个`Scrollbar`，修改它的位置和大小，将`Direction`改成`Bottom To Top`。由于滑块最开始是位于底部，所以得将它的值修改为`1`。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/UGUI-Introductory/10.png)

最后，将它赋值给`Scroll Rect`组件中的`Vertical Scrollbar`：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/UGUI-Introductory/11.png)

这时候你再去运行游戏，滑动条就能够运行自如了：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/UGUI-Introductory/12.png)

?> 滑动条的滑动方向一共有四种，各位可以自己去试一试。

### 场景跳转

Start Game 按钮是用来开始游戏的。当我们点击这个按钮时，游戏理应跳转到另一个场景中。`Button`组件允许我们自定义响应事件，并根据需要调用相应的方法。

创建一个空对象`GameManager`，同时为其创建脚本：

_GameManager.cs_

```csharp
public class GameManager : MonoBehaviour {

	public void OnStartGame(string sceneName) {
        Application.LoadLevel(sceneName);
    }
}
```

`OnStartGame`方法接受一个字符串，并且会根据传入的字符串加载对应场景。接下来，我们需要在按钮中添加对应的事件：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/UGUI-Introductory/13.png)

点击加号添加一个新事件，选择`GameManager`对象，方法为`OnStartGame`，传入的参数为新的场景名。另外，我们还需要在`Buliding Settings`中将两个场景加入进去：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/UGUI-Introductory/14.png)

?> 除了传入字符串，我们也可以将场景的序号传给方法。上图的两个场景的右端数字就是场景序号。

## 游戏场景

---------
我这里做了一个很简单的游戏场景，玩家可以通过一个滑动条来控制立方体的旋转速度。首先创建一个立方体，为它添加脚本：

_Cube.cs_

```csharp
public class Cube : MonoBehaviour {

	public float speed = 90;

	void Update() {
		transform.Rotate(Vector2.up * Time.deltaTime * speed);
	}

	public void ChangeSpeed(float changeSpeed) {
		speed = changeSpeed;
	}
}
```

执行该脚本，立方体会围绕着 Y 轴进行自转。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/UGUI-Introductory/15.png)

现在创建一个`Slider`，然后为它添加响应事件。这里要注意的是，我们需要选择最上面的这个方法，当滑动条的值改变时，组件会自动将参数传递给方法，不需要我们设置参数：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/UGUI-Introductory/16.png)

修改滑动条的数值范围，并将初值设为`90`：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/UGUI-Introductory/17.png)

运行游戏，查看效果：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/UGUI-Introductory/18.png)
