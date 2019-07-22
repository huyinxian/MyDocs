# Lua基础

Lua 做为一门解释型语言，经常用于各类游戏开发之中。那么，在 Unity 中，我们为什么要使用 Lua，又该如何去使用呢？

## 为什么要使用解释型语言

---

一提到游戏开发所用的语言，我们想到的可能就是 C++、C# 这类高级语言。这些语言虽然功能强大，但其实很难满足游戏行业内不断变化的需求。试想一下，如果游戏需要有一个重大的更新，那么我们就必须重新把项目编译一遍，之后再打包上传至服务器。这一过程对于开发者来说没什么，但对于玩家而言，如果每次更新都需要重新下载安装包的话，绝对会让玩家给游戏打上差评。

就目前而言，游戏业内普遍采用的做法是把底层逻辑交给 C++、C# 等高级语言，然后把上层逻辑交给解释型语言。打个比方，游戏内的战斗逻辑一般是不会改变的，那么这一部分就可以用 C# 实现，提高游戏的效率，而对于游戏中常常变动的 UI、活动等部分，就可以交给更灵活的解释型语言。不过呢，不同的公司有不同的做法，有的公司喜欢全部用 C# 写，尽量做到高度可配置，然后采取上传 DLL 文件进行替换更新的方法来修复 BUG，而有的公司则全部都用 Lua 来编写代码，做到全面的热更新。孰优孰劣，都是看各自的需求罢了，并没有一种绝对的方案。

不过遗憾的是，哪怕安卓平台上可以下载脚本进行替换，但 IOS 平台是绝对不可以的。所以，开发者只能够上传完整的安装包或者用 Lua 来进行更新。

?> 有关于游戏热更新的讨论内容我会单独拿出来做笔记，本篇文章只是简要介绍一下 Lua 知识。

## 什么是Lua

---

简单的说一下，Lua 在开发之初就是为了嵌入应用程序中，为程序提供高可扩展性。Lua 由 C 编写，其本身极为小巧轻便，效率在各类脚本语言中也属于前列。除此之外，Lua 也支持面向对象和函数式编程，其提供的 table 表十分强大，可以实现各类功能。更重要的是，Lua 与其他语言可以很快的结合起来，这也是为什么要选择用 Lua 来作为热更新的脚本语言。

## Lua基础

---

### table

Lua 的数据类型不多，值类型有 `nil`、`number`、`string`、`boolean`，引用类型有 `function`、`userdata`、`thread`、`table`。其中，我们最需要掌握的就是 table 表。什么是 table 呢？table 是一个**关联数组**，索引可以是数字或者字符串，创建的方式为 `{}`。它的形式很像 C# 中的 Dictionary，但是二者之间还是有很大不同的。

table 的默认索引从 1 开始，如果你想要指定键值的话可以使用 `[]`。如果不好理解的话可以这么想，凡是被指定了键的元素可以被视作 Dictionary，而没有指定键的元素则可以看做是数组。可以看看下面的例子：

```lua
myArray = { 100, 200, [0] = "one", ['key'] = "value", "two" }
```

table 的遍历方式一共有三种，基本 for 循环、ipairs、pairs。`ipairs` 只能够遍历默认索引，并且遇到空值时就会停止；`pairs` 可以遍历 table 中的所有元素，但是其遍历顺序是先数组后 Dictionary，比如上面的这个表的打印结果为：

    1       100
    2       200
    3       two
    0       one
    key     value

如果你用的是 `ipairs`，那么结果如下：

    1       100
    2       200
    3       two

顺带一提，如果你想要获取 table 的长度，最好自己用循环来判断，因为 `#` 只能够获取数组的长度。还是以上面的表为例，因为表中只有三个默认索引的元素，所以输出的结果为 `3`。

这样就完了吗？当然不是，如果 table 只具备上面的功能，那它也仅仅只是个普通的存储结构罢了。除开默认索引和键，我们还可以在 table 中存储变量和函数：

```lua
function myPrintFunc(val)
    print(val)
end

myTable = { key = "value", myPrintFunc = myPrintFunc}

myTable.myPrintFunc(myTable.key)
```

访问 table 中的变量和函数使用的是 `.` 运算符，你可以使用它为 table 中变量赋值或者定义方法：

```lua
myTable = {}
myTable.key = "value"

function myTable.MyPrint(val)
    print(val)
end

myTable.MyPrint(myTable.key)
```

