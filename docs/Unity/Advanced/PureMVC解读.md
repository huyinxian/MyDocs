# PureMVC解读

PureMVC 可以看做是 MVC 的加强升级版，代码的耦合性很低，但是事件的传递需要进行装箱拆箱，效率比较低。

![](http://cdn.fantasticmiao.cn/image/post/Unity/PureMVC%E8%A7%A3%E8%AF%BB/01.png)

## 框架整体分析

---

### Facade

该模块采用的是外观模式，它是整个框架的入口，用于简化内部系统的调用。我们可以在 Facade 中对模型、视图、控制器进行初始化，同时也可以对其进行操作。

### Model与Proxy

模型层采用了代理模式，Proxy 负责操作数据模型，并且与远程服务通信存取数据。这样做的好处是可以保证 Model 的独立性，方便移植。

### View与Mediator

视图层采用了中介者模式，由 Mediator 来操作具体的视图组件（ViewComponent）。Meditor 还负责添加事件监听，发送和接收 Notification 等。这样做的好处是可以把视图和操作逻辑进行分离。

### Controller与Command

控制器采用了命令模式，由于 Facade 中已经隐式创建好了控制器，因此我们只需要在 Facade 中注册所有的 Command 即可。Command 是无状态的，它只在需要时才会被创建。

控制器通过 Command 处理具体的逻辑，这一点和模型、视图是一样的。Command 可以获取 Proxy 并与之交互，发送 Notification，执行其它的 Command。如果你的系统中存在复杂的操作，比如应用程序启动和关闭，那么就应该写在 Command 里面。

### Notification与Observer

Notification 采用的是观察者模式，Proxy、Mediator、Command、Facade 都可以发送 Notification。Notification 的作用有以下几点：

* 触发 Command：Facade 中保存了 Command 与 Notification 的映射，当 Notification 被发送时，对应的 Command 将会被自动执行。Command 主要是为了实现复杂的操作，降低模型与视图的耦合。
* 通知 Mediator：Mediator 可以发送、注册、接收 Notification。每一个 Mediator 都会事先注册一组感兴趣的通知，当对应的通知发生时，Mediator 就会进行接收，并做出相应的处理。

需要注意的是，虽然 Proxy 可以发送通知，但是它不接收通知，接收通知的是 Command 和 Mediator。不理解的话可以这么想：当 Proxy 从远程服务收到数据时，它需要通知系统；当 Proxy 的数据被更改时，它也需要进行通知。但如果我们让 Proxy 接收通知，那势必就会导致它和视图、控制器进行耦合。

视图层之所以要监听 Proxy 发送的通知，是因为它的职责就是通过可视化的界面使得用户可以与模型层进行交互，但是视图不能够反过来影响模型层的数据。因此，Proxy 不能接收通知的，只能够在 Command 中进行操作。

## Notification的订阅方式

---

Notification 可以通过两种方式进行订阅，一个是直接进行订阅，第二个是与 Command 进行绑定。为什么会存在两种订阅方式呢？

上面我有说过，复杂逻辑一般由 Command 处理，比如“登录”操作就应该写在 Command 中，而“登陆成功”和“登录失败”就可以由 Mediator 进行订阅。除此之外，由于 Proxy 不接收 Command，所以需要用 Command 来执行 Proxy 中的方法修改数据。