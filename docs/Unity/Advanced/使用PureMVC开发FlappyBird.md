# 使用PureMVC开发FlappyBird

光看着也没什么意思，这次就来做一个完整的游戏吧。注意，这次 Demo 会用到一种 UI 框架，假如你没有了解过的话，可以忽略掉 UI 框架的内容，专注于 PureMVC 相关的知识。当然，对于这些额外的内容我也会一笔带过。

## 开发准备

之前我们在导入 PureMVC 框架时，是直接将源代码导入了进来。如果你并不关心具体的类是如何实现的，可以将源码打包成 `.dll` 文件然后导入。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/%E4%BD%BF%E7%94%A8PureMVC%E5%BC%80%E5%8F%91FlappyBird/01.png)

打包可以使用 Visual Studio 来做，具体步骤我就不赘述。接下来，我们需要导入 UI 框架和素材，然后基于框架创建一些窗体。如果你没了解过 UI 框架的话，你可以简单地认为我创建了三个 UI 界面。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/%E4%BD%BF%E7%94%A8PureMVC%E5%BC%80%E5%8F%91FlappyBird/04.png)

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/%E4%BD%BF%E7%94%A8PureMVC%E5%BC%80%E5%8F%91FlappyBird/06.png)

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/%E4%BD%BF%E7%94%A8PureMVC%E5%BC%80%E5%8F%91FlappyBird/07.png)

## 创建游戏界面

对于 FlappyBird 这款游戏来说，背景和地板的无限滚动很简单，主要的问题就是如何让管道随机的产生。其实说来也不难，看到下图后你大概就能明白：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/%E4%BD%BF%E7%94%A8PureMVC%E5%BC%80%E5%8F%91FlappyBird/10.png)

我们先创建三个竖向排列的管道，然后把中间的管道缩小一些，尺寸要能够容纳小鸟通过。接下来，隐藏中间管道的纹理，把它作为一个触发器，用于判断小鸟是否通过了管道。由于游戏是竖屏的，所以这种组合起来的管道其实只需要两组，原理和无限长的背景是一样的。

至于管道的随机位置就更简单了。由于我们把管道合并成了一组，所以你只需要随机改变管道组的 y 坐标就可以了。

## 开始游戏

创建一个空物体，然后给它挂上 `StartGame` 脚本：

_StartGame.cs_

```csharp
namespace PureMVCDemo
{
    public class StartGame : MonoBehaviour
    {
        void Start()
        {
            UIManager.GetInstance().ShowUIForms("StartUIForm");
        }
    }
}
```

`ShowUIForms` 是 UI 框架中的方法，作用是显示之前做好的 UI 窗体。运行程序，效果如下：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/%E4%BD%BF%E7%94%A8PureMVC%E5%BC%80%E5%8F%91FlappyBird/11.png)

未完待续...