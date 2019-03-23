# UnityShader入门

在图形渲染基础中我已经介绍过 Shader 了，可以简单归纳为：

* GPU 流水线上一些可高度编程的阶段。
* 有一些特定类型的着色器，比如顶点着色器、片元着色器等。
* 依靠着色器可以控制流水线中的渲染细节。

Shader 并没有特别之处，它只是一段规定好输入（颜色，贴图等）和输出（渲染器能够读懂的点和颜色的对应关系）的程序。而开发者要做的就是根据输入，进行计算变换，产生输出而已。

## 什么是UnityShader

---

UnityShader 并不是指真正的 Shader，它通常是指 `ShaderLab` 文件，也就是一种以 `.shader` 结尾的文件。

传统 Shader 与 UnityShader 的区别：

| <center>传统Shader</center> | <center>UnityShader</center> |
| :--- | :--- |
| 仅可以编写特定类型的 Shader，比如顶点着色器、片元着色器等 | 可以在同一文件中同时编写顶点着色器和片元着色器 |
| 无法设一些渲染设置，如是否开启混合、深度测试等。需要在另外代码中设置 | 通过一行特定指令就可以完成设置 |
| 要设置着色器输入和输出，并注意对应关系 | 只需在特定语句块中声明一些属性，就可以依靠材质修改这些属性。对于模型自带数据（顶点位置、uv坐标、法线等），也有直接访问的方法 |

传统的 Shader 编写比较复杂，并且需要注意渲染顺序等问题。UnityShader 则是 Unity 引擎的一种封装，能够帮助开发者轻松地管理着色器代码和渲染设置（比如开启/关闭混合、深度测试等）。

## ShaderLab

---

ShaderLab 是一种专门为 UnityShader 服务的语言，它的语法可能和传统的 shader 语言有些类似。传统的语言有 Cg（C for Graphic）、HLSL（High Level Shading Language）、GLSL（OnpenGL Shading Language），而由于微软和英伟达的合作，所以 Cg 和 HLSL 其实是同一种语言，ShaderLab 内部可以嵌套 Cg/HLSL 语言编写着色代码。

ShaderLab 为控制渲染过程提供了一层抽象，开发者只需要使用 ShaderLab 编写 UnityShader 即可完成工作，不需要去和其他文件以及设置打交道。

## UnityShader模板

---

在创建 Shader 文件时，Unity 提供了以下几种模板：

| <center>模板名</center> | <center>功能</center> |
| :--- | :--- |
| Standard Surface Shader | 包含标准光照模型的表面着色器模板 |
| Unlit Shader | 产生一个不包含光照（但包含雾效）的基本的顶点/片元着色器 |
| ImageEffect Shader | 实现各种屏幕后处理效果的基本模版 |
| Compute Shader | 特殊的Shader，利用GPU的并行性来进行一些与常规渲染流水线无关的计算 |

标准着色器模板的信息：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/UnityShader%E5%85%A5%E9%97%A801.png)

下面来解释一下各个属性的意思：

| <center>属性</center> | <center>功能</center> |
| :--- | :--- |
| Default Maps | 默认纹理。 |
| Surface shader | 可以查看 Unity 为该表面着色器生成的顶点/片元着色器 |
| Fixed function | 固定函数着色器（用于旧设备）。如果该着色器是固定函数着色器，那么也可以查看它的顶点/片元着色器 |
| Compiled code | 选择不同图像编程接口，编译成对应的 Shader 代码。直接点击按钮可以查看汇编指令 |
| Cast shadows | 是否捕获阴影 |
| Render queue | 渲染队列优先级 |
| LOD | 着色器的细节层次效果 |
| Ignore projector | 是否忽视投影（常用于半透明物体） |
| Disable batching | 是否不使用批处理 |
| Properties | 属性列表 |

## UnityShader结构

---

### 基础结构

UnityShader 的结构如下：

```
// 着色器名称
Shader "ShaderName"
{
	// 属性 
	Properties
    {
		Name ("display name", PrpertyType) = DefaultValue
	}
	// 子着色器
	SubShader
    {
		// 可选的，标签（告诉Unity如何、何时渲染对象）
		[Tags]
		
		// 可选的，状态设置（开关混合、深度测试，剔除模式，设置深度测试使用函数）
		[RenderSetup]
		
		// 完整渲染流程
		Pass
        {
			[Name]
			[Tags]
			[RanderSetup]
		}
	}

	// SubShader不能运行时会执行（同时渲染阴影）
	Fallback "VertexLit"
}
```

首先，我们要为 Shader 取一个名字，这个名字将会在材质的 Shader 下拉列表中出现。如果需要为 Shader 分组，那么你可以在名字加上组名并用 `/` 隔开。

### Properties

