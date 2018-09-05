# 初识PureMVC

在俄罗斯方块的案例中，我简单地介绍了一下 MVC 的组织结构。但事实上，这只是对于 MVC 这一概念的粗浅应用，我们并没有用到开发框架。如何更加有效且正式地使用开发框架，是我们接下来所需要学习的。

## 为什么需要开发框架

---

游戏开发为什么会需要框架呢？假如你是从我的基础入门笔记一路看过来的，并且有动手去实现每一个游戏案例，那么你肯定会在开发的过程中遇到各种各样的问题，其中最明显的一个问题就是代码组织结构混乱。在 Unity 中，我们可以往物体上面挂脚本，这种方法简单有效，但当物体一旦多起来时，代码结构就会变得混乱，修改代码时往往会显得手足无措，不知从何改起。

除此之外，在实际的游戏开发过程中，项目的需求往往是在不断变动的。为了适应产品部门的“脑洞”，程序部门在编写代码前必须设立好框架，把软件的各部分进行分离，每一部分完成某一功能。

打个简单的比方，假如我们之前做的俄罗斯方块改需求了，要从 2D 游戏变为 3D 游戏，那么你是不是又要重新把代码写一遍呢？其实不然，假如你按照 MVC 的概念严格划分了代码结构，那么模型层的代码其实是不用修改的，你只需要修改控制层和视图层即可。

## 市面上的开发框架

---

其实在之前的开发中，我们就已经用到了开发框架，它们分别是：EmptyGO（空对象管理）、Simple GameManager（单例游戏管理）、Manager Of Managers（管理器）。

所谓的空对象管理就是创建一个空对象，然后把相关的子物体都挂在空物体下。这其实是 Unity 官方鼓励我们做的，为的就是合理地管理物体。此外，单例游戏管理和管理器我们也用的很多，比如我们经常创建的 GameManager、AudioManager、MapManager 等等。虽然这些管理器很好地拆分了游戏的功能，但由于管理器并没有一种统一的开发模式，从而导致每个公司的用法都不一样，使得我们每到一个新公司都要重新熟悉该公司的开发方式。

那么有没有统一的开发框架呢？当然有，使用 MVC 结构的开源框架是 PureMVC，而使用 MVCS 结构的开源框架则是 StrangeIOC。另外，还有使用 MVVM 结构的开源框架 uFrame，不过我这里不会讲。

## HelloPureMVC

---

还是按照老样子，我们先写个 Demo 来使用 PureMVC，至于具体的框架结构我们放到之后再说。PureMVC 框架在 GitHub 上有开源项目，我们可以下载标准版本，然后将 Core、Interfaces、Patterns 三个文件夹导入到工程中。如果不了解导入方法，可以自行查找教程。

我们需要在项目中定义六个脚本，它们的作用如下：

* **ApplicationFacade**：全局管理类。
* **MyData**：模型层，用于存储数据。
* **DataProxy**：模型层代理类，用于数据的操作。
* **DataCommand**：控制层，处理与用户的交互。
* **DataMediator**：视图层，显示玩家的界面。
* **StartGame**：游戏的入口。

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/%E5%88%9D%E8%AF%86PureMVC/01.png)

首先是模型层。由于我们这个程序很简单，当用户点击按钮后数字增加，所以模型层就只有一个数据：

_MyData.cs_

```csharp
namespace PureMVCDemo
{
    public class MyData
    {
        private int number = 0;

        public int Number
        {
            get
            {
                return number;
            }
            set
            {
                number = value;
            }
        }
    }
}
```

模型层有了，为了获取数据，我们需要写一个模型层的代理类。注意，继承自 PureMVC 相关的类需要引入命名空间：

_DataProxy.cs_

```csharp
using PureMVC.Patterns;

namespace PureMVCDemo
{
    public class DataProxy : Proxy
    {
        /// <summary>
        /// 声明本类的名称
        /// </summary>
        public new const string NAME = "DataProxy";

        /// <summary>
        /// 引用的实体类
        /// </summary>
        private MyData myData = null;

        public DataProxy() : base(NAME)
        {
            myData = new MyData();
        }

        /// <summary>
        /// 增加数字
        /// </summary>
        /// <param name="addNumber"></param>
        public void AddNumber(int addNumber)
        {
            myData.Number += addNumber;

            // 将变化后的数据发送给视图层
            SendNotification("Msg_AddNumber", myData);
        }
    }
}
```

`NAME` 属性在定义时使用了 `new` 关键字，这是为了覆盖基类中的显示变量声明。我们在代理类中初始化了一个模型层的实体类，然后为其编写了 `AddNumber` 方法。接下来，由于方法调用时发送了消息给视图层，那么我们就得在视图层中接收相关消息：

_DataMediator.cs_

