# NGUI的使用

这一章将介绍一下 NGUI 各项功能的使用方法。

## 标签

---

如果需要显示文字，你可以使用标签。

### 创建标签

首先把已经做好的背景拖动到 Hierarchy 窗口，这时候 NGUI 就会自动创建一个 `UI Root`，在它的下面还有一个相机和背景图。UI Root 的作用可以简单的理解为 UGUI 中的画布，而相机则是专门用来渲染 UI 的。

创建标签的步骤很简单，首先选中 UI Root，然后在场景视图中右键选择 `Create` -> `Label`。这样，我们就能够创建一个标签了。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/01.png)

!> 步骤不要乱，如果你没有选中 UI Root，那么你点击右键是没有反应的。NGUI 毕竟只是插件，就操作性来说没有 UGUI 方便。

### 字体文件

标签的属性栏中有着一系列自定义组件，虽然这些组件与官方的文本框有着差别，但我们基本上还是能够猜出它们的作用。`UI Label` 用于控制标签的功能，类似的还有 `UI Button` 等等。

我们看到第一行有着 `Unity` 以及 `Font` 选项，后面的则是一个字体文件。第一个选项有 `Unity` 和 `NGUI` 两种，选择 Unity 的话则是会搜索 Untiy 自带的字体，还有就是我们导入的字体。这种字体是动态字体，格式为 `.ttf`，像是 Windows 系统就是使用的这种字体。

如果你选择 NGUI 的话，那么搜索出来的字体就是 NGUI 自带的字体，以及你使用 `Font Maker` 制作的字体（这种工具之后会说）。这种字体是静态字体，以预制体的形式存储。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/03.png)

?> 如果没有显示出字体，那你可以点击 Show All 刷新一下。

### 标签参数

参数说明如下：

* **Modifier**：字母的大小写设置。
* **Overflow**：当文本内容超出文本框时的处理方式。
* **Alignment**：文字对齐方式。
* **Spacing**：调整文字间距。
* **Gradient**：可以设置文字上下的两种颜色。
* **Effect**：可以设置文字的阴影、描边等效果。

这里说一下 Overflow，它一共有四种方式：`Shrink Content`、`Clamp Content`、`Resize Freely`、`Resize Height`。

Shrink Content 可以自由地调节文本框的大小，如果文本内容超出文本框，那么字体会相应的缩小。

Clamp Content 也能够调节文本框大小，但是超出的文本内容会被直接裁掉。

Resize Freely 会锁定文本框大小，并自动将它调节至与文本内容的大小相符合的程度。你可以为它设置最大高度与宽度。

Resize Height 会锁定文本框的高度，但是你可以调节它的宽度。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/02.png)

## 精灵

---

创建一个精灵，点击 `Atlas` 属性，选择你想要的图集：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/04.png)

NGUI 的图集是以预制体的形式存储的，你也可以使用 `Atlas Maker` 来创建属于自己的图集。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/05.png)

精灵的属性不多，可以自己去试一试，比如 `Flip` 是控制精灵的翻转。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/06.png)

## 面板

---

Panel 是一种容器，可以用于 UI 的分层。创建一个 Panel，然后为它创建精灵和标签两个子对象。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/07.png)

正如之前所说，Panel 可以控制其下所有 Widget 的透明度。`Depth` 用于设置其渲染顺序，`Clipping` 则是用于控制面板的显示区域。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/08.png)

Clipping 有三个额外的属性：

* **Texture Mask**：可以为面板蒙上一层材质。
* **Soft Clip**：则是让面板只显示方框内的内容。
* **Constrain But Bont Clip**：与第二种类似，但是不会裁切掉方框外的内容。

至于其他的属性这里暂时不讲。

## 按钮

---

按钮是 UI 中很常用的组件，不管是场景的跳转还是事件的触发都需要用到按钮。

### 标签按钮

NGUI 的按钮比较独特，它并不是一个单独出来的控件，你可以把一个 Label 或者 Sprite 做成按钮。首先创建一个标签，然后右键选择 `Attach` -> `Box Collider`。

按钮的 Collider 并不是用来做碰撞检测的，而是为了判断鼠标是否点中了按钮。再次右键选择 `Attach` -> `Button Scirpt`，这样你就创建出了一个按钮。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/09.png)

UIButton 有这么几个属性：

* **Tween Target**：按钮控件所作用的游戏物体。
* **Color**：按钮在四种状态下的颜色。
* **OnClick**：点击按钮所触发的事件。

