# 软件系统安全赛初赛2025部分wp


&lt;!--more--&gt;

## 前言

比赛时间有点阴间，题目的话全是二进制，能看的题就只有三道，Java听说是个1day，摆了

## 钓鱼邮件

记事本打开是一大串base64编码，解码后是个zip，下载下来爆破密码，最后密码为 20001111，得到个exe，拿到奇安信在线沙箱出

![image-20250105184120190](https://bu.dusays.com/2025/01/06/677bc66e1447e.png)

尝试后发现是222.218.218.218那个，md5一下就行

## CachedVisitor

main.lua

```
local function read_file(filename)
    local file = io.open(filename, &#34;r&#34;)
    if not file then
        print(&#34;Error: Could not open file &#34; .. filename)
        return nil
    end

    local content = file:read(&#34;*a&#34;)
    file:close()
    return content
end

local function execute_lua_code(script_content)
    local lua_code = script_content:match(&#34;##LUA_START##(.-)##LUA_END##&#34;)
    if lua_code then
        local chunk, err = load(lua_code)
        if chunk then
            local success, result = pcall(chunk)
            if not success then
                print(&#34;Error executing Lua code: &#34;, result)
            end
        else
            print(&#34;Error loading Lua code: &#34;, err)
        end
    else
        print(&#34;Error: No valid Lua code block found.&#34;)
    end
end

local function main()
    local filename = &#34;/scripts/visit.script&#34;
    local script_content = read_file(filename)
    if script_content then
        execute_lua_code(script_content)
    end
end

main()
```

最后是执行了一个文件，而`/scripts/visit.script`的作用是从外部 URL 获取数据并将其缓存到 Redis 中，一个很明显的ssrf打redis，刚开始走偏了，以为是打个cve

可以直接把`/scripts/visit.script`覆盖成执行`/readflag`并获取回显，用dict协议老是会报引号有问题，用gopher协议打

![image-20250106004111813](https://bu.dusays.com/2025/01/06/677bc66dbe0a0.png)

这里选择的是phpshell，默认生成的是shell.php，解码改一下文件名和长度

![image-20250106004210817](https://bu.dusays.com/2025/01/06/677bc66cdf879.png)

最后编码的话还是建议用bp直接编码，其他的总会出点问题

![image-20250106004340597](https://bu.dusays.com/2025/01/06/677bc66fcddbf.png)

接下来是二进制爷的wp

## donntyousee

打开文件逐步调试

![屏幕截图 2025-01-05 212105](https://bu.dusays.com/2025/01/06/677bc66dcd9c3.png)

这里面是call-ret型花指令，所以把相关内容nop掉，大致格式是：

```asm
push rax;
call loop;
loop:
xxxxx;
ret;
pop rax;
```

每个相关的函数都有这个，可以全部nop掉

![image-20250105222355494](https://bu.dusays.com/2025/01/06/677bc66d870ef.png)

![image-20250105222407010](https://bu.dusays.com/2025/01/06/677bc66d8b59d.png)

调用了这两个函数，分别都是call r8，只有动态调试

第一个是执行Sbox的初始化，用的数据是byte_5C5110，这个是key

![屏幕截图 2025-01-05 212123](https://bu.dusays.com/2025/01/06/677bc66d63c6d.png)

第二个函数是执行加密，很明显是一个魔改RC4，可以直接逆向。

继续往后调试

![屏幕截图 2025-01-05 212240](https://bu.dusays.com/2025/01/06/677bc66dd6ee3.png)

这里进行了比较，sub_4E4060是输出正确与否，所以看v7的终值，直接解密即可

key:921C2B1FBAFBA2FF07697D77188C

data:25 cd 54 af 51 1c 58 d3 a8 4b 4f 56 ec 83 5d d4 f6 47 4a 6f e0 73 b0 a5 a8 c3 17 81 5e 2b f4 f6 71 ea 2f ff a8 63 99 57

flag:dart{y0UD0ntL4cKg0oD3y34T0F1nDTh3B4aUtY}


---

> 作者: 6s6  
> URL: http://localhost:1313/posts/6c39531/  

