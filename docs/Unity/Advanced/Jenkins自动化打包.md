# Jenkins自动化打包

`Jenkins` 是基于 Java 开发的一种持续集成的工具，用于监控持续重复的工作。对于 Unity 开发而言，Jenkins 可以用于持续地发布测试版本，避免每次进行测试时都需要开发人员手动进行打包，大大地减少了重复性的工作。

## Jenkins 环境配置

---

### 安装Jenkins

首先，我们需要在 Jenkins 官网上下载对应平台的安装包。安装完成之后，在浏览器中输入默认的 `http://localhost:8080/` 地址即可进入到 Jenkins 的插件安装界面。初次使用时可以直接选择安装 Jenkins 推荐的插件，之后便是等待所有插件安装完成：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/01.png)

安装完成之后，我们可以新建一个管理员用于登录系统。创建完成之后，我们还需要设置 Jenkins 的后台访问地址，这样一来初始化配置就完成了。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/02.png)

输入后台地址，就能够看到 Jenkins 的界面了：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/03.png)

在正式进行开发前，我们还需要进行一些配置操作。点开 `Manage Jenkins` 选项，我们就能够看到平台的相关设置。

### 全局工具配置

这一部分主要是对 JDK、Git 等工具进行配置，如果各位在之前没有安装过这些工具，那么可以自行搜索对应的网站进行安装。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/04.png)

### 插件管理

之前在初始化阶段，我们曾安装过一系列的插件。由于此篇笔记针对的是 Unity 自动化打包，因此我们还需要安装 Unity 相关的插件。首先，打开插件管理界面：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/05.png)

搜索插件 `Unity3D`，点击进行安装，安装完毕后重启 Jenkins 即可。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/06.png)

安装完插件，我们还需要在全局工具配置里面添加 Unity 的安装目录：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/07.png)

## 构建测试项目

---

### 项目设置

接下来，我们就尝试构建一个简单的项目，熟悉一下 Jenkins 的整体流程。首先，新建一个自由风格的任务。任务可供设置的选项有很多，在这里我们只关注以下两项：

* `Discard old builds`：设置老项目的保存天数以及项目的保存个数上限。
* `This project is parameterized`：设置项目参数，这里我选择的参数类型是字符串。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/08.png)

点开高级选项，设置项目使用的自定义工作空间：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/09.png)

由于这只是一个用于熟悉 Jenkins 流程的测试项目，所以这里就不使用版本管理工具了。接下来，我们可以为任务构建一个触发器，这个触发器可以是在某个项目构建后触发，或者是通过某个脚本触发。当然，自动化打包中用得最多的就是定时触发，它可以在指定的时间进行更新（相关的语法可以点击选项后的问号查看）：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/10.png)

最后，我们可以设置项目的构建方式以及构建后的操作。前者可以调用相关插件进行项目构建，比如 Unity、Python、Windows 等等；后者则是用于设置项目构建后的操作，例如将项目归档存储、构建完该项目之后继续构建其他项目、发送通知邮件、在 Github 上进行提交、删除构建等等。

为了进行测试，在这里我们调用 Windows 的命令行打印一条消息：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/11.png)

### 项目构建

项目创建完之后，我们查看刚刚创建好的项目，然后点击 `Build with Parameters`。界面中显示的是我们之前填写的字符串参数，而项目也将用这些参数进行构建：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/12.png)

当然，由于这个项目仅仅只是用于测试的，因此不会产生任何的变化。点击构建历史中的对应条目，即可查看之前的构建操作是否成功以及花费的时间：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/13.png)

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/14.png)

点击控制台输出，我们可以看到项目构建完成之后打印出了一段信息：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/15.png)