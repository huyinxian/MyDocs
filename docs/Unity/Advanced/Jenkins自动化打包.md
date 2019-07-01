# Jenkins自动化打包

`Jenkins` 是基于 Java 开发的一种持续集成的工具，用于监控持续重复的工作。对于 Unity 开发而言，Jenkins 可以用于持续地发布测试版本，避免每次进行测试时都需要开发人员手动进行打包，大大地减少了重复性的工作。

!> 注意，只有 Untiy Pro 版本才能够与 Jenkins 交互。

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

## SVN部署流程

---

项目开发常用的版本管理工具有两个，一个是 SVN，另一个则是 GitLab。这里我以 SVN 为例来介绍一下 Jenkins 的部署流程。

### SVN服务端搭建

SVN 的服务端使用的是 `VisualSVN Server`。安装完毕后，打开 VisualSVN Server。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/16.png)

接下来，我们需要在仓库中新建一个空项目：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/17.png)

除此之外，项目还需要有参与用户。我们可以为项目组中的每个成员新建一个账户：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/18.png)

顺带一提，项目的属性中可以对项目的读写权限进行设置，各位可以根据需要自行修改：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/19.png)

### SVN客户端搭建

服务端搭建完毕后，我们需要在客户端进行 SVN 的操作。客户端部分用的是 `TortoiseSvn`，官方提供了中文语言包，如果刚上手觉得英文界面不熟悉的话可以自己安装一个。

首先，我们需要把项目从服务端上拉下来。在右键弹出的菜单中选择 `Check Out`，输入项目的地址以及存储路径：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/20.png)

接着登录我们之前创建好的账户：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/21.png)

当然，由于项目是空的，所以里面只有一些默认的文件：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/22.png)

?> SVN 的使用教程各位可以自行查阅相关资料，我这里就不作赘述了。

### Jenkins调用SVN

沿用之前创建的 Jenkins 任务，我们稍作一些修改。首先，把我们自定义的字符串参数以及工作空间路径修改成项目的存储路径，然后将命令行语句修改成以下形式：

    svn cleanup %ProjectPath%
    svn revert %ProjectPath%
    svn update %ProjectPath% --username huyinxian --password 123456

`ProjectPath` 是我们之前定义的字符串参数，这个字符串参数存储的是项目的路径。你也可以选择在命令行中手动输入路径，但为了方便起见还是建议把它写成参数的形式。上述的三行命令调用 SVN 的功能，分别是清理、撤销、更新。

注意，虽然这些语句是在 Windows 命令行中执行的，SVN 在安装时也自动对环境变量进行了配置，但我们仍需要在 Jenkins 中额外配置环境变量。进入到 Jenkins 的配置页面，添加 SVN 的路径：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/24.png)

接下来执行构建，查看输出信息：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/25.png)

## PC端打包

---

### 打包流程

在开始打包之前，首先让把项目上传至 SVN 服务端：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/26.png)

Unity 提供了相关的方法供我们直接调用项目中的函数。修改构建任务的配置参数，添加 Unity3D 的构建：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/27.png)

`Unity3d installation name` 是我们之前设置的全局工具配置，里面包含了 Unity 的安装目录。下面的一行语句代表的是 Unity 命令行参数，它们的含义如下：

* `-projectpath`：工程目录。
* `-quit`：执行完所有命令后退出程序。
* `-batchmode`：运行 Unity 时不会弹出编辑器界面。
* `-executeMethod`：执行某个类文件中的静态方法。
* `-logFile`：打印日志文件。

在打完包之后，我们还可以额外调用 WinRAR 对包体进行压缩，生成一个压缩包（如果不需要的话可以不做这一步）。为了方便 WinRAR 修改压缩包的名称，我们可以对原本的资源加载框架进行了一些小的修改：

```csharp
[MenuItem("Build/PC包")]
public static void BuildPC()
{
    // 打AB包
    BundleEditor.Build();

    // 打App前先把AB包临时拷到StreamingAssets下
    string assetBundlesPath = AppConst.AssetBundlesOutputPath + "/" + EditorUserBuildSettings.activeBuildTarget.ToString() + "/";
    Copy(assetBundlesPath, AppConst.AssetBundlesLoadingPath);

    string productName = PlayerSettings.productName;
    string dir = productName + "_" + EditorUserBuildSettings.activeBuildTarget.ToString() + string.Format("_{0:yyyy_MM_dd_HH_mm}", DateTime.Now);
    string name = string.Format("{0}.exe", productName);
    string savePath = AppConst.WindowsPath + "/" + dir + "/" + name;

    BuildPipeline.BuildPlayer(FindEnableScenes(), savePath, EditorUserBuildSettings.activeBuildTarget, BuildOptions.None);

    // 把临时拷贝的AB包删掉
    DeleteDir(AppConst.AssetBundlesLoadingPath);

    WriteBuildName(name);
}

/// <summary>
/// 将App名称写入到txt文件中，供打包机调用
/// </summary>
public static void WriteBuildName(string name)
{
    FileInfo fileInfo = new FileInfo(Application.dataPath + "/../BuildName.txt");
    StreamWriter sw = fileInfo.CreateText();
    sw.WriteLine(name);
    sw.Close();
    sw.Dispose();
}
```

上述代码会额外在项目根目录下生成一个 `BuildName` 的文本文件，WinRAR 在进行压缩时就可以从这个文件中读取出包名。

一切准备就绪，我们可以执行 Jenkins 的构建了。在正式构建之前，一定要记得把还在运行的 Unity 编辑器关闭掉。另外，工程目录一定要定位到项目的根目录下，否则 Unity 将无法查找到对应的类文件以及函数。打包成功后可以查看日志信息：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/28.png)

