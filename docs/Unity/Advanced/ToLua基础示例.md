# ToLua基础示例

本篇笔记将着重介绍一下比较流行的热更新方案 ToLua，并编写相关示例以供熟悉框架。

## ToLua简介与安装

---

目前主流的有三种热更新方式：

* Lua：使用 Lua 进行热更新就是在 Unity 中内嵌一个 Lua 虚拟机，那些需要经常变动并且对于执行效率没有要求的逻辑可以用 Lua 实现。Lua 代码属于文本资源，并且是运行时编译的，因此可以在绝大多数平台上实现热更新。
* C# Light：这个是一个简单的嵌入脚本，模仿 C# 的语法风格。
* C# 反射：使用反射的话需要把部分逻辑封装成 DLL，然后将 DLL 文件打成 AssetBundle 包。运行时需要用 C# 反射加载程序集，动态地从 AssetBundle 中加载脚本。由于 IOS 禁止 JIT（即时编译），因此该方法无法在 IOS 使用。DLL 热更部分我会单独写一篇笔记介绍。

介绍完了主流热更方式，下面就该轮到我们的主角 ToLua 上场了。**ToLua 使用的是静态绑定，而非反射**。框架可以自动生成用于在 Lua 中访问 Unity 的静态绑定代码（Wrap 文件），并且会把 C# 的常量、函数、属性、类、枚举等暴露给 Lua。

ToLua 的下载地址：[GitHub - topameng/tolua: The fastest unity lua binding solution](https://github.com/topameng/tolua)，这里面还有很多实用的工具，比如 Debug 插件、针对 lua 的 protoc 编译器、性能分析工具等等。

安装的话只需要把 `Assets`、`Unity5.x`、`Luajit64`、`Luajit` 这四个文件夹复制到我们的工程目录即可。加载完毕后可能会弹出一个提示框，这时候我们直接点确定让 ToLua 自动生成常用类型注册文件即可。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/ToLua%E5%9F%BA%E7%A1%80%E7%A4%BA%E4%BE%8B/01.png)

## 基础示例

---

下面给各位演示一下 ToLua 框架是如何与 Unity 进行交互的。首先，我们先编写一个 lua 脚本，用于控制小球的移动：

```lua
Control = {}    -- 定义一个空类
local this = Control
local GameObject = UnityEngine.GameObject   -- 调用UnityEngine的类
local Input = UnityEngine.Input
local AudioSource = UnityEngine.AudioSource
local Rigidbody = UnityEngine.Rigidbody
local Color = UnityEngine.Color
local Sphere
local rigid
local force

function this:Start()
    -- 查找场景中的小球，为它加上刚体等组件
    Sphere = GameObject.Find("Sphere")
    Sphere:GetComponent("Renderer").material.color = Color(1, 0.1, 1)
    Sphere:AddComponent(typeof(AudioSource))
    rigid = Sphere:AddComponent(typeof(Rigidbody))
    force = 5
end

function this:Update()
    -- 读取输入，控制小球的位移
    local h = Input.GetAxis("Horizontal")
    local v = Input.GetAxis("Vertical")
    rigid:AddForce(Vector3(h, 0, v) * force)
end
```

Unity 的常用接口已经被 ToLua 框架转换成了静态绑定文件（Wrap 文件），我们在编写 lua 文件时可以直接进行调用。脚本代码写完之后，我们还需要编写一个 C# 脚本用于启动 lua 虚拟机：

