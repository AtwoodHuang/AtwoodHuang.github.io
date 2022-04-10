---
layout: article
title: openresty 性能优化的一些小心得
tags: openresty
aside:
  toc: true
---

# openresty 性能优化的一些小心得
openresty代码性能优化技巧其实网上已经有很多资料可以参考了，本文主要是收集作者实际碰到的一些小问题做点总结。
<!--more-->
## 1 . 不要做无效的优化
优化大部分请求都会经过的热代码路径才是最有效的，那种隔一段时间才能执行的定时器逻辑或者很难触发的请求逻辑能优化最好，不能优化也没太大问题。

## 2. 避免频繁的创建临时变量
这里所指的临时对象有两类，一类是 table 类，一类是字符串拼接。

### 2.1 避免频繁的字符串拼接
lua的字符串储存和c或者go这种我们常见的语言不一样。不管是全局变量还是局部变量，相同的字符串只会储存一份，因此每次创建字符串的时候都会对字符串算个哈希值看一下之前有没有相同字符串。

如果在你的火焰图中发现有大量的时间花在lj_str_new这个c函数上面说明你的代码中有就有大量临时字符串创建操作。一般的新建字符串不会造成性能的问题，出现这种问题一般是大量循环的字符串拼接造成的，当然也有例外的情况。之前作者就碰到过在热代码路径中有新建一个很长的字符串逻辑导致性能下降的情况。对于这种循环字符串拼接操作可以用table.concat来解决，比如如下代码：
```lua
local s = ""
for i = 1, 100000 do
    s = s .. "a"
end
可以优化成：

local t = {}
local index = 1
for i = 1, 100000 do
    t[index] = "a"
    index = index + 1
end
local s = table.concat(t, "")
```
但是其实正常业务代码中这种情况比较少。正常的打日志的时候使用string.fmt 或者 .. 来进行字符串拼接基本不会造成性能瓶颈（字符串很长很长的时候可能会，另外string.fmt 内部也是使用的..实现的 ）。

为了避免临时字符串的创建，openresty的一些接口其实允许我们传table进去，避免在lua代码层面做字符串拼接。在 ngx.say、ngx.print 、ngx.log、cosocket:send 等这些可能接受大量字符串的 API 中，它不仅接受 string 作为参数，也同时接受 table 作为参数：
```lua
local t = {}
local index = 1for i = 1, 100000 do
    t[index] = "a"  
    index = index + 1
end
ngx.say(t)
```

openresty会在c代码中帮我们完成字符串拼接的工作。

频繁的创建字符串也会导致额外的gc开销，但是在实际应用中其实可以发现字符串的gc基本没啥开销，gc的开销主要还是临时table造成的。

### 2.2 避免频繁的创建临时table
频繁的创建临时table会造成额外的初始化还有gc的开销（实际上发现主要是gc），如果在你的火焰图中发现大量的时间花在lj_tab_newxxxx，lj_gc_xxx,  rehashtab等c函数上面，那么多半就是table使用出了问题。

首先要注意的是table能复用就尽量复用不要在热代码路径下频繁出现如下代码：

```lua
function xxx()
    local a = {}
    a["a"] = b
    a["b"] = b
    a["c"] = b
end
```
这种代码会导致频繁的创建，gc table。比较好的做法是将这种需要频繁使用的table变量放到模块变量里面，用完之后使用table.clear清空table下次再用，或者直接使用openresty提供的pool池相关的库。

其次要善于使用table.new()。像下面这种代码
```lua
local t ={}
for i = 1, 1000 do 
  t[i] = i
end
```
看似没有问题，但是其实会导致大量table的rehash操作，造成性能浪费，但是如果用table.new()一次性建好一个容量为100的table就不会有这种问题
```lua
local t = table.new(100, 0)
for i = 1, 100 do  
   t[i] = i
end
```
如果我们没法确定这个table到底有多大，可以采取用空间换时间的做法，尽量初始化一个大一点的table。另外容易忽略的一点是用#获取数据长度其实是一个o(n)的操作。

## 3. 避免写出阻塞的代码
nginx是单线程的，一旦代码中有阻塞逻辑，整个进程就被卡住了没法处理请求。我们要尽量避免代码中有耗时太长的阻塞逻辑。比如把一些很耗cpu的计算逻辑用c实现。这里有个比较实用的小技巧，有时候有一些很耗时的嵌套循环逻辑可以用ngx.sleep(0)把代码逻辑切出去避免进程被卡住。当然要是代码逻辑因为磁盘io被卡住可能就没有什么太好的办法了（但是理论上这种情况应该比较少，因为操作系统的io操作都是有缓存的，一般来说读会卡住，写很少卡），实际生产环境都是尽量避免直接把日志打到磁盘上。

## 4 jit的问题
openresty使用的是luajit，要想让我们代码性能更好就要尽量让代码能被jit。但是其实在我们实际使用中很难保证自己的代码全部被jit（毕竟都是业务代码，什么逻辑都可能有），并且其实对于大部分情况来说代码没有被jit也不会造成性能瓶颈（luajit的作者是这么说的，没有必要一味的追求jit）。我目前只发现在使用了ffi的情况下代码不被jit会有很明显的性能差距。下面贴一个operesty官方博客的case<https://blog.openresty.com.cn/cn/lua-cpu-flame-graph/>

下面这段代码
```lua
local ffi = require "ffi"
local C = ffi.C
ffi.cdef[[
    double sqrt(double x);
]]
local _M = {}
local N = 1e7
 
local function heavy()
    local sum = 0
    for i = 1, N do
        -- sum = sum + i
        sum = sum + C.sqrt(i)
    end
    return sum
end
 
local function foo()
    local a = heavy()
    a = a + heavy()
    return a
end
local function bar()
    return (heavy())
end
function _M.main()
    ngx.say(foo())
    ngx.say(bar())
end
return _M
```
在执行这个官方case的时候如果把代码的jit关掉会明显的感觉到代码的卡顿。在作者的实际使用中也出现过由于使用了可变参数函数导致共享内存相关操作没有被jit引发的性能瓶颈。

那么怎么判断哪些代码没有被jit呢？我们先用stapxx火焰图的nojit选项大概看一下哪些热点函数没有被jit，然后通过jit.v或者jit.dump找一下trace断在哪里了，来进一步排查。

## 5. 其他的一些luajit的通用优化技巧
luajit的通用优化技巧可以参考作者的一封邮件

<http://wiki.luajit.org/Numerical-Computing-Performance-Guide>