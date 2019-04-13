# ToLua示例

本篇笔记将着重介绍一下比较流行的热更新方案 ToLua，并编写相关示例以供熟悉框架。

## ToLua简介与安装

---

目前主流的有三种热更新方式：

* Lua：使用 Lua 进行热更新就是在 Unity 中内嵌一个 Lua 虚拟机，那些需要经常变动并且对于执行效率没有要求的逻辑可以用 Lua 实现。Lua 代码属于文本资源，并且是运行时编译的，因此可以在绝大多数平台上实现热更新。
* C#Light：这个是一个简单的嵌入脚本，模仿 C# 的语法风格。
* C#反射：使用反射的话需要把部分逻辑封装成 DLL，然后将 DLL 文件打成 AssetBundle 包。运行时需要用 C# 反射加载程序集，动态地从 AssetBundle 中加载脚本。由于 IOS 禁止 JIT（即时编译），因此该方法无法在 IOS 使用。DLL 热更部分我会单独写一篇笔记介绍。

介绍完了主流热更方式，下面就该轮到我们的主角 ToLua 上场了。ToLua 使用的是静态绑定，框架可以自动生成用于在 Lua 中访问 Unity 的绑定代码，并且会把 C# 的常量、函数、属性、类、枚举等暴露给 Lua。

ToLua 的下载地址：[GitHub - topameng/tolua: The fastest unity lua binding solution](https://github.com/topameng/tolua)。

安装的话只需要把 `Assets`、`Unity5.x`、`Luajit64`、`Luajit` 这四个文件夹复制到我们的工程目录即可。