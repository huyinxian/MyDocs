# Lua与C#的高效数据交互

Lua 与 C# 的交互通常用的是虚拟堆栈，需要调用 Lua 的 C API 进行参数的入栈和出栈。在日常的开发中，部分功能模块可能会产生高频次的函数调用，比如位置的修改、网络协议的解析等。为了进一步优化游戏性能，我们需要修改底层的数据交互方式，实现 Lua 与 C# 的高效数据传输。

!> 注意，本文所展示的结构基于 Lua5.3，如果使用的是 Lua5.1 或者 LuaJit 需要根据源码结构进行相关的调整。

## 基本思路

---

Lua 中的 `TValue` 用于表示 Lua 的各类型变量，它的结构定义在源码的 `lobject.h` 中，如下所示：

```c
/*
** Tagged Values. This is the basic representation of values in Lua,
** an actual value plus a tag with its type.
*/

/*
** Union of all Lua values
*/
typedef union Value {
    GCObject *gc;    /* collectable objects */
    void *p;         /* light userdata */
    int b;           /* booleans */
    lua_CFunction f; /* light C functions */
    lua_Integer i;   /* integer numbers */
    lua_Number n;    /* float numbers */
} Value;

#define TValuefields    Value value_; int tt_

typedef struct lua_TValue {
    TValuefields;
} TValue;
```

`Value` 是一个联合体，它可以用于表示 Lua 中的各种数据类型。由于联合体内的各个成员都是共享的同一段内存，所以 `tt_` 就是用来表示当前的 `TValue` 属于哪种 Lua 类型。

相对应的，Lua 的 Table 结构也是定义在 `lobject.h` 中，如下所示：

```c
/*
** Common Header for all collectable objects (in macro form, to be
** included in other objects)
*/
#define CommonHeader    GCObject *next; lu_byte tt; lu_byte marked

typedef struct Table {
    CommonHeader;
    lu_byte flags;  /* 1<<p means tagmethod(p) is not present */
    lu_byte lsizenode;  /* log2 of size of 'node' array */
    unsigned int sizearray;  /* size of 'array' array */
    TValue *array;  /* array part */
    Node *node;
    Node *lastfree;  /* any free position is before this position */
    struct Table *metatable;
    GCObject *gclist;
} Table;
```

既然 Lua 中的变量类型和表结构都是确定的，那么我们就可以在 C# 端定义同样的结构体，让 C# 的结构与 Lua 的结构进行对齐。这样一来，我们只需要把 Lua 表的指针传给 C#，然后在 C# 中使用指针访问内存进行数据读写即可，避免了繁琐的堆栈交互。

## C#如何定义联合体

---

C# 实现联合体使用的是 `StructLayout` 属性，它提供了 `LayoutKind.Explicit` 和 `LayoutKind.Sequential` 这两种类型。先来看看 `TValue` 是如何实现的：

```csharp
[StructLayout(LayoutKind.Explicit, Size = 8)]
public struct LuaTValue32
{
    // GCObject*
    [FieldOffset(0)]
    public IntPtr gc;

    // bool
    [FieldOffset(0)]
    public int b;

    //lua_CFunction
    [FieldOffset(0)]
    public IntPtr f;

    // number
    [FieldOffset(0)]
    public float n;

    // integer value
    [FieldOffset(0)]
    public int i;

    // int tt_
    [FieldOffset(4)]
    public int tt_;
}
```

`Explicit` 可以设置结构体的总大小，而 `FieldOffset` 则是用于设置结构体内成员的字节偏移。由于 `TValue` 是由 `Value` 和 `tt_` 组成的（其中 `Value` 是联合体），因此我们只需要让 `Value` 内的成员共用前 4 个字节，而 `tt_` 使用后 4 个字节即可。

再来看看 `Table` 是如何实现的：

```csharp
[StructLayout(LayoutKind.Sequential)]
public struct LuaTableRawDef
{
    public IntPtr next;

    // lu_byte tt; lu_byte marked; lu_byte flags; lu_byte lsizenode;
    public uint bytes;

    // unsigned int sizearray
    public uint sizearray;

    // TValue* array
    public IntPtr array;

    // Node* node
    public IntPtr node;

    // Node* lastfree
    public IntPtr lastfree;

    // Table* metatable
    public IntPtr metatable;

    // GCObejct* gclist
    public IntPtr gclist;
}
```

标记为 `Sequential` 结构体会严格按照成员顺序组成结构体，因此 `LuaTableRawDef` 内的成员顺序一定要和 Lua 源码中的结构体一致。另外，Lua 的表结构会有四个 `byte` 类型的成员，这里为了方便直接用了一个 `uint` 占位，毕竟我们也不会用到这些成员。

## 如何修改数据

---

修改数据其实很简单。我们首先要在 Lua 中声明一个大的数组，然后把表的头指针传给 C# 端。C# 在拿到了指针后，就可以使用 `unsafe` 的方式读写 Lua 表的内存。