Shader 的 `Properties` 负责定义属性，格式为 `Name ("display name", PropertyType) = DefaultValue`。各部分含义如下：

* Name：在 Shader 中访问时用到的变量名。
* display name：在属性面板显示的名字。
* PropertyType：属性类型。
* DefaultValue：默认值。

下面介绍一下常用的几种属性：

```
Properties
{
	// Numbers and Sliders
	_Int ("Int", Int) = 2
	_Float ("Float", Float) = 1.5
	_Range("Range", Range(0.0, 5.0)) = 3.0
	// Colors and Vectors
	_Color ("Color", Color) = (1,1,1,1)
	_Vector ("Vector", Vector) = (2, 3, 6, 1)
	// Textures
	_2D ("2D", 2D) = "" {}
	_Cube ("Cube", Cube) = "white" {}
	_3D ("3D", 3D) = "black" {}
}
```

这里唯一需要注意的是，在 Unity 5 以前，纹理后面的花括号中可以写入一些对纹理的设置。但在版本更新后，如果你想要指定纹理属性，那么就需要自己在顶点着色器中计算纹理坐标。

将上述编写的 Shader 赋给一个默认材质，得到如下属性面板：

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/UnityShader%E5%85%A5%E9%97%A802.png)

### SubShader

子着色器（SubShader）是 UnityShader 中的重要部分。一个 Shader 文件可以包含多个 SubShader，但至少要有一个。Shader 语句块的定义通常如下：

```
SubShader
{
    // 可选的，标签（告诉Unity如何、何时渲染对象）
    [Tags]
    
    // 可选的，状态设置（开关混合、深度测试，剔除模式，设置深度测试使用函数）
    [RenderSetup]
    
    // 完整渲染流程
    Pass
    {
        [Name]
        [Tags]
        [RanderSetup]
    }
}
```

每个 `Pass` 语句块定义了一次完整的渲染流程，但如果 Pass 过多会导致渲染性能下降。Pass 块可以指定名称，比如 `Name "MyPass"`，我们可以在其它的 UnityShader 中调用该 Pass：

```
// Unity内部会把所有Pass转成大写，因此调用的时候要记得用大写形式
UsePass "TestShader/MYPASS"
```

标签和状态同样可以写在 Pass 中，但不同的是，SubShader 中会有一些特殊的标签，与 Pass 使用的标签不一样。

常用的状态设置如下：

| <center>状态名称</center> | <center>设置指令</center> | <center>解释</center> |
| :--- | :--- | :--- |
| Cull | Cull Back \| Front \| Off | 设置剔除模式：剔除背面/正面/关闭剔除 |
| ZTest | ZTest Less Greater \| LEqual \| GEqual \| Equal \| NotEqual \| Always | 设置深度测试时使用 |
| ZWrite | ZWrite On \| Off | 开启/关闭深度写入 |
| Blend | Blend SrcFactor DstFactor | 开启并设置混合模式 |

SubShader 的标签如下：

| <center>标签类型</center> | <center>说明</center> | <center>例子</center> |
| :--- | :--- | :--- |
| Queue | 控制渲染顺序，指定物体属于哪一个渲染队列 | Tags { "Queue" = "Transparent" } |
| RenderType | 对着色器分类，比如透明的着色器或者不透明的着色器 | Tags { "RenderType" = "Opaque" } |
| DisableBatching | 指明该 SubShader 是否使用批处理 | Tags { "DisableBatching" = "True" } |
| ForceNoShadowCasting | 控制使用该 SubShader 的物体是否会投射阴影 | Tags { "ForceNoShadowCasting" = "True" } |
| IgnoreProjector | 指明使用该 SubShader 的物体是否忽略 Projector 的影响 | Tags { "IgnoreProjector" = "True" } |
| CanUseSpriteAtlas | 当 SubShader 作用于 Sprite 时，需要将该标签设置为 False | Tags { "CanUseSpriteAtlas" = "False" } |
| PreviewType | 指明材质面板将会如何预览该材质。默认情况下材质会显示为球形，你可以指定为 Plane 或者 SkyBox 来改变预览 | Tags { "PreviewType" = "Plane" } |

Pass 的标签如下：

| <center>标签类型</center> | <center>说明</center> | <center>例子</center> |
| :--- | :--- | :--- |
| LightMode | 定义该 Pass 在 Unity 渲染流水线中的角色 | Tags { "LightMode" = "ForwardBase" } |
| RequireOptions | 用于指定某些条件，只有满足条件时才会渲染该 Pass | Tags { "RequireOptions" = "SoftVegetation" } |

虽然 SubShader 中可以设置许多标签和状态，但它最重要的还是指定各种着色器的代码，也就是说我们可以在这里编写真正意义上的 Shader 代码。