按钮的四种状态分别为：`Normal` 正常显示、`Hover` 鼠标放在按钮上、`Pressed` 按钮被按下、`Disabled` 按钮被禁用。

运行效果如下：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/10.png)

### 精灵按钮

当然，你也可以用精灵来创建按钮，步骤与标签类似。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/12.png)

精灵按钮可以设置它在四种状态下的精灵图片，比如我用 emoji 标签做了一个按钮，正常时它显示微笑标签，鼠标放上去时显示惊讶表情，按钮按下时显示愤怒表情。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/11.png)

### 组合按钮

一般按钮都是由文字和底框组合起来的，创建方法也很简单，就是先创建精灵按钮，然后为它添加一个子标签。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/13.png)

不过当你按下按钮的时候，只有按钮的底框会变色，标签是保持不变的。想要让标签跟着变色的话，你需要为精灵再添加一个 UIButton 组件（必须是精灵，因为标签没有 Collider）。当然，如果按照 `Attach` -> `Button Scirpt` 是无法再次添加组件的，你必须在属性栏中搜索 `Button` 然后添加。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/15.png)

?> 如果你想为某个控件添加子控件，可以选中它然后创建你想要的控件，这时就会多出 `Child` 和 `Sibling` 两种选项，分别为子对象和同级对象。

### 按钮事件

创建一个脚本，随便选择一个物体挂上，然后添加代码：

_NGUIButton_

```csharp
void OnLabelButtonClick() {
    Debug.Log("Click Label Button");
}

void OnSpriteButtonClick() {
    Debug.Log("Click Sprite Button");
}
```

之后的操作方法与 UGUI 类似，将挂载了脚本的物体拖拽至 `Notify`，然后选择你想要调用的方法：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/16.png)

## 图集

---

在介绍其他控件之前，先来学一学怎么制作属于自己的图集。

### 制作图集

NGUI 内置了 `Atlas Maker`，我们可以在菜单栏中选择 `NGUI` -> `Open` -> `Atlas Maker` 打开它。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/17.png)

选择 `New`，然后在 Project 窗口中选择你想要打包成图集的图片。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/18.png)

看到图片右边的 `Add` 了吗？这表示精灵是新添加到图集中的。点击 `Create`，稍等片刻后就能够看到打包好的图集：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/19.png)

第一个是材质球；第二个是预制体，也就是我们需要用到的图集；第三个是将所有精灵汇合到一起的大图。

既然已经创建了第一个属于我们的图集，那么不妨用它来创建一个按钮吧：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/21.png)

### 修改图集

如果你想要删除某些图片，可以再次打开 Atlas Maker，选中我们的图集，然后点击对应图片的删除按钮。但如果你想要更换图集中的某些图片，你无须删除旧图片，只需将新的图片名改成和旧图片一样，然后选中它：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/22.png)

Atlas Maker 会自动检测图集中的图片，如果图集中没有就会添加它，如果重名了则会替换旧图片。

## 九宫格切图

---

借用上次自定义的按钮，我们将它进行拉伸：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/25.png)

由于按钮放大了很多倍，所以它的边框在我们看来是很模糊的。如果我们不希望按钮在放大后产生这种效果，就有必要用到九宫格切图。

将精灵的 `Type` 属性改为 `Sliced`，然后编辑精灵：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/26.png)

编辑选项有三个属性，分别为 `Dimensions`、`Border`、`Padding`。Dimensions 是精灵的尺寸，不要修改；Padding 表示的是与精灵边框的间距；而 Border 则是裁切的选项。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/27.png)

修改这四个值，裁切按钮的四个角：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/28.png)

这样一来，按钮的边框就不会再放大了：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/29.png)

为什么叫做九宫格切图呢？因为在切图的时候，上下左右四根线刚好能够把图片分成九个区域，而只有中间的区域才会进行缩放，边框区域是不会进行缩放的。对于游戏中的圆角等按钮，九宫格切图能够让它们的边框不失真。

## 字体

---

字体的属性选项有两种，一种属于 Unity，格式一般为 `.ttf`，可以直接使用系统的字体；另一种则是属于 NGUI，使用预制体或者图集存储。如果你想要创建自己的字体，NGUI 提供了 `Font Maker` 供你使用。

### 创建动态字体

打开 Font Maker，选择 `Dynamic`，然后点击 `Source` 选择你想要创建的动态字体。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/30.png)