```csharp
namespace PureMVCDemo
{
    public class DataMediator : Mediator
    {
        public new const string NAME = "DataMediator";

        private Text text;
        private Button button;

        /// <summary>
        /// 构造函数
        /// </summary>
        /// <param name="goRootNode">UI界面的根节点</param>
        public DataMediator(GameObject goRootNode)
        {
            text = goRootNode.transform.Find("Text").gameObject.GetComponent<Text>();
            button = goRootNode.transform.Find("Button").gameObject.GetComponent<Button>();

            // 注册按钮的响应事件
            button.onClick.AddListener(OnButtonClick);
        }

        /// <summary>
        /// 按钮点击事件
        /// </summary>
        private void OnButtonClick()
        {
            // 定义消息，发送给控制层
            SendNotification("Reg_StartDataCommand");
        }

        /// <summary>
        /// 定义视图层允许接受的消息
        /// </summary>
        /// <returns></returns>
        public override IList<string> ListNotificationInterests()
        {
            IList<string> listResult = new List<string>();

            // 可以接收的消息
            listResult.Add("Msg_AddNumber");

            return listResult;
        }

        /// <summary>
        /// 处理接收到的消息
        /// </summary>
        /// <param name="notification"></param>
        public override void HandleNotification(PureMVC.Interfaces.INotification notification)
        {
            switch (notification.Name)
            {
                case "Msg_AddNumber":
                    // 把模型层发来的数据显示到文本框
                    MyData myData = notification.Body as MyData;
                    text.text = myData.Number.ToString();
                    break;
                default:
                    break;
            }
        }
    }
}
```

视图层在初始化时需要传入 UI 界面的根节点，用以获取 UI 控件。按钮注册了一个响应事件，当用户点击按钮时，会给控制层发送一个消息。至于这个消息干什么用，我们暂时先不用管。`ListNotificationInterests` 和 `HandleNotification` 都是重写了基类中的虚函数，分别用于定义视图层允许接受的消息以及消息的处理。注释已经说的很明白了，我这里就不细说了。

视图层弄完了，接下来开始写控制层：

_DataCommand.cs_

```csharp
namespace PureMVCDemo
{
    public class DataCommand : SimpleCommand
    {
        /// <summary>
        /// 执行方法
        /// </summary>
        /// <param name="notification"></param>
        public override void Execute(PureMVC.Interfaces.INotification notification)
        {
            // 调用模型层的AddNumber的方法
            DataProxy data = (DataProxy)Facade.RetrieveProxy(DataProxy.NAME);
            data.AddNumber(10);
        }
    }
}
```

在控制层中，`Execute` 方法会通过模型层代理类的名称获取到实例，然后调用 `AddNumber` 方法来增加数字。可能有人会问，这个方法什么时候会被调用呢？别急，接着往下看：

_ApplicationFacade.cs_

```csharp
namespace PureMVCDemo
{
    public class ApplicationFacade : Facade
    {
        public ApplicationFacade(GameObject goRootNode)
        {
            // MVC三层的关联绑定
            // 控制层注册，并将某个消息与控制层绑定。当该消息调用时，控制层中的Execute方法将被调用
            RegisterCommand("Reg_StartDataCommand", typeof(DataCommand));

            // 视图层注册
            RegisterMediator(new DataMediator(goRootNode));

            // 模型层注册
            RegisterProxy(new DataProxy());
        }
    }
}
```

在全局控制类中，我们注册了 MVC 的三层。需要注意的是，控制层在注册时，必须要给它绑定一个消息。当我们在某个位置发送该消息时，就会调用控制层中 `Execute` 方法。显然，我们需要在按钮按下时发送这条消息。

最后，为了让游戏运行起来，还得编写游戏入口文件：

_StartGame.cs_

```csharp
namespace PureMVCDemo
{
    public class StartGame : MonoBehaviour
    {
        void Start()
        {
            // this.gameObject表示UI层的根节点
            new ApplicationFacade(this.gameObject);
        }
    }
}
```

我们可以直接把入口文件挂在 `Canvas` 上，这样就能够自动传入 UI 层的根节点。

运行起来，效果如下：

![](http://obkyr9y96.bkt.clouddn.com/image/post/U3D/%E5%88%9D%E8%AF%86PureMVC/02.png)

## 总结

---

虽说是一个很简单的程序，但我们依旧需要编写多个脚本，这也是 PureMVC 的缺点之一。在上述案例中，程序的运行流程可简单地归为以下步骤：

* 游戏开始，调用全局控制类。
* 在全局控制类中注册 MVC 三层。
* 当玩家点下按钮时，会发送一个与控制层绑定的消息，从而调用控制层中的 `Execute` 方法。
* 控制层调用模型层代理类的方法 `AddNumber`，修改模型层的数据。`AddNumber` 会把修改完的数据发送给视图层。
* 视图层中已经定义了可接收的消息类型，会根据传来的数据修改 UI。

当然，或许你还有许多不明白的 API，但就目前来说你只需要明白 PureMVC 的基本流程即可。