最终得到的包如下：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/29.png)

?> Unity 命令行参数以及 WinRAR 的压缩命令各位可以自行查阅资料。

### 归档成品

完成了 Jenkins 的自动化构建后，我们还可以对成品进行归档，以便日后进行查找。归档成品需要在任务的配置中设置成品所在的目录：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/30.png)

成品归档完成之后，可以在构建任务的主界面进行查看并下载：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/31.png)

归档的文件默认保存在 Jenkins 安装目录的 `jos\任务名\lastSuccessful\archive` 下：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/32.png)

### 添加更多的配置参数

事实上，我们在正式的开发中不可能只用到一个参数，像是版本号、构建次数、项目名称等等这些都可以作为 Jenkins 的配置参数。为了让 Unity 能够获取到 Jenkins 中设置的参数，我们需要把它们添加到命令行中：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/33.png)

Jenkins 的执行命令其实就是 Windows 的批处理命令，而 Unity 也提供了获取命令行语句的 API：

```csharp
/// <summary>
/// 设置PC的构建配置
/// </summary>
public static string SetPCBuildSetting(BuildSetting buildSetting)
{
    // 后缀名
    string suffix = string.Empty;

    if (!string.IsNullOrEmpty(buildSetting.version))
    {
        PlayerSettings.bundleVersion = buildSetting.version;
        suffix += "_" + buildSetting.version;
    }
    if (!string.IsNullOrEmpty(buildSetting.name))
    {
        PlayerSettings.productName = buildSetting.name;
    }
    if (buildSetting.isDebug)
    {
        EditorUserBuildSettings.development = true;
        EditorUserBuildSettings.connectProfiler = true;
        suffix += "_Debug";
    }
    else
    {
        EditorUserBuildSettings.development = false;
        EditorUserBuildSettings.connectProfiler = false;
    }

    return suffix;
}

/// <summary>
/// 获取PC的构建配置
/// </summary>
public static BuildSetting GetPCBuildSetting()
{
    // 将命令行的语句提取为字符串数组
    string[] parameters = Environment.GetCommandLineArgs();
    BuildSetting buildSetting = new BuildSetting();
    foreach (string str in parameters)
    {
        if (str.StartsWith("Version"))
        {
            var temp = str.Split(new string[] { "=" }, StringSplitOptions.RemoveEmptyEntries);
            if (temp.Length == 2)
            {
                buildSetting.version = temp[1].Trim();
            }
        }
        else if (str.StartsWith("Name"))
        {
            var temp = str.Split(new string[] { "=" }, StringSplitOptions.RemoveEmptyEntries);
            if (temp.Length == 2)
            {
                buildSetting.name = temp[1].Trim();
            }
        }
        else if (str.StartsWith("Debug"))
        {
            var temp = str.Split(new string[] { "=" }, StringSplitOptions.RemoveEmptyEntries);
            if (temp.Length == 2)
            {
                bool.TryParse(temp[1], out buildSetting.isDebug);
            }
        }
    }

    return buildSetting;
}

public class BuildSetting
{
    public string version = "";
    public string name = "";
    public bool isDebug = false;
}
```

完成编辑器脚本的修改之后，我们再进行 Jenkins 构建。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/34.png)

最终打包得到的文件会包含对应的版本号，并且当 Untiy 处于 Debug 模式构建时，包名上也会进行标注。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/35.png)

### 配置表备份

Jenkins 的每个任务都会有一个对应的配置表，里面记录了该任务的相关设置。配置信息可以在 Jenkins 安装目录的 `jos\任务名\config.xml` 中找到，当你需要在其它设备上建立相同任务时就可以替换该任务的配置表文件。

## 安卓端打包

---

新建一个安卓打包任务，配置和 PC 打包基本差不多，只需要稍微改动一下调用的函数名称即可。由于安卓打包得到的是 APK，所以这里就不再对其进行压缩了。

### 配置环境

打安卓平台的包需要先在 Unity 中配置好 SDK 和 JDK，然后再生成打包所需的密钥。生成密钥可以在任意位置调用命令行，然后输入以下命令：

```
keytool -genkey -alias android.keystore -keyalg RSA -validity 36500 -keystore loadassetframework.keystore
```

根据命令行提示的语句依次输入信息，完成后将在命令行调用的文件夹位置生成一个 `.keystore` 的密钥文件，然后我们就可以在 Unity 的 PlayerSettings 中设置密钥。

不过这样有一个问题，我们每次在打安卓包时，都需要在 PlayerSettings 中输入密钥的密码，所以我们得把这一工作放到代码中来做：

```csharp
PlayerSettings.Android.keystorePass = "123456";
PlayerSettings.Android.keyaliasPass = "123456";
```

除此之外，你也可以给安卓打包添加其它的参数，比如发布渠道、是否使用多线程渲染、是否使用 IL2CPP 编译等等。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/36.png)

### 安卓打包的几个问题

**1.Unable to locate Android SDK**

这个问题就是没有在 Jenkins 中设置 SDK 的环境变量，只需要在系统设置中补上即可：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/37.png)

**2.Validating Project structure**

当使用访客账号登录时会缺少权限，需要为 Jenkins 服务指定登录账号。打开服务，找到 Jenkins：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/38.png)

把登录电脑的账户填进去：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/Jenkins%E8%87%AA%E5%8A%A8%E5%8C%96%E6%89%93%E5%8C%85/39.png)

?> 我在使用 Jenkins 打包时也会有这个提示，但是整体的打包流程并没有受到影响，个人猜测可能是最新的版本进行了修复。