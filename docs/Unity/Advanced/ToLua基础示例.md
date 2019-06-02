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

## 稍微复杂一点的示例

---

### 关于逗号和冒号运算符

在开始之前，我觉得有必要重申一下 `.` 和 `:` 的选择问题。`.` 运算符主要用于访问 table 表，如果使用点运算符进行赋值，就相当于对表中对应的字段进行修改：

```lua
t = {}
t.x = 1
t.y = 2

function t.PrintXY(x, y)
    print(x, y)
    print(t.x, t.y)
end

t.PrintXY(3, 4)     -- 3 4
                    -- 1 2
```

如果修改一下代码，将函数的点运算符改成冒号运算符，那么该函数将隐式传递一个 `self` 参数：

```lua
t = {}
t.x = 1
t.y = 2

-- 相当于t.PrintXY(self, x, y)
function t:PrintXY(x, y)
    print(x, y)
    print(t.x, t.y)
end

t.PrintXY(3, 4)     -- 4 nil
                    -- 1 2
t:PrintXY(3, 4)     -- 3 4
                    -- 1 2
```

上述例子分别使用了点运算符和冒号运算符对函数进行了调用。从结果可以看出，使用冒号运算符定义的函数会隐式传递一个 `self` 参数，因此使用点运算符调用该函数时，第一个参数会传给 self，因此就出现了 `4 nil` 的输出结果。当使用冒号运算符调用该函数时，会先把 `t` 这张表传给了 self，所以输出的结果是正常的 `3 4`。

不过要注意的是，由于 `:` 会把表传递给 self 参数，所以它只能够由具体的对象进行调用，而非具体的对象则无法使用冒号运算符。举个简单的例子，我们在使用 C# 接口时通常会这么写：

```lua
-- GameObject.Find是静态方法，要用.运算符
local global = GameObject.Find("Global")
-- global是具体的对象，因此调用AddComponent时可以用:运算符
global:AddComponent(typeof(AudioSource))
```

另外，有些时候我们会传入某些模块的某个方法，这时候也要注意一下：

```lua
require("Music")
-- 这里传入Music模块的方法，要用.运算符
coroutine.start(Music.PlaySound)
```

!> 在编写代码时，一定要注意某个表中的函数定义用的是 `.` 还是 `:`，另外调用该函数时也要注意运算符的使用

### C#部分

按照惯例，我们先写一个 LuaManager 用于管理 lua 脚本的启动：

_LuaManager.cs_

```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using LuaInterface;
using UnityEngine.SceneManagement;

public class LuaManager : MonoBehaviour
{
    private LuaState lua;
    private LuaResLoader loader;
    private LuaLooper looper;

    private static LuaManager _instance = null;
    public static LuaManager Instance
    {
        get
        {
            return _instance;
        }
    }

    private void Awake()
    {
        loader = new LuaResLoader();
        lua = new LuaState();
        lua.LuaSetTop(0);

        LuaBinder.Bind(lua);
        LuaCoroutine.Register(lua, this);

        _instance = this;
        DontDestroyOnLoad(gameObject);
    }

    private void Start()
    {
        lua.AddSearchPath(Application.dataPath + "/ResourceDynamic/Lua/JumpGame");
        looper = gameObject.AddComponent<LuaLooper>();
        looper.luaState = lua;

        lua.Start();
        SceneManager.LoadScene("JumpGamePlay");
    }

    public void DoFile(string fileName)
    {
        lua.DoFile(fileName);
    }

    public void CallFunc(string func, GameObject go)
    {
        LuaFunction luaFunc = lua.GetFunction(func);
        luaFunc.Call(go);
        luaFunc.Dispose();
        luaFunc = null;
    }

    private void OnApplicationQuit()
    {
        looper.Destroy();
        looper = null;

        lua.Dispose();
        lua = null;
        loader = null;
    }
}
```

LuaManager 主要的工作就是开启 lua 虚拟机，并且提供文件的加载和方法调用的功能。