### Fallback

我们可以在最后面写一个 Fallback 指令，这个指令会告诉 Unity，如果所有 SubShader 都没有办法运行，那么就使用这个最基本的 Shader。

```
Fallback "TestShader"
// 或者
Fallback Off
```

你可以指定最基本的 Shader，当然你也可以直接把它关掉。

## UnityShader中的着色器形式

---

我们知道，渲染管线可以分为固定渲染管线和可编程渲染管线。UnityShader 依据这个分类将着色器划分为了三种形式。

### 表面着色器

表面着色器是 Unity 自创的一种着色器代码，它的代码量比较少，主要工作都由 Unity 代替，所以消耗也很大。它本质上和我们下面要讲的顶点/片元着色器是一样的，只不过 Unity 会在内部将其转换成对应的顶点/片元着色器。说得简单点，表面着色器是一层更高级的封装，用起来比较方便。

示例如下：

```
Shader "Custom/MyShader"
{
	SubShader
	{
		Tags { "RenderType"="Opaque" }

		CGPROGRAM
		#pragma surface surf Lambert
		struct Input
		{
			float4 color : COLOR;
		};

		void surf (Input IN, inout SurfaceOutput o)
		{
			o.Albedo = 1;
		}
		ENDCG
	}
	FallBack "Diffuse"
}
```

上述代码中，表面着色器被定义在 SubShader 的 `CGPROGRAM` 和 `ENDCG` 之间，也就是说开发者不需要去关心 Pass 如何渲染的问题，我们只需要指示 Unity 使用哪些纹理、哪些光照模型等即可，不需要关心具体怎么做。

CGPROGRAM 和 ENDCG 中间的代码是用 Cg/HLSL 编写的，也就是说我们可以用这种方式来嵌套 Cg/HLSL 语言。注意，这里的 Cg/HLSL 并不是原生的语言，语法上会有细微的不同。

### 顶点/片元着色器

Unity 中可以使用 Cg/HLSL 来编写顶点/片元着色器，它们比表面着色器复杂，但是灵活性更高。和表面着色器类似，顶点/片元着色器的代码也要定义在 CGPROGRAM 和 ENDCG 之间。但不同的是，顶点/片元着色器是写在 Pass 中的，而非 SubShader 中。虽然这样一来我们需要编写更多代码，但是这样一来就可以控制渲染的实现细节，更加灵活多变。

### 固定函数着色器

上面的两种着色器都是用了可编程管线，而对于旧设备而言，它们只能支持固定渲染管线，所以就要用到固定函数着色器。这些着色器往往只能实现非常简单的效果：

```
Shader "Tutorial/Basic"
{
    Properties
    {
        _Color ("Main Color", Color) = {1,0.5,0.5,1}
    }
    SubShader
    {
        Pass
        {
            Material
            {
                Diffuse [_Color]
            }
            Lighting On
        }
    }
}
```

可以看出，固定函数着色器只能在 Pass 中调整一些简单的配置，并且我们必须要完全使用 ShaderLab 的语法，不能用 Cg/HLSL。

### 如何选择着色器的形式

* 如果想使用各种光源，可以使用表面着色器，但需要注意它在移动设备的表现效果。
* 如果使用的光照较少，或者需要定义渲染效果，可以使用顶点/片元着色器。
* 如果要在旧设备上运行，那么可以使用固定函数着色器，但并不推荐。

## 总结

---

这一篇笔记介绍了许多新东西，不过不需要急着去背，我们可以在之后的学习中逐步熟悉这些代码。总的来说，UnityShader 并不是真正意义上的 Shader，它指的是一个 ShaderLab 文件，也就是以 .shader 结尾的文件。UnityShader 简化了开发者的工作，但是由于其高度封装性，我们可以编写的 Shader 类型和语法都被限制了。当然，绝大部分的时候我们只需要和 UnityShader 打交道，并不需要关心渲染底层的细节。

上面在介绍 Unity 的着色器形式时，我们一共介绍了三种形式的着色器。其实，在 Unity5.2 之后，Unity 中只存在顶点/片元着色器这一种，其它的两种着色器会被 Unity 转换成对应的顶点/片元着色器。我们在编写表面着色器时，会把 Cg/HLSL 代码写在 SubShader 中，但由于表面着色器会被自动转换，所以相关的 Cg/HLSL 代码其实都是写在 Pass 语句块中。如果对这方面有疑问的话，可以查看 Shader 文件的属性面板中的 `Compiled Code` 一栏，点击后可以阅读相关的汇编代码，同时也可以设置 Shader 需要编译的平台等。

![](http://cdn.fantasticmiao.cn/image/post/Unity/Advanced/UnityShader%E5%85%A5%E9%97%A803.png)