这种用法是不是有点眼熟呢？别着急，我们接着往下看。

### 元表

什么是元表呢？你可以把它理解为 table 的附加物或者背包，当 table 中找不到元素时，就会到它的元表中去寻找。元表中有许多特殊的元方法，可以为我们实现不同的功能。

处理元表的两个重要函数如下：

* setmetatable(table,metatable)：将 metatable 设置为 table 的元表。如果 metatable 中已经存在 _metatable 键值，那么 setmettable 会失败。
* getmetatable(table)：返回 table 的元表。

接下来开始介绍元表中的元方法。首先，最常用的是 `__index`。当你通过键来访问 table 时，假如这个键没有值，那么 Lua 就会寻找该 table 的元表中的 __index。如果 __index 是一个表，那么 Lua 就会在表中寻找相应的键；如果 __index 是一个函数，Lua 就会调用该函数，table 和键可以作为参数传递给函数。

```lua
newTable = { key1 = "value" }

setmetatable(newTable, { __index = function(myTable, key)
    print("__index")
end})
```

总而言之，Lua 在查找一个表时分为以下步骤：

* 在表中查找元素，如果找到就返回该元素，找不到则继续。
* 判断该表是否有元表，没有元表返回 nil，有的话就继续。
* 判断元表中有没有 __index，如果 __index 是一个方法，那么调用它；如果 __index 是一个表，那么重复上述步骤。

__index 用于表的访问，而 __newindex 则用于表的更新。如果你给表中的一个缺少的键赋值时，解释器会去查找 __newindex 方法，如果存在该方法则会调用它并且不会进行赋值操作：

```lua
newTable = {}
metaTable = {
    name = "123",
    __newindex = function(table, key, value)
        print(table)
        print("查找的键不存在："..key.."，要赋予的值为："..value)
    end
}
setmetatable(newTable, metaTable)
```

上述代码中，__newindex 被赋予了一个函数，该函数接收表、键、值三个参数。__newindex 可以用于监控表的赋值行为。

`__tostring` 可以修改表的打印方式：

```lua
mytable = setmetatable({ 10, 20, 30 }, {
  __tostring = function(mytable)
    sum = 0
    for k, v in pairs(mytable) do
        sum = sum + v
    end
    return "表所有元素的和为 " .. sum
  end
})

print(mytable)
```

?> 除了上述几种，还有可以改变表之间运算方式的元方法，如果各位感兴趣可以自行了解。

### OOP

元表的功能似乎很有趣，但它具体有什么作用呢？事实上，这正是 Lua 用来实现 OOP（面向对象编程）的主要手段。

假设有一个父类，它定义了公有变量 `age`，而继承自它的子类则定义了变量 `sex`。当我们调用子类的元素 sex 时，显然是能够得到对应的结果的，但如果我们调用的是 age 呢？学过高级编程语言的人应该都清楚，子类可以继承父类的公有变量，所以就算子类并没有定义过 age，它依旧可以返回父类所定义的变量。

这种继承的形式其实就和 Lua 查找 table 是一样的，我们可以把 table 的元表视作是它的父类，当 table 中没有某个键时，就会去搜索它的父类中是否有这个键。如果这一块有点搞不懂的话，建议仔细读一读我上面写的 Lua 查询表的步骤。

在开始介绍 Lua 如何实现 OOP 之前，我先讲一下 `.` 和 `:` 的区别。它们都可以用来定义方法，只不过使用 `:` 会默认传入一个参数 `self`，这个 self 其实就是调用该方法的 table 表。

下面是一个简单的类实现：

```lua
classA = { value = 0 }

function classA:new(o)
    local o = o or {}       -- 如果没有提供对象，就自行创建
    setmetatable(o, self)   -- 将o的元表设置为classA
    self.__index = self     -- 这一步很关键，我们可以通过元表来访问classA
    return o
end

myClass = classA:new()
print(myClass.value)
```

我们来看看上述代码都干了些什么。首先，我们定义了一个表 `classA`，然后为其添加了一个变量 `value`。随后我们为这个表定义了一个方法 `new`，然后创建了一个空表。为了能够访问 `classA`，我们需要把空表 `o` 的元表设置为 `classA`，并且把元表的 `__index` 键设置为 `classA`。这样一来，当我们访问 `o` 时，由于它是一个空表，自然就会到元表中去寻找元素，从而达到了访问 `classA` 的目的。