接下来，我们需要在 lua 中注册 UI 的点击事件，因此需要一个脚本进行处理：

_UIEvent.cs_

```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;
using LuaInterface;

public class UIEvent : MonoBehaviour
{
	public static void AddButtonClick(GameObject go, LuaFunction func)
    {
        if (go == null) { return; }

        Button button = go.GetComponent<Button>();
        button.onClick.AddListener(() =>
        {
            func.Call(go);
        });
    }
}
```

`AddButtonClick` 会接受一个 lua 方法，然后把该方法注册到按钮的事件监听器中。

最后，我们还需要一个启动脚本来调用 lua：

_Login.cs_

```csharp
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using LuaInterface;

public class Login : MonoBehaviour
{
    private void Start()
    {
        LuaManager.Instance.DoFile("Login.lua");
        LuaManager.Instance.CallFunc("Login.Awake", gameObject);
    }
}
```

该脚本会加载 `Login.lua` 文件，然后调用里面的 `Awake` 方法。这里需要注意逗号运算符与冒号运算符的使用。

### lua部分

C# 部分主要用于提供框架底层功能以及启动 lua，游戏的主要逻辑我们全都写在 lua 中。

_Login.lua_

```lua
Login = {}
local this = Login
require("Music")
local ui

GameObject = UnityEngine.GameObject
Transform = UnityEngine.Transform
ParticleSystem = UnityEngine.ParticleSystem
Color = UnityEngine.Color
SceneManagement = UnityEngine.SceneManagement
Input = UnityEngine.Input
KeyCode = UnityEngine.KeyCode
Time = UnityEngine.Time
Camera = UnityEngine.Camera
AudioSource = UnityEngine.AudioSource
Resources = UnityEngine.Resources
www = UnityEngine.WWW

function this.Awake(obj)
    -- 注意，lua调用C#类的静态方法要用.运算符，调用非静态方法则是用:
    local global = GameObject.Find("Global")
    global:AddComponent(typeof(AudioSource))
    coroutine.start(Music.PlaySound)
    UIEvent.AddButtonClick(obj, OnLoginClick)
end

function OnLoginClick()
    print("Click")
end
```

上述代码一共干了两件事，第一件事是获取 `Global` 物体并为其添加音效组件，然后开启协程下载音乐并播放。第二件事是为按钮注册一个回调方法。

既然是要播放音乐，那么我们就需要下载一个音乐然后播放。下载功能可以使用 ToLua 框架封装的协程，它比 lua 自带的协程要好用的多。

_Music.lua_

```lua
Music = {}
local this = Music

function this:PlaySound()
    local audio = GameObject.Find("Global"):GetComponent("AudioSource")
    local url = www("https://etnly.oss-cn-shanghai.aliyuncs.com/%E5%B2%A1%E9%83%A8%E5%95%93%E4%B8%80%20-%20%E9%81%BA%E3%82%B5%E3%83%AC%E3%82%BF%E5%A0%B4%E6%89%80%EF%BC%8F%E6%96%9C%E5%85%89.ogg")
    coroutine.www(url)
    audio.clip = url:GetAudioClip()
    audio:Play()
end
```

如果你的代码没有出错的话，稍微等待一会你就能够听到音乐了（这里用的是尼尔机械纪元的背景音乐）。另外，当你点击按钮时也会打印出信息。

## 总结

---

在本章的末尾，我想说一说我个人的观点。我们之所以要在项目中使用 lua，并不是因为这个语言有多好用，效率有多高，仅仅是因为它能够很方便地进行热更新而已。在商业开发中，开发人员经常会在 lua 的性能上遇到瓶颈，会尝试用各种各样的方法去解决性能问题，所以 lua 这个东西其实是一把双刃剑，有利也有弊。我之后还会写几篇关于 C# 热更、XLua 的笔记，各位可以把这几个方案对比一下，选择一个最适合自己的解决方案。