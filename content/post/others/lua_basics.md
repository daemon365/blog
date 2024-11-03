---
title: "lua基础"
date: "2022-10-25T00:00:00+08:00"
tags: 
- lua
showToc: true
---


## 什么是lua

[Lua](https://www.lua.org/)是一个小巧的[脚本语言](https://baike.baidu.com/item/脚本语言)。它是[巴西](https://baike.baidu.com/item/巴西/5422)[里约热内卢天主教大学](https://baike.baidu.com/item/里约热内卢天主教大学/1050607)（Pontifical Catholic University of Rio de Janeiro）里的一个由Roberto Ierusalimschy、Waldemar Celes 和 Luiz Henrique de Figueiredo三人所组成的研究小组于1993年开发的。 其设计目的是为了通过灵活嵌入应用程序中从而为应用程序提供灵活的扩展和定制功能。Lua由标准C编写而成，几乎在所有操作系统和平台上都可以编译，运行。Lua并没有提供强大的库，这是由它的定位决定的。所以Lua不适合作为开发独立应用程序的语言。Lua 有一个同时进行的JIT项目，提供在特定平台上的即时编译功能。

## 安装

mac：

```bash
brew install lua
```

检验：

```go
lua -v
Lua 5.4.2  Copyright (C) 1994-2020 Lua.org, PUC-Rio
```

## 注释

### 单行注释

两个减号是单行注释:

```lua
--
```

### 多行注释

```lua
--[[
 多行注释
 多行注释
 --]]
```

## 标示符

Lua 标示符用于定义一个变量，函数获取其他用户定义的项。标示符以一个字母 A 到 Z 或 a 到 z 或下划线 **_** 开头后加上 0 个或多个字母，下划线，数字（0 到 9）。

最好不要使用下划线加大写字母的标示符，因为Lua的保留字也是这样的。

Lua 不允许使用特殊字符如 **@**, **$**, 和 **%** 来定义标示符。 Lua 是一个区分大小写的编程语言。

## 关键词

以下列出了 Lua 的保留关键词。保留关键字不能作为常量或变量或其他用户自定义标示符：

| and      | break | do    | else   |
| -------- | ----- | ----- | ------ |
| elseif   | end   | false | for    |
| function | if    | in    | local  |
| nil      | not   | or    | repeat |
| return   | then  | true  | until  |
| while    | goto  |       |        |

一般约定，以下划线开头连接一串大写字母的名字（比如 _VERSION）被保留用于 Lua 内部全局变量。

## 全局变量

在默认情况下，变量总是认为是全局的。

全局变量不需要声明，给一个变量赋值后即创建了这个全局变量，访问一个没有初始化的全局变量也不会出错，只不过得到的结果是：nil。

```lua
\> print(b)
nil
\> b=10
\> print(b)
10
\>
```

如果你想删除一个全局变量，只需要将变量赋值为nil。

```lua
b = nil
print(b)      --> nil
```

这样变量b就好像从没被使用过一样。换句话说, 当且仅当一个变量不等于nil时，这个变量即存在。

# Lua 数据类型

Lua 是动态类型语言，变量不要类型定义,只需要为变量赋值。 值可以存储在变量中，作为参数传递或结果返回。

Lua 中有 8 个基本类型分别为：nil、boolean、number、string、userdata、function、thread 和 table。

| 数据类型 | 描述                                                         |
| :------- | :----------------------------------------------------------- |
| nil      | 这个最简单，只有值nil属于该类，表示一个无效值（在条件表达式中相当于false）。 |
| boolean  | 包含两个值：false和true。                                    |
| number   | 表示双精度类型的实浮点数                                     |
| string   | 字符串由一对双引号或单引号来表示                             |
| function | 由 C 或 Lua 编写的函数                                       |
| userdata | 表示任意存储在变量中的C数据结构                              |
| thread   | 表示执行的独立线路，用于执行协同程序                         |
| table    | Lua 中的表（table）其实是一个"关联数组"（associative arrays），数组的索引可以是数字、字符串或表类型。在 Lua 里，table 的创建是通过"构造表达式"来完成，最简单构造表达式是{}，用来创建一个空表。 |

### table

Lua 中的表（table）其实是一个"关联数组"（associative arrays），数组的索引可以是数字或者是字符串。

```lua
a = {"zhaohaiyu", "baidu", "taobao", "qq"}
for key, val in pairs(a) do
    print("Key", key,"value",val)
end
```

```lua
lua 1.lua
Key	1	value	zhaohaiyu
Key	2	value	baidu
Key	3	value	taobao
Key	4	value	qq
```


```lua
a = {}
a["zhaohaiyu"] = "https://www.zhaohaiyu.com/"
a["百度"] = "https://www.baidu.com/"
a["淘宝"] = 11
a["淘宝"] = a["淘宝"] +  12
for k, v in pairs(a) do
	print(k .. " : " .. v)
end
```

```lua
lua 1.lua
zhaohaiyu : https://www.zhaohaiyu.com/
淘宝 : 23
百度 : https://www.baidu.com/
```

### function（函数）

在 Lua 中，函数是被看作是"第一类值（First-Class Value）"，函数可以存在变量里:

```lua
-- function_test.lua 脚本文件
function factorial1(n)
    if n == 0 then
        return 1
    else
        return n * factorial1(n - 1)
    end
end
print(factorial1(5))
factorial2 = factorial1
print(factorial2(5))
```

脚本执行结果为：

```
$ lua function_test.lua 
120
120
```

### thread（线程）

在 Lua 里，最主要的线程是协同程序（coroutine）。它跟线程（thread）差不多，拥有自己独立的栈、局部变量和指令指针，可以跟其他协同程序共享全局变量和其他大部分东西。

线程跟协程的区别：线程可以同时多个运行，而协程任意时刻只能运行一个，并且处于运行状态的协程只有被挂起（suspend）时才会暂停。

### userdata（自定义类型）

userdata 是一种用户自定义数据，用于表示一种由应用程序或 C/C++ 语言库所创建的类型，可以将任意 C/C++ 的任意数据类型的数据（通常是 struct 和 指针）存储到 Lua 变量中调用。

## Lua 变量

变量在使用前，需要在代码中进行声明，即创建该变量。

编译程序执行代码之前编译器需要知道如何给语句变量开辟存储区，用于存储变量的值。

Lua 变量有三种类型：全局变量、局部变量、表中的域。

Lua 中的变量全是全局变量，那怕是语句块或是函数里，除非用 local 显式声明为局部变量。

局部变量的作用域为从声明位置开始到所在语句块结束。

变量的默认值均为 nil。

```lua
a = 5               -- 全局变量
local b = 5         -- 局部变量
```

## for while if 

```lua
a = {1,2,3,4,5,6,7,8,9,10}

for i, v in ipairs(a) do
    print(i,v)
end
```

### 数值for循环

Lua 编程语言中数值 for 循环语法格式:

```lua
for var=exp1,exp2,exp3 do  
    <执行体>  
end  
```

var 从 exp1 变化到 exp2，每次变化以 exp3 为步长递增 var，并执行一次 **"执行体"**。exp3 是可选的，如果不指定，默认为1。

```lua
for i=10,1,-1 do
    print(i)
end
```

```lua
--while
a=10
while( a < 20 )
do
   print("a 的值为:", a)
   a = a+1
end
```

```lua
-- repeat
a = 10
repeat
   print("a的值为:", a)
   a = a + 1
until( a > 15 )
```

### break

Lua 编程语言 break 语句插入在循环体中，用于退出当前循环或语句，并开始脚本执行紧接着的语句。

如果你使用循环嵌套，break语句将停止最内层循环的执行，并开始执行的外层的循环语句。

### goto

Lua 语言中的 goto 语句允许将控制流程无条件地转到被标记的语句处。

```lua
local a = 1
::label:: print("--- goto label ---")

a = a+1
if a < 3 then
    goto label   -- a 小于 3 的时候跳转到标签 label
end
```

```lua
--if
if(0)
then
    print("0 为 true")
end
```

## Lua 函数

在Lua中，函数是对语句和表达式进行抽象的主要方法。既可以用来处理一些特殊的工作，也可以用来计算一些值。

Lua 提供了许多的内建函数，你可以很方便的在程序中调用它们，如print()函数可以将传入的参数打印在控制台上。

Lua 函数主要有两种用途：

- 1.完成指定的任务，这种情况下函数作为调用语句使用；
- 2.计算并返回值，这种情况下函数作为赋值语句的表达式使用。

```lua
--[[ 函数返回两个值的最大值 --]]
function max(num1, num2)

   if (num1 > num2) then
      result = num1;
   else
      result = num2;
   end

   return result;
end
-- 调用函数
print("两值比较最大值为 ",max(10,4))
print("两值比较最大值为 ",max(5,6))
```