我之前也有说过，任何表都可作为其他表的元表，自身也可以成为自身的元表，上述代码就是借助了这一概念。另外，如果子类自己定义了与父类同名的变量或者方法，就会覆盖掉父类的变量和方法（因为只有当子类找不到索引时才会去查询元表）。

### 协程

协程与线程比较类似，拥有独立的堆栈，独立的局部变量，独立的指令指针等。它与线程的区别在于，多个协同程序需要彼此协作执行，也就是说任意时刻只能够有一个协程在执行，并且这个在运行的协程只有被明确要求挂起时才会被挂起。

协程的基本语法如下：

* coroutine.create()：创建协程，并返回这个协程，参数为一个函数。
* coroutine.resume()：启动协程。
* coroutine.yield()：挂起协程，并将参数返回。
* coroutine.status()：查看协程的状态。有 dead，suspend，running 三种。
* coroutine.wrap()：创建协程，并返回一个函数，参数为一个函数。
* coroutine.running()：返回正在执行的协程。协程也是一个线程，返回的是协程的线程号。

先来看一个例子：

```lua
co = coroutine.create(function (a)
    local r = coroutine.yield(a + 1)    -- yield()返回a+1给调用它的resume()函数，即2
    print("r=" ..r)                     -- r的值是第2次resume()传进来的，100
end)
status, r = coroutine.resume(co, 1)     -- resume()返回两个值，一个是自身的状态true，一个是yield的返回值2
coroutine.resume(co, 100)               -- resume()返回true
```

resume 用于启动协程，它会默认返回一个 boolean 值，用于表示协程是否启动成功。如果调用的协程中有 yield 方法，那么它还会返回 yield 的参数。

yield 用于挂起协程，在上述的例子中，协程在第一次启动时，会运行到 `local r = coroutine.yield(a + 1)` 时挂起。当我们再次启动协程时，才会执行后面的 `print("r=" ..r)`。

协程有三个状态，当协程创建时，其状态默认为 `suspend`；当协程被 resume 调用时，协程的状态为 `running`；当协程被挂起时，其状态改为 `suspend`；当协程中的方法执行到 `return` 语句时，就会转入 `dead` 状态，同时返回 return 的值。协程变为 `dead` 状态后无法再重新启动。

需要注意的是，创建协程的方法有 create 和 wrap，但是这两者是略有不同的。create 会返回一个协程，必须使用 resume 才能将其启动。而 wrap 则是返回一个函数，当调用函数时协程就会启动。

## Lua面向对象的探究

---

上面我简单地介绍了一下 Lua 是如何实现类的，不过那种简单的写法体会不到面向对象的好处。下面我将介绍一下比较优秀的 Lua 类实现。

下面的代码是由云风大大实现的：

```lua
local _class={}
function class(super)
    local class_type={}
    class_type.ctor     = false
    class_type.super    = super
    class_type.new      = 
        function(...)
            local obj={}
            do
                local create
                create = function(c,...)
                    if c.super then
                        create(c.super,...)
                    end
                    if c.ctor then
                        c.ctor(obj,...)
                    end
                end

                create(class_type,...)
            end
            -- 把该类的vtbl表设置为实例对象obj的元表
            setmetatable(obj,{ __index = _class[class_type] })
            return obj
        end
    local vtbl={}
    _class[class_type]=vtbl

    -- 当为某个类赋新值时，会把值存到该类对应的vtbl表中
    setmetatable(class_type,{__newindex=
        function(t,k,v)
            vtbl[k]=v
        end
    })
    
    -- 如果vtbl表中不存在某对象，那么就到父类中找（其实是在父类的vtbl中），找到了之后把结果拷贝到vtbl中，以供下次使用
    if super then
        setmetatable(vtbl,{__index=
            function(t,k)
                local ret=_class[super][k]
                vtbl[k]=ret
                return ret
            end
        })
    end

    return class_type
end
```

首先看 new 函数，每个 class 都会自动生成该函数，用于类的实例化。new 函数中定义了一个 create 方法，该方法会先调用父类的 ctor（构造方法），然后再调用子类的 ctor。ctor 需要传入 obj 作为参数，而 obj 就是 new 方法中最开始定义的一个空表，所有 ctor 中定义的变量（包括父类 ctor 中定义的）都会创建在 obj 中。这种做法的好处就是在 ctor 中定义的变量会在 obj 中创建（包括父类），而不在 ctor 中定义的变量则不会在 obj 中创建（不理解的话可以接着往下看）。new 函数的结尾定义了 obj 的元表，可能有人对于 ` _class[class_type]` 不太理解。不用急，下面就会解释。