创建成功后会生成一个预制体，如果要使用的话就需要将字体属性切换至 NGUI，然后选择你刚刚创建的字体：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/31.png)

?> Font Maker 是可以用来创建动态字体的，不过一般来说是应该直接使用 Unity 的字体，这里只是做一个演示。

## Widget（小部件）

---

每个 NGUI 组件都有一个 Widget 属性。它相当于一个容器，用于管理控件的位置和大小。

创建一个精灵，在 UISprite 组件中可以看到 Widget 属性：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/32.png)

`Pivot` 是控件的中心点，六个选项从左至右分别是：左、水平居中、右、上、垂直居中、下。中心点代表的是控件的位置，如果它的中心点不同，坐标其实是不一样的（哪怕控件的位置看起来是一样的）：

`Depth` 代表控件的深度，之前已经讲过，Widget 的权级是要低于 Panel 的。对于处于同一 Panel 下的两个控件，Widget 属性中的 Depth 必须是不同的（相同时会有警告），这一点对于 Panel 也是适用的。

`Size` 可以修改控件的长宽。

`Aspect` 代表长宽比，一共有三个选项：`Free` 模式下长宽比取决于具体的 Size，不可手动修改；`Based On Width` 模式下不可以修改高度，只能通过长宽比和宽度来改变高度；`Based On Height` 与第二种情况是相反的。

## 锚点

---

锚点（Anchor）用于 UI 界面的屏幕适配。就其字面意思来说，锚点相当于某个控件抛下的锚，将自己牢牢地固定在 UI 界面这片“大海”上。

不使用锚点的结果是显而易见的，比如你随意地将游戏窗口缩小，你就会发现界面的上的控件一个个地跑到了屏幕外面了。至于原因其实也很简单，因为控件的位置是相对于 UIRoot 而言的，比如一个位置在 (100, 100) 的控件，它距离中心点的坐标始终是不变的。当游戏窗口缩小时，它为了保持这个坐标，就不得不跑到屏幕外面去了。

### 制作背景图

想要使用锚点很简单。创建一个精灵，它的属性栏中就有 `Anchors` 属性：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/33.png)

假设这是一张背景图，那么为了让背景铺满整个摄像机，一般来说有两种做法。第一种是把背景图做大一些，保证摄像机的尺寸不会超过背景图。这一种做法的好处是不会让背景的比例失衡，不过我们这里介绍的是另外一种做法。

将锚点的 `Type` 属性设置为 `Unified`，属性栏变成了下面这样：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/34.png)

`Target` 代表的是参照物，为了让背景图与 UI 界面一样大，锚点的参照物就选择为 UIRoot。接下来的四个属性其实比较相似，分别代表了精灵的上下左右边框距离参照物中心点（Taget's Center）的距离。这个时候如果你再拖动游戏窗口，背景图就始终是位于中心点的，并且尺寸是保持不变。

为什么呢？很简单，决定图片大小的其实就是四个边框的间距，既然它们和中心点的距离保持不变，那么背景图的尺寸当然不会变。

说回正题，如何让背景图的尺寸与 UI 界面保持同步呢？试着把上下左右边框的设置改为下面这样：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/35.png)

这样一看似乎有道理，毕竟背景图的边框与 UI 界面的距离不变，想必也会随着相机的尺寸进行变化吧？

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/36.png)

喔，似乎有哪里出现问题了？其实仔细想想就能够明白，为了让背景图与相机的距离保持不变，那么当相机尺寸变化时，势必就要影响背景图的尺寸。正确的做法应该是这样：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/37.png)

非常好，效果达到了。

### 制作小地图

有了背景图，接下来试着做一个小地图吧：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/38.png)

小地图应该始终位于右上角，并且尺寸保持不变。为了达到这一点，那么左右边框距离参考物右边框的距离应该不变，上下边框距离参考物上边框的距离应该不变。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/39.png)

效果也是很棒的。

### 制作其他UI

接下来再做一个进度条吧。

进度条是位于窗口中心的，它的长度应该保持不变，且距离相机下边框的距离也不变：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/40.png)

效果如下：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/41.png)

### 高级模式

到此为止，锚点的基本使用想必你已经掌握了。其实除了 Unified，锚点还有 `Advanced`（高级）模式：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/NGUI%E5%9F%BA%E7%A1%80/42.png)

看到了吗？每一个边框都能够设置自己的参照物，对于复杂的 UI 界面应该是足以胜任了。