```csharp
public unsafe class LuaArrAccess
{
    LuaTableRawDef *tableRawPtr;

    public int GetInt(int index)
    {
        if (tableRawPtr != null && index > 0 && index <= tableRawPtr->sizearray)
        {
            // Lua传入的索引从1开始，这里在访问时要-1
            index = index - 1;
            // 访问Lua表的数组部分
            LuaTValue32 *tv = (LuaTValue32 *)tableRawPtr->array + index;
            // 判断类型
            if (tv->tt_ == LUA_TNUMINT)
            {
                return tv->i;
            }
            else
            {
                return (int)tv->n;
            }
        }
        else
        {
            // 读取错误，抛出异常
            return 0;
        }
    }

    public void SetInt(int index, int value)
    {
        if (tableRawPtr != null && index > 0 && index <= tableRawPtr->sizearray)
        {
            index = index - 1;
            LuaTValue32 *tv = (LuaTValue32 *)tableRawPtr->array + index;
            tv->i = value;
            tv->tt_ = LUA_TNUMINT;
        }
        else
        {
            // 写入错误，抛出异常
        }
    }
}
```

`tt_` 的类型在 Lua 的源码 `lobject.h` 和 `lua.h` 中也有定义，如下所示：

```c
/*
** basic types, from lua.h
*/
#define LUA_TNONE               (-1)
#define LUA_TNIL                0
#define LUA_TBOOLEAN            1
#define LUA_TLIGHTUSERDATA      2
#define LUA_TNUMBER             3
#define LUA_TSTRING             4
#define LUA_TTABLE              5
#define LUA_TFUNCTION           6
#define LUA_TUSERDATA           7
#define LUA_TTHREAD             8
#define LUA_NUMTAGS             9

/* Variant tags for numbers, from lobject.h */
#define LUA_TNUMFLT     (LUA_TNUMBER | (0 << 4))  /* float numbers */
#define LUA_TNUMINT     (LUA_TNUMBER | (1 << 4))  /* integer numbers */
```

## 注意事项

---

### 数组与散列表

Lua 中的表分成了数组和散列表两个部分，比如下面这样：

```lua
local index = 1
local _M = {}

--// Lua数组访问
_M[index] = 100

--// HashTable访问
_M.index = 100
```

本文提供的代码只实现了数组部分的访问，并没有实现散列表的访问。至于为什么不实现散列表的访问，其主要原因是数组的访问速度要高于散列表，并且散列表的访问实现起来比较麻烦。为了尽可能地提高性能，建议使用数组来存储数据。

使用数组访问时需要注意以下几点：

* Lua 的数组必须是连续的，如果内部出现空缺，那么很有可能会导致后续加入的数据被放到了散列表中，也就无法被 C# 读取到了。所以，我们在使用时最好一次性分配一片连续的数组内存，避免自己插入时出现错误。
* Lua 数组的扩容只能够在 Lua 端执行，不允许在 C# 中插入额外的数据来扩充长度。
* Lua 端不要强引用 C# 创建的 `LuaArrAccess`。强引用会导致 Lua array 进行 GC 时，C# 端无法进行响应。

?> 当然，如果你期望这个工具变得更加强大，那么也可以尝试着阅读 Lua 源码并做一些修改，我个人也会持续关注这一部分功能的完善。

### Lua数据类型

就目前而言，我只实现了 int 和 double 类型的读写，而 string 类型则没有实现。string 在 Lua 中也是用散列表实现的，变量只保存字符串的引用，因此实现起来比较繁琐。

其实从本质上来说，任何数据类型的读写都是可以实现的，区别只是实现的难易程度而已。不过既然我们只关注性能的关键点，那么实现 int 和 double 的读写就足够了，其他数据类型的读写可以用一些方式规避掉。

### Lua版本差异

首先要声明一点，Lua5.1、Lua5.3、LuaJit 的底层结构是有差别的，所以本文展示的代码并不能够兼容其他版本。

在 Lua5.1 中，number 对应的是 double 类型。由于游戏中经常会需要 64 位整数来存储玩家 ID，因此一般是用如下方式来间接实现：

```lua
local internalLow = 123456
local internalHigh = 123
local long = internalHigh * 4294967296 + internalLow
```

但问题是，double 所能表示的整数范围在 `-2^53~2^53` 之间。如果按照上面方式来进行计算，那么当整数超过 53 位时就会产生精度差异。因此，我们需要将结果转换成字符串或者用第三库来实现长整型。

Lua5.3 原生支持 int64，并且进一步区分了 32 位和 64 位版本。32 位版本使用的是 int 和 float，一般用于嵌入式开发或者小型机器。标准版本的 Lua 默认使用 64 位，使用的是 long 和 double。本文的代码只展示了 32 位版本的 `TValue`，因此还需要对照 Lua 源码实现 64 位版本。

LuaJit 的源码和 Lua 是不同的，如果项目使用的是 LuaJit，那么就需要对照着 LuaJit 的源码进行实现。

除此之外，在 `Plugins` 的 `x86` 和 `x86_64` 目录下都有一个 `tolua.dll` 文件，分别对应 Luajit 编译后的 32 位和 64 位的源码。同样的，在 `tolua.bundle -> Contents -> MacOS` 下也有一个 `tolua` 文件，不过这个是 Lua5.1 编译的源码（因为 IOS 没法使用 JIT 技术）。如果你想要测试 Lua5.3，那么你就得自行升级 Lua 版本并且关掉 JIT 模式。

## 应用场景

---

游戏中需要高频交互数据的地方有很多，这些地方都可以用本文提供的方式进行优化。举个例子，游戏中人物的头顶 UI 经常需要修改位置，那么这时候就可以申请一片数组内存用于数据交互。为了能够正确读取数据，Lua 与 C# 需要约定一种数据的读取方式，比如数组的第一个值是坐标的个数，然后按照 X、Y、Z 的顺序存储/读取数据。除此之外，一些常见的逻辑数据，比如角色信息、位置、属性，又或者是网络协议的解析等都可以用这种方式实现。