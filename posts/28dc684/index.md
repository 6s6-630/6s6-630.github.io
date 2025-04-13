# CodeQL入门


&lt;!--more--&gt;

## 前言

CodeQL 不多说，帮助我们进行代码审计的一个挺好的工具

## CodeQL 安装

CodeQL本身包含两部分解析引擎&#43;`SDK`。

解析引擎用来解析我们编写的规则，虽然不开源，但是我们可以直接在官网下载二进制文件直接使用。

`SDK`完全开源，里面包含大部分现成的漏洞规则，我们也可以利用其编写自定义规则。

### 引擎安装

去地址：https://github.com/github/codeql-cli-binaries/releases 下载已经编译好的codeql执行程序，解压之后把codeql文件夹放入～/CodeQL，我这里是 windows ，下的是 codeql-win64.zip

为了方便测试我们需要把ql可执行程序加入到环境变量当中：

```
setx PATH &#34;E:\CodeQL\codeql;%PATH%&#34;
```

然后重开个cmd输入 codeql，出现如下图就说明引擎设置完成

![image-20250409131900882](https://bu.dusays.com/2025/04/13/67fb53f4cf635.png)

### SDK安装

在这边下载规则库文件https://github.com/github/codeql，用来在后续库中进行查询

将解压后的文件放入~/CodeQL中，之后输入codeql pack ls来查看当前SDK中支持的规则集。

![image-20250409154407515](https://bu.dusays.com/2025/04/13/67fb541c9d8ba.png)

### VSCode开发插件安装

在 vscode 中安装 CodeQL 插件

![image-20250409154926481](https://bu.dusays.com/2025/04/13/67fb541cae0dd.png)

然后在该插件的设置中设置 codeql 的可执行文件路径（路径中最好不要有中文）

![image-20250409171548823](https://bu.dusays.com/2025/04/13/67fb53f4c8f02.png)

然后就设置好了，接下来写个demo测试一下

## demo

由于`CodeQL`的处理对象并不是源码本身，而是中间生成的AST结构数据库，所以我们先需要把我们的项目源码转换成`CodeQL`能够识别的`CodeDatabase`。

切换到源代码所在的目录然后再执行创建数据库的命令

```
codeql database create &lt;数据库名&gt; --language=&lt;语言标识符&gt; --source-root=&lt;源码路径&gt;
```

如果源代码是一个Maven项目，可能需要使用Maven命令来构建项目，并在创建数据库时指定该命令

```
--command=&#34;mvn clean install&#34;
```

不同的 language 所对应的语言标识符

| Language              | Identity   |
| --------------------- | ---------- |
| C/C&#43;&#43;                 | cpp        |
| C#                    | csharp     |
| Go                    | go         |
| Java                  | java       |
| javascript/Typescript | javascript |
| Python                | python     |

这里我主要是用 codeql 来审计 java 代码，所以用 [micro_service_seclab](https://github.com/l4yn3/micro_service_seclab/) 这个靶场来做测试

```
codeql database create micro_service_seclab_database --language=&#34;java&#34; --command=&#34;mvn clean install -Dmaven.test.skip=true&#34; --source-root=E:\CodeQL\test\micro_service_seclab
```

&gt; 稍微解释下，在当前目录下创建个 `micro_service_seclab_database` 当作存放数据库的目录，指定语言为java，然后写出构建和清楚命令，最后指定源代码跟目录

然后在这里把生成的 `micro_service_seclab_database` 添加进去

![image-20250409180422875](https://bu.dusays.com/2025/04/13/67fb53f50d7b5.png)

然后出现这个就表示数据库加载成功了

![image-20250409180504379](https://bu.dusays.com/2025/04/13/67fb53f488cde.png)

接着再添加CodeQL SDK：“文件”-“将文件夹添加到工作区”

然后在 sdk/java/ql 目录下创建个 demo.ql 查询文件，然后 `Run Query on Selected Database`，就会打印 `heloooo test`

![image-20250409182017378](https://bu.dusays.com/2025/04/13/67fb541cb7d99.png)

## 基本语法

以上面的靶场为例

![image-20250409182314089](https://bu.dusays.com/2025/04/13/67fb53f50a195.png)

可以看到 CodeQL 引擎的作用就是帮我们把源码转换为 CodeQL 能识别的数据库，所以我们能做的就是编写 QL 规则，再通过其引擎来运行我们的规则，这样就可以达到一个自动审计的功能

### QL语法

```
import java
 
from int i
where i = 1
select i
```

第一行表示我们要引入CodeQL的类库，这里我们以 java 为例

&gt; `from int i`，表示我们定义一个变量i，它的类型是int，表示我们获取所有的int类型的数据
&gt;
&gt; `where i = 1`, 表示当i等于1的时候，符合条件
&gt;
&gt; `select i`，表示输出i

如果QL文件在 `sdk\java\ql\` 目录中时就不需要 `import java` 了，因为这是 CodeQL 标准库目录，其会自动隐式加载该目录下的依赖，但如果写在其他目录下就需要手动 `import java` ，包括其子目录

这里我是写在 `sdk\java\ql\examples` 下的，上面的代码很好理解，所有int类型的数据中筛选出为1的情况，输出就是 1

![image-20250409185510005](https://bu.dusays.com/2025/04/13/67fb541c8e622.png)

QL查询的语法结构为：

```
from [datatype] var
where condition(var = something)
select var
```

从上面的例子中我们发现完整的QL语法无非就分三部分，先是限定一个查询的区域，然后写出过滤规则，最后输出

### 类库

上面我们说了CodeQL引擎会将代码转换为数据库，这个数据库其实就是可识别的AST数据库

在AST里面Method代表的就是类当中的方法；比如说我们想过的所有的方法调用，MethodAccess获取的就是所有的方法调用。

我们经常会用到的ql类库大体如下：

| 名称         | 解释                                                         |
| ------------ | ------------------------------------------------------------ |
| Method       | 方法类，Method method表示获取当前项目中所有的方法            |
| MethodAccess | 方法调用类，MethodAccess call表示获取当前项目当中的所有方法调用 |
| Parameter    | 参数类，Parameter表示获取当前项目当中所有的参数              |

结合ql的语法，我们尝试获取micro-service-seclab项目当中定义的所有方法：

```
import java
 
from Method kkk
select kkk
```

![image-20250410215223683](https://bu.dusays.com/2025/04/13/67fb53f4c222a.png)

然后添加下过滤条件，筛选出名字为 getStudent 的方法名称

```
import java
 
from Method k
where k.hasName(&#34;getStudent&#34;)
select k.getName(), k.getDeclaringType()
```

![image-20250410215935790](https://bu.dusays.com/2025/04/13/67fb53f4c07ca.png)

```
k.hashName() 判断名字是否匹配
k.getName() 获取的是当前方法的名称
k.getDeclaringType() 获取的是当前方法所属class的名称
```

### 谓词

如果限制条件比较多，where 语句就会很冗长。CodeQL提供一种机制可以帮助我们把很长的查询语句封装成函数，而这个函数，就叫谓词。

比如上面的案例，我们可以写成如下，获得的结果跟上面是一样的：

```
import java
 
predicate isStudent(Method k) {
    k.getName()=&#34;getStudent&#34;
}
 
from Method k
where isStudent(k)
select k.getName(), k.getDeclaringType()
```

&gt; predicate 表示当前方法没有返回值。

### 设置Source和Sink

&gt; 什么是source和sink
&gt;
&gt; 在代码自动化安全审计的理论当中，有一个最核心的三元组概念，就是(source，sink和sanitizer)
&gt;
&gt; source是指漏洞污染链条的输入点。比如获取http请求的参数部分，就是非常明显的Source
&gt;
&gt; sink是指漏洞污染链条的执行点，比如SQL注入漏洞，最终执行SQL语句的函数就是sink(这个函数可能叫query或者exeSql，或者其它)
&gt;
&gt; sanitizer又叫净化函数，是指在整个的漏洞链条当中，如果存在一个方法阻断了整个传递链，那么这个方法就叫sanitizer，也就是waf

只有当source和sink同时存在，并且从source到sink的链路是通的，才表示当前漏洞是存在的。

![image-20250410221945041](https://bu.dusays.com/2025/04/13/67fb541c72fb3.png)

**设置Source**

在CodeQL中我们通过以下方法来设置Source

```
override predicate isSource(DataFlow::Node src) {}
```

在这个靶场中，我们使用的是`Spring Boot`框架，那么source就是http参数入口的代码参数，比如在下面的代码中，source就是username：

```
@RequestMapping(value = &#34;/one&#34;)
public List&lt;Student&gt; one(@RequestParam(value = &#34;username&#34;) String username) {
    return indexLogic.getStudent(username);
}
```

本例中我们设置Source的代码为：

```
override predicate isSource(DataFlow::Node src) { src instanceof RemoteFlowSource }
```

&gt; `RemoteFlowSource` 是 CodeQL 标准库中预定义的 **“远程数据源”** 类，比如HTTP 请求参数，用户输入以及其他外部输入等都是

这是`SDK`自带的规则，里面包含了大多常用的Source入口。我们使用的SpringBoot也包含在其中，可以直接使用。

注: instanceof 语法是CodeQL提供的语法，后面在CodeQL进阶部分会提到，这里就是检查获得的 src 是否为 RemoteFlowSource

**设置Sink**

在CodeQL中我们通过以下方法来设置Sink

```
override predicate isSink(DataFlow::Node sink) {}
```

在实际中，我们最后都是触发到某个恶意方法，如 getter，setter，所以 sink 应该是个方法，假设我们这里的sink 点是个`query`方法(Method)的调用(MethodAccess)，所以我们设置Sink为：

```
override predicate isSink(DataFlow::Node sink) {
exists(Method method, MethodAccess call |
      method.hasName(&#34;query&#34;)
      and
      call.getMethod() = method and
      sink.asExpr() = call.getArgument(0)
	  )
}
```

&gt; 这里我们使用了exists子查询，这个是CodeQL谓词语法里非常常见的语法结构，它根据内部的子查询返回true or false，来决定筛选出哪些数据。
&gt;
&gt; `sink.asExpr() = call.getArgument(0)`：将 sink 节点转换为表达式，并检查它是否等于 `call` 的第一个参数
&gt;
&gt; 故上面sink语句的作用是查找一个query()方法的调用点，并把它的第一个参数设置为sink

在靶场系统(`micro-service-seclab`)中，sink为

```
jdbcTemplate.query(sql, ROW_MAPPER);
```

当刚才设置的source变量流入这个方法时，说明注入点和触发点是通的，就能产生注入漏洞

### Flow数据流

设置好Source和Sink，就相当于搞定了首尾，接下来就是疏通中间的利用链。一个受污染的变量，能够毫无阻拦的流转到危险函数，就表示存在漏洞

这个连通工作就是CodeQL引擎本身来完成的。我们通过使用`config.hasFlowPath(source, sink)`方法来判断是否连通。

比如如下代码：

```
from VulConfig config, DataFlow::PathNode source, DataFlow::PathNode sink
where config.hasFlowPath(source, sink)
select source.getNode(), source, sink, &#34;source&#34;
```

&gt; 我们传递给`config.hasFlowPath(source, sink)`我们定义好的source和sink，系统就会自动帮我们判断是否存在漏洞了。
&gt;
&gt; `source.getNode()`：获取源节点的底层语法树节点（AST Node），显示漏洞源头在代码中的具体位置

## 代码测试

### 初步检测

综上，可以写个 ql 查询代码来检测 sql 注入漏洞

```
/**
 * @id java/examples/vuldemo
 * @name Sql-Injection
 * @description Sql-Injection
 * @kind path-problem
 * @problem.severity warning
 */

import java
import semmle.code.java.dataflow.FlowSources
import semmle.code.java.security.QueryInjection
import DataFlow::PathGraph

class VulConfig extends TaintTracking::Configuration {
     VulConfig() { this = &#34;SqlInjectionConfig&#34;}
    
    override predicate isSource(DataFlow::Node src) {
        src instanceof RemoteFlowSource
    }
    
    override predicate isSink(DataFlow::Node sink) {
        exists(Method method, MethodAccess call |
            method.hasName(&#34;query&#34;)
            and
            call.getMethod() = method and
            sink.asExpr() = call.getArgument(0)
        )
    }

}

from VulConfig config, DataFlow::PathNode source, DataFlow::PathNode sink
where config.hasFlowPath(source, sink)
select source.getNode(), source, sink, &#34;source&#34;
```

CodeQL 在定义类上的语法和 Java 类似，其中 extends 的父类 `TaintTracking::Configuration` 是官方提供用来做数据流分析的通用类，提供很多数据流分析相关的方法，比如isSource(定义source)，isSink(定义sink)

`src instanceof RemoteFlowSource` 表示src 必须是 RemoteFlowSource 类型。在RemoteFlowSource里，官方提供很非常全的source定义，我们本次用到的Springboot的Source就已经涵盖了。

- 注：上面的注释和其它语言是不一样的，不能够删除，它是程序的一部分，因为在我们生成测试报告的时候，上面注释当中的name，description等信息会写入到审计报告中。

这里的`isSource` 和`isSink` 根据自己需要进行重写，而判断中间是否疏通可以使用CodeQL提供的`config.hasFlowPath(source, sink)`来帮我们处理

![image-20250411180745584](https://bu.dusays.com/2025/04/13/67fb53f5aef1e.png)

可以看到真的很方便，直接把source处和整个从source到sink的链子都显示出来了

这里爆警告说是 `DataFlow::PathGraph` 在新版本中被弃用了，所以这段代码只能在低版本的规则库里跑

这里开头有 `@kind path-problem` ，说明结果至少是4列，写了这个结果就会输出完整的污点传播路径，除此之外对输出还有其他要求，比如每列要输出的类型也有要求，但是我这 `source.getNode(),source, sink,&#34;source&#34;` 就能正常输出，`source, sink,source.getNode(),&#34;source&#34;` 却是啥也没有

### 误报分析

我们可以看到有一处的 sql 注入，其输入的参数类型为 `List&lt;Long&gt;` ，不可能存在注入

![image-20250411183001837](https://bu.dusays.com/2025/04/13/67fb541d328c4.png)

这里说明我们给的限制并未严格要求参数类型，就会导致以上的误报产生，我们可以用 isSanitizer 来避免这种情况

![image-20250411183843751](https://bu.dusays.com/2025/04/13/67fb541c3dbf1.png)

isSanitizer是CodeQL的类TaintTracking::Configuration提供的净化方法。它的函数原型是：

```
override predicate isSanitizer(DataFlow::Node node) {}
```

在CodeQL自带的默认规则里，对当前节点是否为基础类型做了判断

```
override predicate isSanitizer(DataFlow::Node node) {
node.getType() instanceof PrimitiveType or
node.getType() instanceof BoxedType or
node.getType() instanceof NumberType
}
```

表示如果当前节点是上面提到的基础类型，那么此污染链将被净化阻断，漏洞将不存在，可以看到这里默认规则只是一些基础类型，没有类似 `List&lt;long&gt;` 等的复合类型

我们将 TaintTracking::Configuration 中的 isSanitizer 重写下就好了

```
override predicate isSanitizer(DataFlow::Node node) {
    node.getType() instanceof PrimitiveType or
    node.getType() instanceof BoxedType or
    node.getType() instanceof NumberType or
    exists(ParameterizedType pt| node.getType() = pt and pt.getTypeArgument(0) instanceof NumberType )  // 这里的 ParameterizedType 代表所有泛型，判断泛型当中的传参是否为 Number 型
  }
```

这样在检测到 `List&lt;long&gt;` 时就会将其净化掉，这样链子就不通了，也就不会出现误报的情况

### 漏报补充

我们发现如下 sql 注入没有被捕捉到（参考文章是这样写的，但我这是捕捉到了的）

```
public List&lt;Student&gt; getStudentWithOptional(Optional&lt;String&gt; username) {
        String sqlWithOptional = &#34;select * from students where username like &#39;%&#34; &#43; username.get() &#43; &#34;%&#39;&#34;;
        //String sql = &#34;select * from students where username like ?&#34;;
        return jdbcTemplate.query(sqlWithOptional, ROW_MAPPER);
    }
```

宁可错杀一百不可放过一个，在代码审计中是如此，误报可能需要花费时间去筛选，但是漏报的损失则无法挽回

在 CodeQL 中，我们可以通过 isAdditionalTaintStep 方法来将断了的节点给它强制连接上

![image-20250412162413776](https://bu.dusays.com/2025/04/13/67fb541c7ae6d.png)

isAdditionalTaintStep 方法是CodeQL的类`TaintTracking::Configuration`提供的的方法，它的原型是：

```
override predicate isAdditionalTaintStep(DataFlow::Node node1, DataFlow::Node node2) {}
```

它的作用是将一个可控节点，比如A强制传递给另外一个节点B，那么节点B也就成了可控节点。

假设这里是 username.get() 这一步断掉了，我们可以强制让 username 流转到 username.get() ，代码为

```
/**
 * @id java/examples/vuldemo
 * @name Sql-Injection
 * @description Sql-Injection
 * @kind path-problem
 * @problem.severity warning
 */

 import java
 import semmle.code.java.dataflow.FlowSources
 import semmle.code.java.security.QueryInjection
 import DataFlow::PathGraph
 
 predicate isTaintedString(Expr expSrc, Expr expDest) {
     exists(Method method, MethodAccess call, MethodAccess call1 | expSrc = call1.getArgument(0) and expDest=call and call.getMethod() = method and method.hasName(&#34;get&#34;) and method.getDeclaringType().toString() = &#34;Optional&lt;String&gt;&#34; and call1.getArgument(0).getType().toString() = &#34;Optional&lt;String&gt;&#34;  )
 }
 
 class VulConfig extends TaintTracking::Configuration {
   VulConfig() { this = &#34;SqlInjectionConfig&#34; }
 
   override predicate isSource(DataFlow::Node src) { src instanceof RemoteFlowSource }
 
   override predicate isSanitizer(DataFlow::Node node) {
     node.getType() instanceof PrimitiveType or
     node.getType() instanceof BoxedType or
     node.getType() instanceof NumberType or
     exists(ParameterizedType pt| node.getType() = pt and pt.getTypeArgument(0) instanceof NumberType )
   }
 
   override predicate isSink(DataFlow::Node sink) {
     exists(Method method, MethodAccess call |
       method.hasName(&#34;query&#34;)
       and
       call.getMethod() = method and
       sink.asExpr() = call.getArgument(0)
     )
   }
 override predicate isAdditionalTaintStep(DataFlow::Node node1, DataFlow::Node node2) {
     isTaintedString(node1.asExpr(), node2.asExpr())
   }
 }
 
 
 from VulConfig config, DataFlow::PathNode source, DataFlow::PathNode sink
 where config.hasFlowPath(source, sink)
 select source.getNode(), source, sink, &#34;source&#34;
```

&gt; `expSrc = call1.getArgument(0)`：规定了污染源为第一个参数
&gt;
&gt; `expDest = call and call.getMethod() = method and method.hasName(&#34;get&#34;)`：规定了污染目标为 `xxx.get()`
&gt;
&gt; `method.getDeclaringType().toString() = &#34;Optional&lt;String&gt;&#34; and call1.getArgument(0).getType().toString() = &#34;Optional&lt;String&gt;&#34; `：规定了污染源和污染目标的类型为`Optional&lt;String&gt;`

### Lombok问题解决

Lombok是一个非常有名的Java类库，在开发中应该经常遇到，它通过简单的注解来帮助我们简化消除一些必须有但显得很臃肿的Java代码的工具，比如我们可以不用编写 getter，setter

demo

```
public class Student {
    private int id;
    private String username;

    public void setId(int id) {
        this.id = id;
    }

    public int getId() {
        return id;
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

}
```

当我们使用 Lombok 的注解，就不需要手动编写 getter 和 setter，以下代码等同于上面的 demo

```
import lombok.Data;

@Data
public class Student {
    private int id;
    private String username;
    private int sex;
    private int age;
}
```

可以看到用了 lombok 注解后就没有了 getter 和 setter，这就会导致 CodeQL 不能正常检测，source 到 sink 的链条断裂，从而造成漏报

有两种解决办法，第一种是在 pom.xml 中加入依赖，再重新编译即可

```
&lt;build&gt;  
   &lt;sourceDirectory&gt;target/generated-sources/delombok&lt;/sourceDirectory&gt;  
   &lt;testSourceDirectory&gt;target/generated-test-sources/delombok&lt;/testSourceDirectory&gt;  
&lt;plugins&gt;  
   &lt;plugin&gt;  
      &lt;groupId&gt;org.projectlombok&lt;/groupId&gt;  
      &lt;artifactId&gt;lombok-maven-plugin&lt;/artifactId&gt;  
      &lt;version&gt;1.18.20.0&lt;/version&gt;  
      &lt;executions&gt;  
         &lt;execution&gt;  
            &lt;id&gt;delombok&lt;/id&gt;  
            &lt;phase&gt;generate-sources&lt;/phase&gt;  
            &lt;goals&gt;  
               &lt;goal&gt;delombok&lt;/goal&gt;  
            &lt;/goals&gt;  
            &lt;configuration&gt;  
               &lt;addOutputDirectory&gt;false&lt;/addOutputDirectory&gt;  
               &lt;sourceDirectory&gt;src/main/java&lt;/sourceDirectory&gt;  
            &lt;/configuration&gt;  
         &lt;/execution&gt;  
         &lt;execution&gt;  
            &lt;id&gt;test-delombok&lt;/id&gt;  
            &lt;phase&gt;generate-test-sources&lt;/phase&gt;  
            &lt;goals&gt;  
               &lt;goal&gt;testDelombok&lt;/goal&gt;  
            &lt;/goals&gt;  
            &lt;configuration&gt;  
               &lt;addOutputDirectory&gt;false&lt;/addOutputDirectory&gt;  
               &lt;sourceDirectory&gt;src/test/java&lt;/sourceDirectory&gt;  
            &lt;/configuration&gt;  
         &lt;/execution&gt;  
      &lt;/executions&gt;  
   &lt;/plugin&gt;  
      &lt;plugin&gt;  
      &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;  
      &lt;artifactId&gt;spring-boot-maven-plugin&lt;/artifactId&gt;  
      &lt;/plugin&gt;  
   &lt;/plugins&gt;  
  
&lt;/build&gt;
```

第二种是有人在 issue 中提出的：https://github.com/github/codeql/issues/4984

```
# get a copy of lombok.jar
wget https://projectlombok.org/downloads/lombok.jar -O &#34;lombok.jar&#34;
# run &#34;delombok&#34; on the source files and write the generated files to a folder named &#34;delombok&#34;
java -jar &#34;lombok.jar&#34; delombok -n --onlyChanged . -d &#34;delombok&#34;
# remove &#34;generated by&#34; comments
find &#34;delombok&#34; -name &#39;*.java&#39; -exec sed &#39;/Generated by delombok/d&#39; -i &#39;{}&#39; &#39;;&#39;
# remove any left-over import statements
find &#34;delombok&#34; -name &#39;*.java&#39; -exec sed &#39;/import lombok/d&#39; -i &#39;{}&#39; &#39;;&#39;
# copy delombok&#39;d files over the original ones
cp -r &#34;delombok/.&#34; &#34;./&#34;
# remove the &#34;delombok&#34; folder
rm -rf &#34;delombok&#34;
```

两种方法都是去掉 lombok 注解，并还原 setter 和 getter 方法，实现手法不一样罢了，貌似第一种更方便，不知道为啥我这啥都没修改还是检测出来了

![image-20250412171148651](https://bu.dusays.com/2025/04/13/67fb53f5947ab.png)

### 批量化实现

以上 ql 规则可以成功跑出靶场的 sql 注入漏洞，我们也可以将其运用到其他项目上，先生成相应的数据库，然后再用写好的 ql 文件去分析就好，codeql 命令为

```
codeql database analyze /CodeQL/databases/micro-service-seclab /CodeQL/ql/java/ql/examples/demo --format=csv --output=/CodeQL/Result/micro-service-seclab.csv --rerun
```

## CodeQL进阶

### instanceof语法糖

```
a instanceof b
```

上面语句就是检测 a 对象是否是 b 类型

在遇到复杂的类型时，可能要用多个 exist 子查询语句，但是只用一个 instanceof 语句就好，而且只要匹配一个抽象类，就能匹配到这个抽象类下的所有子类，比如

```
class MyCustomSource extends RemoteFlowSource
```

如果 x 为 MyCustomSource 类型，则 `x instanceof RemoteFlowSource` 会返回true

### 递归

CodeQL里面的递归调用语法是：在谓词方法的后面跟*或者&#43;，来表示调用0次以上和1次以上（和正则类似），0次会打印自己。

demo

```
package org.example;

public class hello {
        public class StudentService {

            class innerOne {
                public innerOne(){}

                class innerTwo {
                    public innerTwo(){}

                    public String Nihao() {
                        return &#34;Nihao&#34;;
                    }
                }
                public String Hi(){
                    return &#34;hello&#34;;
                }
            }

        }
    }

```

此时如果想要根据innerTwo类定位到最外层的 hello 类

我们可以用以下 ql 查询语句来获得，调用两次 getEnclosingType 方法即可

```
import java
 
from Class classes
where classes.getName().toString() = &#34;innerTwo&#34;
select classes.getEnclosingType().getEnclosingType().getEnclosingType()   // getEnclosingtype获取作用域
```

但是实际情况我们不知道要调用几次，而且这样写也比较麻烦

这时候就可以用到递归了，我们在调用方法后面加*(从自身开始调用)或者&#43;(从上一级开始调用)，来解决此问题。

```
from Class classes
where classes.getName().toString() = &#34;innerTwo&#34;
select classes.getEnclosingType&#43;()   // 获取作用域
```

`&#43;` 是从上一级开始调用

![image-20250412182154821](https://bu.dusays.com/2025/04/13/67fb53f532720.png)

`*` 是从自身开始调用

![image-20250412182236018](https://bu.dusays.com/2025/04/13/67fb53f532672.png)

自己封装方法来实现递归调用也是可以的，比如这里我这只想找相差一层的类

```
import java
 
RefType demo(Class classes) {
    result = classes.getEnclosingType().getEnclosingType()
}
 
from Class classes
where classes.getName().toString() = &#34;innerTwo&#34;
select demo*(classes)   // 获取作用域
```

![image-20250412182753822](https://bu.dusays.com/2025/04/13/67fb53f538503.png)

### 强制类型转换

在 CodeQL 中可以用 getType() 来对返回结果做强制类型转换

查询下当前数据库中所有的参数及其类型

```
import java
 
from Parameter param
select param, param.getType()
```

![image-20250413132629826](https://bu.dusays.com/2025/04/13/67fb53f52aa76.png)

可以看到有5k多条而且有不同类型的参数，强制转换下，只留下是整型的参数

![image-20250413132902211](https://bu.dusays.com/2025/04/13/67fb53f50ae35.png)

只有1k多条了，而且只含有整型的

## 总结

综上可以知道CodeQL的强大了，稍微入了下门，知道怎么用了，难点就在于规则的编写，最主要的就是source，sink和中间Sanitizer的编写

接下来就是找案例来练手了

## 参考

https://www.freebuf.com/articles/web/283795.html

https://www.ascotbe.com/2024/12/27/CodeQL/

https://drun1baby.top/2023/09/03/CodeQL-%E5%85%A5%E9%97%A8/


---

> 作者: 6s6  
> URL: http://localhost:1313/posts/28dc684/  