我们接着看 vtbl 表。当对于 class_type 赋新值相当于直接在 vtbl 中赋值。我们在类中（class_type）添加任何类成员时，其实都是在 vtbl 中创建。由于实例对象的元表是 vtbl，因此实例对象可以访问到类中所有的方法和变量。

我们再来看代码最后一部分，这一部分其实是为了实现继承的逻辑。举个例子，类 B 继承自类 A，而 B 中的 vtbl 只能够访问到 B 中添加的方法和变量，无法访问 A 中的方法。此时我们就只能为 vtbl 设置一个元方法，当类 B 中不存在某变量或方法时，就去 B 的父类 A 中查找，如果父类也找不到就去父类的父类中找。`vtbl[k]=ret` 相当于把父类中查找到的结果拷贝到子类的 vtbl 中，避免下次查找相同值时又去搜索父类。

需要注意的是，那些不在 ctor 中声明的变量会保存到类的 vtbl 中，因为该表是唯一的，因此类的所有实例化对象是共用这张表的。这有点类似静态变量，但又不完全是，因为在多重继承时，父类的 vtbl 结果会在子类访问时被拷贝一次（在 C++ 中父类和子类共用一个静态成员）。

使用范例如下：

```lua
base_type=class()               -- 定义一个基类 base_type
function base_type:ctor(x)      -- 定义 base_type 的构造函数
    print("base_type ctor")
    self.x=x
end
function base_type:print_x()    -- 定义一个成员函数 base_type:print_x
    print(self.x)
end
function base_type:hello()      -- 定义另一个成员函数 base_type:hello
    print("hello base_type")
end

test=class(base_type)       -- 定义一个类 test 继承于 base_type
function test:ctor()        -- 定义 test 的构造函数
    print("test ctor")
end
function test:hello()       -- 重载 base_type:hello 为 test:hello
    --test.super:hello()
    print("hello test")
end

a=test.new(2)   -- 输出两行，base_type ctor 和 test ctor。这个对象被正确的构造了。
a:print_x()     -- 输出 1，这个是基类 base_type 中的成员函数。
a:hello()       -- 输出 hello test ，这个函数被重载了。
```

再来说一下 ctor 的问题。从上面的代码中我们已经知道了，ctor 中定义的变量会直接保存在实例 obj 中，而不在 ctor 中定义的变量和方法则保存在 vtbl 中，供类的所有实例对象使用。这种做法的好处就是节省了复制类变量和方法所消耗的时间，比如你经常能够看到这种代码：

```lua
--Create an class.
function class(classname, super)
    local superType = type(super)
    local cls

    if superType ~= "function" and superType ~= "table" then
        superType = nil
        super = nil
    end

    if superType == "function" or (super and super.__ctype == 1) then
        -- inherited from native C++ Object
        cls = {}

        if superType == "table" then
            -- copy fields from super
            for k,v in pairs(super) do cls[k] = v end
            cls.__create = super.__create
            cls.super    = super
        else
            cls.__create = super
        end

        cls.ctor    = function() end
        cls.__cname = classname
        cls.__ctype = 1

        function cls.new(...)
            local instance = cls.__create(...)
            -- copy fields from class to native object
            for k,v in pairs(cls) do instance[k] = v end
            instance.class = cls
            instance:ctor(...)
            return instance
        end

    else
        -- inherited from Lua Object
        if super then
            cls = clone(super)
            cls.super = super
        else
            cls = {ctor = function() end}
        end

        cls.__cname = classname
        cls.__ctype = 2 -- lua
        cls.__index = cls

        function cls.new(...)
            local instance = setmetatable({}, cls)
            instance.class = cls
            instance:ctor(...)
            return instance
        end
    end

    return cls
end
```

上述代码是 Cocos2dx 3.0 给出的 lua 类实现。代码中有这么一段：

```lua
if superType == "table" then
    -- copy fields from super
    for k,v in pairs(super) do cls[k] = v end
    cls.__create = super.__create
    cls.super    = super
else
    cls.__create = super
end
```

