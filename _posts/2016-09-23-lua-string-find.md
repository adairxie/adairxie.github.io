---
layout:     post
title:      Lua string.find 中的 “坑”
subtitle:   ""
date:       2016-09-23
author:     ms2008
header-img: img/post-bg-digital-native.jpg
catalog:    true
tags:
    - Lua
    - Regex
---

我们的线上环境，ngx_lua api 都是以模块形式加载到 lua 级别的 vm 中，已达到最大性能。而且我们并没有使用传统的 “包” 的形式来加载(也就是 `require "xx.xx.xx"` )，而是直接以模块名为加载( `require "xx"` )，这就意味着我们需要不断的来动态设置 `package.path` 来配合 `require` 的机制。于是我们写了下面这个方法，来实现我们的需求：

```lua
function tools:loadluapath(root_path)
    if root_path == nil then
        return
    end

    local root_path = root_path .. "/?.lua;"
    if string.find(package.path,root_path) == nil then
        package.path = package.path .. root_path
    end
end
```

但近期我却发现 `package.path` 好像存在泄漏点，有将近 60K 的大小，而且还在一直持续增长。这并不符合我们的预期，其应该是在启动阶段过后，在一段时间内不断增长，之后应该是趋于稳定，到最后完全是一个常数级的大小。所以说，上面的 `loadluapath` 方法一定是出了问题，那么肯定就是 `string.find(package.path,root_path) == nil` 这条语句喽，继续测试：

```lua
Lua 5.1.4  Copyright (C) 1994-2008 Lua.org, PUC-Rio
>
>
> do
>>     print(package.path)
>>     print(string.rep("=",20))
>>     print(string.find(package.path,"/test/yyyy-mm-dd/?.lua"))
>> end
./?.lua;
/usr/local/lua/share/lua/5.1/?.lua;/usr/share/lua/5.1/?.lua;
/usr/share/lua/5.1/?/init.lua;/usr/lib64/lua/5.1/?.lua;
/usr/lib64/lua/5.1/?/init.lua;
/test/yyyy-mm-dd/?.lua
====================
nil
>
>
```

貌似不对，为什么 `string.find` 返回 `nil`，又是万能的 SO 上找到了解释：

> string.find(), by default, does not find strings in strings, it finds patterns in strings. str:find(pattern, init, plain) which allows you to pass in true as a last argument and search for plain strings.

原来 `string.find` 是当做 pattern 来查找的。修改一下，继续测试：

```lua
>
> do
>>     print(package.path)
>>     print(string.rep("=",20))
>>     print(string.find(package.path,"/test/yyyy-mm-dd/?.lua",1,true))
>> end
./?.lua;
/usr/local/lua/share/lua/5.1/?.lua;/usr/share/lua/5.1/?.lua;
/usr/share/lua/5.1/?/init.lua;/usr/lib64/lua/5.1/?.lua;
/usr/lib64/lua/5.1/?/init.lua;
/test/yyyy-mm-dd/?.lua
====================
162 183
```

Bingo ! 这下看到预期效果了。

如果你也不确定 `find` 的字符串会不会包含元字符，靠谱的方式，还是加个 `true` 参数比较好！