```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using LuaInterface;

public class Control : MonoBehaviour
{
    LuaState luaState = null;
    LuaFunction startUpFunc = null;

    private void Start()
    {
        // 加载lua文件
        new LuaResLoader();
        // 定义lua虚拟机并初始化
        luaState = new LuaState();
        luaState.Start();
        // 如果调用的lua文件需要使用Unity的接口，就必须要在LuaState中注册
        LuaBinder.Bind(luaState);
        // 标识lua文件的存放路径
        string luaPath = Application.dataPath + "/ResourceDynamic/Lua";
        luaState.AddSearchPath(luaPath);
        // DoFile加载文件，Require加载模块
        luaState.DoFile("Control.lua");
        CallFunc("Control.Start", gameObject);
    }

    private void Update()
    {
        CallFunc("Control.Update", gameObject);
    }

    private void OnApplicationQuit()
    {
        // 释放内存
        luaState.Dispose();
        luaState = null;
    }

    private void CallFunc(string func, GameObject go)
    {
        // 获取lua文件中的对应方法，并进行调用
        startUpFunc = luaState.GetFunction(func);
        startUpFunc.Call(go);
        startUpFunc.Dispose();
        startUpFunc = null;
    }
}
```

把这个脚本挂到小球上，你就可以对小球进行操控了。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/ToLua%E5%9F%BA%E7%A1%80%E7%A4%BA%E4%BE%8B/02.png)

## ToLua目录结构

---

### Edtior

在 `Editor/Custom` 下有一个 `CustomSettings` 文件，里面的代码大多是类似这样的：

```csharp
_GT(typeof(Behaviour)),
_GT(typeof(MonoBehaviour)),        
_GT(typeof(GameObject)),
_GT(typeof(TrackedReference)),
_GT(typeof(Application)),
_GT(typeof(Physics)),
_GT(typeof(Collider)),
_GT(typeof(Time)),        
_GT(typeof(Texture)),
_GT(typeof(Texture2D)),
_GT(typeof(Shader)),        
_GT(typeof(Renderer)),
_GT(typeof(WWW)),
_GT(typeof(Screen)),        
_GT(typeof(CameraClearFlags)),
_GT(typeof(AudioClip)),        
_GT(typeof(AssetBundle)),
_GT(typeof(ParticleSystem)),
_GT(typeof(AsyncOperation)).SetBaseType(typeof(System.Object)),        
_GT(typeof(LightType)),
_GT(typeof(SleepTimeout)),
```

这些代码主要用于标识哪些 C# 类是静态类、哪些是需要注册到 lua 的类型等等，作者在代码中都有注释标明。我们在编写项目时经常会需要在 lua 脚本中调用 C# 的接口，这时候你就可以把自己写的类放到这里面，然后使用框架的自动生成功能进行编译。编译完成后，我们就可以在 lua 中调用已经注册好的类。

### Source

`Source` 文件夹下有 `Generate` 和 `LuaConst`。LuaConst 文件主要用于配置 lua 相关路径。Generate 文件夹中主要存放 ToLua 框架导出的静态绑定代码，也就是 `Wrap` 文件：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/ToLua%E5%9F%BA%E7%A1%80%E7%A4%BA%E4%BE%8B/03.png)

如果 C# 文件发生了更改，或者在 CustomSettings 中添加了新的类，那么你就需要重新生成 Wrap 文件：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/ToLua%E5%9F%BA%E7%A1%80%E7%A4%BA%E4%BE%8B/04.png)

### ToLua

ToLua 中有以下文件：

* BaseType：基础类型的绑定代码（Coroutine、Type、Object、String 等）。
* Core：框架核心功能，包括 `LuaFunction`、`LuaTable`、`LuaThread`、`LuaState`、`LuaEvent` 等。
* Examples：ToLua 示例，供新手学习框架的使用方法。
* Misc：Lua 常用的工具类，比如 `LuaClient`、`LuaCoroutine`（框架提供的协程功能，比较好用）、`LuaResLoader`（用于加载 lua 文件）、`LuaLooper`（继承自 MonoBehavior，在 Update/LateUpdate/FixedUpdate 中执行 LuaEvent）。
* Reflection：反射相关代码。

ToLua 中的功能其实大部分都在示例中有演示，各位如果对某一模块不太了解的话可以到 Examples 中查看对应的代码。