这种实现方式是让子类继承父类时，把父类所有的对象拷贝到子类中，从此之后子类和父类再无联系了，子类调用的父类变量和函数都是拷贝过来的。如果此时子类再重载父类的同名函数，那么之后子类将再也无法调用到对应的父类函数了（因为拷贝过来的被覆盖了）。

然后是它的实例化操作：

```lua
function cls.new(...)
    local instance = cls.__create(...)
    -- copy fields from class to native object
    for k,v in pairs(cls) do instance[k] = v end
    instance.class = cls
    instance:ctor(...)
    return instance
end
```

发现问题了吗？是的，每次实例化的时候都要把类中所有的变量和方法全部拷贝给实例对象。如果这个类有父类的话，那么还得把父类的也拷贝一遍。

在云风大大的代码中，**类中并不会创建任何成员**，因为添加类成员相当于给该类的 vtbl 表赋值（_newindex 被覆写了），而 vtbl 表对于所有的类对象来说是共用的。另外，ctor 中定义的变量是存到实例对象中的，也不会直接在类中创建，因此类中其实并没有创建任何的成员。比起上面那种每次实例化时都要把类成员拷贝一遍的方法，显然是云风大大的这种效率更高。

## Lua与C/C++交互

---

### Lua堆栈

Lua 和 C/C++ 交互用的其实是 Lua 堆栈。栈的特点是先进后出，堆栈的索引可以是正数也可以是负数，但正数索引 1 永远表示栈底，负数索引 -1 永远表示栈顶。

下面通过一个实例来解释上面这段话：

```cpp
#include <iostream>
 
extern "C" {
#include "lua.h"
#include "lualib.h"
#include "lauxlib.h"
}
 
using namespace std;
 
int main()
{
    //创建Lua环境
    lua_State* L=lua_open();
    //打开Lua标准库,常用的标准库有luaopen_base、luaopen_package、luaopen_table、luaopen_io、
    //luaopen_os、luaopen_string、luaopen_math、luaopen_debug
    luaL_openlibs(L);
    //压入一个数字20
    lua_pushnumber(L,20);
    //压入一个数字15
    lua_pushnumber(L,15);
    //压入一个字符串Lua
    lua_pushstring(L,"Lua");
    //压入一个字符串C
    lua_pushstring(L,"C");
    //获取栈元素个数
    int n=lua_gettop(L);
    //遍历栈中每个元素
    for(int i=1;i<=n;i++)
    {
        cout << lua_tostring(L ,i) << endl;
    }
    return 0;
}
```

看不懂方法没关系，你只需要理解这个过程即可。首先，我们创建了一个 Lua 虚拟机（堆栈），然后往这个堆栈中压入了四个元素并将它们全部打印。我们遍历时是从索引 1 开始的，也就是从栈底开始，最先入栈的会处于栈底。因此，如果我们按照递增的顺序输出元素，其实相当于自下而上进行输出。Lua 堆栈还提供了其它的接口，它们大部分都是与栈操作有关。

### Lua与C++交互

Lua 与 C++ 交互分为 Lua 调用 C++ 和 C++ 调用 Lua。首先来看 C++ 调用 Lua，这个其实直接用 Lua 环境就能够调用 Lua 脚本：

```cpp
#include <iostream>
 
using namespace std;
 
#include <iostream>
 
extern "C" {
#include "lua.h"
#include "lualib.h"
#include "lauxlib.h"
}
 
using namespace std;
 
int main()
{
    //创建Lua环境
    lua_State* L=luaL_newstate();
    //打开Lua标准库,常用的标准库有luaopen_base、luaopen_package、luaopen_table、luaopen_io、
    //luaopen_os、luaopen_string、luaopen_math、luaopen_debug
    luaL_openlibs(L);
 
    //下面的代码可以用luaL_dofile()来代替
    //加载Lua脚本
    luaL_loadfile(L,"script.lua");
    //运行Lua脚本
    lua_pcall(L,0,0,0);
 
    //将变量arg1压入栈顶
    lua_getglobal(L,"arg1");
    //将变量arg2压入栈顶
    lua_getglobal(L,"arg2");
 
    //读取arg1、arg2的值
    int arg1=lua_tonumber(L,-1);
    int arg2=lua_tonumber(L,-2);
 
    //输出Lua脚本中的两个变量
    cout <<"arg1="<<arg1<<endl;
    cout <<"arg2="<<arg2<<endl;
 
    //将函数printf压入栈顶
    lua_getglobal(L,"printf");
    //调用printf()方法
    lua_pcall(L,0,0,0);
 
    //将函数sum压入栈顶
    lua_getglobal(L,"sum");
    //传入参数
    lua_pushinteger(L,15);
    lua_pushinteger(L,25);
    //调用printf()方法
    lua_pcall(L,2,1,0);//这里有2个参数、1个返回值
    //输出求和结果
    cout <<"sum="<<lua_tonumber(L,-1)<<endl;
 
    //将表table压入栈顶
    lua_getglobal(L,"table");
    //获取表
    lua_gettable(L,-1);
    //输出表中第一个元素
    cout <<"table.a="<<lua_tonumber(L,-2)<<endl;
}
```

加载 Lua 脚本主要有 `luaL_loadfile()` 和 `luaL_dofile()` 两个方法，前者是仅加载脚本，而后者则会在加载的同时调用脚本文件。

Lua 脚本的代码如下：

```lua
--在Lua中定义两个变量
arg1=15
arg2=20
 
--在Lua中定义一个表
table=
{
    a=25,
    b=30
}
 
--在Lua中定义一个求和的方法
function sum(a,b)
  return a+b
end
 
--在Lua中定义一个输出的方法
function printf()
  print("This is a function declared in Lua")
end
```

可以看到，我们在 C++ 代码中调用 `lua_getglobal()` 方法把 Lua 脚本中的参数和方法压入了堆栈。当我们需要调用堆栈中的方法时，首先就是要把方法压入堆栈，然后通过 push 系列方法为栈中的方法传入参数。完成参数传入后，就可以使用 `lua_pcall()` 来执行方法。这里解释一下它的四个参数：第一个代表 Lua 环境 Lua_State，第二个是要传入的参数个数，第三个是返回值的个数，第四个一般默认为 0。执行该方法后，结果将被压入栈顶，我们可以使用索引 -1 访问结果。如果方法有多个返回值，那么 -1 代表最后一个返回值。

接下来再看看 Lua 调用 C++。首先我们要在 C++ 中定义一个方法，该方法必须以 `lua_State *` 为参数，返回值则为 int 类型，代表返回值个数。

```cpp
static int AverageAndSum(lua_State *L)
{
	//返回栈中元素的个数
	int n = lua_gettop(L);
	//存储各元素之和
	double sum = 0;
	for (int i = 1; i <= n; i++)
	{
	    //参数类型处理
		if (!lua_isnumber(L, i))
		{
		    //传入错误信息
			lua_pushstring(L, "Incorrect argument to 'average'");
			lua_error(L);
		}
		sum += lua_tonumber(L, i);
	}
	//传入平均值
	lua_pushnumber(L, sum / n);
	//传入和
	lua_pushnumber(L, sum);
 
	//返回值的个数，这里为2
	return 2;
}
```

然后要在 C++ 中注册该方法：

```cpp
lua_register(L, "AverageAndSum", AverageAndSum);
```

接下来我们就可以在 Lua 中调用 C++ 的方法：

```lua
average,sum=AverageAndSum(20,52,75,14)
print("Average=".average)
print("Sum=".sum)
```

Lua 与 C++ 的交互就先到此为止了，如果有其他需要的建议自行去了解，我这篇笔记的重点是介绍 Lua 和 C# 的交互。

## Lua与C#交互

---

### 常见Lua项目

介绍一下常见的 Lua 项目：

* LuaInterface：开源的 C# 与 Lua 的桥接库，能够快速实现 C# 与 Lua 的相互调用、事件传递。不过这个东西已经很久没更新了，适合初学者练习。
* nLua：LuaInterface 的分支，相当于对原有版本进行了升级。
* uLua：基于 LuaInterface 实现，集成了 Lua、Luajit、LuaInterface，效率比较高，在手游行业应用很广。
* toLua：uLua 升级版，效率更高，我之前实习过的游戏公司就是用的这个。
* sLua：从 uLua 扩展过来，效率也不错，不过这个经常被拿来和 toLua 对比，孰优孰劣我这里不做评价。
* xLua：腾讯推出的热补丁方案，我之后有时间会写一篇笔记来分享。

按道理来说，我们前面已经知道 Lua 如何与 C++ 交互了，那么我们应该也可以通过编写 dll 来访问 Lua 脚本。不过呢，既然已经有了成熟的开源项目，那么我们还是直接用比较好。

### LuaInterface

Lua 解释器能够被轻松嵌入宿主库，而 LuaInterface 则是 CLR 访问 Lua 解释器的主要接口。一个 LuaInterface 类对象相当于一个 Lua 解释器，解释器可以存在多个，并且完全相互独立。

使用 LuaInterface 需要下载 LuaInterface.dll、Luanet.dll，然后在脚本中引用。

C# 调用 Lua 如下：

```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using LuaInterface;

namespace TestCSharpAndLuaInterface
{
    static void Main(string[] args)
    {
        // 新建一个Lua解释器，每一个Lua实例都相互独立
        Lua lua = new Lua();

        // Lua的索引操作[]可以创建、访问、修改global域，括号里面是变量名
        // 创建global域num和str
        lua["num"] = 2;
        lua["str"] = "a string";
        
        // 创建空table
        lua.NewTable("tab");

        // 执行lua脚本，这两个方法都会返回object[]记录脚本的执行结果
        lua.DoString("num = 100; print(\"i am a lua string\")");
        lua.DoFile("C:\\luatest\\testLuaInterface.lua");
        object[] retVals = lua.DoString("return num,str");

        // 访问global域num和str
        double num = (double)lua["num"];
        string str = (string)lua["str"];

        Console.WriteLine("num = {0}", num);
        Console.WriteLine("str = {0}", str);
        Console.WriteLine("width = {0}", lua["width"]);
        Console.WriteLine("height = {0}", lua["height"]);
    }
}
```

LuaInterface 创建堆栈用的是 `new Lua()`，并且这个堆栈可以用 `[]` 直接访问，很是方便。LuaInterface 有两个重要的方法：`Lua.DoString()` 和 `Lua.DoFile()`。DoString 可以直接执行 Lua 代码，你只需要把代码作为字符串传入即可；DoFile 则是执行指定路径的 Lua 脚本。

Lua 调用 C# 也很简单，我们需要先注册方法，然后再进行调用：

```csharp
namespace TestCSharpAndLuaInterface
{
    class TestClass
    {
        private int value = 0;

        public void TestPrint(int num)
        {
            Console.WriteLine("TestClass.TestPrint Called! value = {0}", value = num);
        }

        public static void TestStaticPrint()
        {
            Console.WriteLine("TestClass.TestStaticPrint Called!");
        }
    }

    class Program
    {
        static void Main(string[] args)
        {
            Lua lua = new Lua();
            
            TestClass obj = new TestClass();
            // 注册类对象方法到Lua，供Lua调用
            lua.RegisterFunction("LuaTestPrint", obj, obj.GetType().GetMethod("TestPrint"));
            // 注册类静态方法到Lua，供Lua调用
            lua.RegisterFunction("LuaStaticPrint", obj, typeof(TestClass).GetMethod("TestStaticPrint"));

            // 在Lua中调用C#的函数
            lua.DoString("LuaTestPrint(10)");
            lua.DoString("LuaStaticPrint()");
        }
    }
}
```

上述代码中我是直接用 DoString 来测试的。如果你需要在 Lua 脚本中调用 C#，可以看下面的例子：

```lua
-- 引用LuaNet.dll
require "luanet"
-- 引用C#的System
luanet.load_assembly("System")
-- 引用System的Int32类
Int32 = luanet.import_type("System.Int32")
-- 将字符串转换成int
num = Int32.Parse("123")
print(num)
```

在 Lua 中访问 C# 的属性和方法其实就是访问对应的索引，比如访问属性的话就是 `obj.name` 或者 `obj["name"]`，访问方法的话则是 `obj:method()`。

LuaInterface 的基本用法就到此为止了，用上面的方法便能够在 Unity 中处理 C# 与 Lua 的交互。当然，如果你要把项目移植到各个平台的话（特别是 IOS），那么你是不能够使用 LuaInterface 的，因为某些平台并不支持即时编译，也就是说你无法使用反射动态加载对象。这种时候你就需要把 C# 代码转换为静态文件以供 Lua 调用。各位如果想把 Lua 运用到自己的项目中，可以去看一看 uLua、toLua 等开源项目，这些东西用起来效率更高，并且可以移植到各个平台。
