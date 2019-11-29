### 一、前言

Dubbo是拓展性极好的框架，其采用 Microkernel + Plugin 模式，Microkernel 只负责组装 Plugin，Dubbo 自身的功能也是通过扩展点实现的，也就是 Dubbo 的所有功能点都可被用户自定义扩展所替换。

本篇作为 Dubbo 学习记录的第一篇，便以 Dubbo SPI 一斑窥豹。

### 二、Java SPI

SPI机制（Service Provider Interface)其实源自服务提供者框架（Service Provider Framework，参考【EffectiveJava】page6)，是一种将服务接口与服务实现分离以达到解耦、大大提升了程序可扩展性的机制。引入服务提供者就是引入了spi接口的实现者，通过本地的注册发现获取到具体的实现类，轻松可插拔。

Java SPI 实际上是“基于接口的编程＋策略模式＋配置文件”组合实现的动态加载机制。

1. 定义一个接口。
2. 编写此接口的实现类。
3. 在 src/main/resources/ 下建立 /META-INF/services 目录， 新增一个以接口命名的文件。
4. 文件中写要使用的实现类，每行一个类。
5. 使用 ServiceLoader 来加载配置文件中指定的实现。

代码结构：

```java
+--src
|   +--com
|       +--test
|           +--spi
|               --RunService
|               --RunServiceImpl
```

使用方式：

```java
public class SPIMain {
    public static void main(String[] args) {
        ServiceLoader<RunService> runService = ServiceLoader.load(RunService.class);
        for (runService r : runService) {
            r.doRun();
        }
    }
}
```

dubbo中重新实现了一套SPI机制，做了一些优化和改进，包括不限于拓展类被调用得到时候再加载，自动包装机制，加载出错的异常通知等特性。

代码在这里：

![](https://github.com/JiangYang4Github/WebLog/blob/master/2019/%E6%8A%80%E6%9C%AF/Dubbo/SPI/20191126102423.png?raw=true)

### 三、SPI注解

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE})
public @interface SPI {
    String value() default "";
}
```

修饰在 class 上面，在 Dubbo 中都是在修饰接口，制定接口的默认实现类，如下，我们可以看到声明了使用 javasist 作为代理类的动态编译工厂的默认实现方式。

```java
@SPI("javassist")
public interface ProxyFactory {
    ...
}
```

如下，Dubbo中内置了各种协议，如DubboProtocol，HttpProtocol，HessianProtocol等等。我们可以看到声明了rpc模块默认 protocol 实现为 **DubboProtocol**。

```java
@SPI("dubbo")
public interface Protocol {
	...
}
```

### 四、Adaptive注解

使用此注解可以动态的通过URL中的参数来确定要使用哪个具体的实现类，从而解决自动加载中的实例注入问题。如下，注解可以放在类（以及枚举和接口）和方法上。

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Adaptive {    
    String[] value() default {};
}
```

示例代码，结构如下。

```java
+--src
|   +--com
|       +--test
|           +--spi
|               --SimpleExt
|               --SimpleExtImpl1
|               --SimpleExtImpl2
```

其中接口与实现类的代码依次如下。

```java
@SPI("impl1")
public interface SimpleExt {
    
    @Adaptive
    String echo(URL url, String s);
    
    @Adaptive({"key1", "key2"})
    String yell(URL url, String s);

}
```

```java
public class SimpleExtImpl1 implements SimpleExt {
    
    public String echo(URL url, String s) {
        return "Ext1Impl1-echo";
    }

    public String yell(URL url, String s) {
        return "Ext1Impl1-yell";
    }
}
```

```java
public class SimpleExtImpl2 implements SimpleExt {
    
    public String echo(URL url, String s) {
        return "Ext1Impl2-echo";
    }

    public String yell(URL url, String s) {
        return "Ext1Impl2-yell";
    }
}
```

好了，我们看到接口 **SimpleExt** 的SPI注解的值为**impl1**，这就说明使用**SimpleExtImpl1**作为这个接口的默认实现，但是我们发现其两个方法都用注解**Adaptive**修饰了，这就说明我们可以通过在URL中添加参数来动态的切换实现方法，我们来看一段代码。

```java
SimpleExt ext = ExtensionLoader.getExtensionLoader(SimpleExt.class).getAdaptiveExtension();

Map<String, String> map = new HashMap<String, String>();
URL url = new URL("p1", "1.2.3.4", 1010, "path1", map);

String echo = ext.echo(url, "haha");
assertEquals("Ext1Impl1-echo", echo);
```

其中的 **ExtensionLoader** 我稍后会详细说，它类似于上文提到的 Java SPI 中的 **ServiceLoader** 。

我们来切换实现方法，可以这样：

```java
SimpleExt ext = ExtensionLoader.getExtensionLoader(SimpleExt.class).getAdaptiveExtension();

Map<String, String> map = new HashMap<String, String>();
map.put("simple.ext", "impl2");
URL url = new URL("p1", "1.2.3.4", 1010, "path1", map);

String echo = ext.echo(url, "haha");
assertEquals("Ext1Impl2-echo", echo);
```

还有如下几种设置方法。

```java
//以Adaptive注解值为key，value为实现类进行切换
map.put("key1", "impl2");
map.put("key2", "impl2");

//向url对象中追加参数
url.addParameter("key1", "impl2"); 

//构造器制定
url = new URL("impl2", "1.2.3.4", 1010, "path1", map);

//设置协议
URL url = new URL(null, "1.2.3.4", 1010, "path1", map);
url = url.setProtocol("impl2");
```

### 五、Activate注解

自动激活注解，通过group和value配置激活条件，被注解的扩展点在满足某种条件时会被激活，在Dubbo中更多的用来做不同条件下激活不同Filter进行处理这个场景。

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD})
public @interface Activate {
	
    //URL中的分组如果匹配将会被激活
    String[] group() default {};

    //在URL中查找此Key数组
    String[] value() default {};
    
    //表示哪些拓展点要在当前拓展点之前激活
    String[] before() default {};

    //表示哪些拓展点要在当前拓展点之后激活
    String[] after() default {};

    //排序
    int order() default 0;
    
}
```

示例代码就不贴了，大概思路就是不同条件，调用 **ExtensionLoader** 的 **getActivateExtension** 方法是会返回满足注解条件的、一定顺序的拓展点实现集合。我们现在来看 **ExtensionLoader** 这个类。

### 六、ExtensionLoader 

ExtensionLoader 有三个关键方法，**getExtension**、**getAdaptiveExtension**和**getActivateExtension**，如上文所说，他们的作用分别是获取普通拓展点、获取自适应拓展点和获取自动激活拓展点。

#### 1、获取普通拓展点

```java
public T getExtension(String name) {
    if (name == null || name.length() == 0)
        throw new IllegalArgumentException("Extension name == null");
    if ("true".equals(name)) {
        return getDefaultExtension();
    }
    Holder<Object> holder = cachedInstances.get(name);
    if (holder == null) {
        cachedInstances.putIfAbsent(name, new Holder<Object>());
        holder = cachedInstances.get(name);
    }
    Object instance = holder.get();
    if (instance == null) {
        synchronized (holder) {
            instance = holder.get();
            if (instance == null) {
                //关键方法，创建拓展点
                instance = createExtension(name);
                holder.set(instance);
            }
        }
    }
    return (T) instance;
}
```

```java
private T createExtension(String name) {
    Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    try {
        T instance = (T) EXTENSION_INSTANCES.get(clazz);
        if (instance == null) {
            //构造，注册
            EXTENSION_INSTANCES.putIfAbsent(clazz, clazz.newInstance());
            instance = (T) EXTENSION_INSTANCES.get(clazz);
        }
        //对实例进行数据注入
        injectExtension(instance);
        //遍历当前拓展点的包装类，并且将当前name对应的拓展点实例作为参数传入该包装类实例的构造函数，将该包装类实例化
        Set<Class<?>> wrapperClasses = cachedWrapperClasses;
        if (wrapperClasses != null && !wrapperClasses.isEmpty()) {
            for (Class<?> wrapperClass : wrapperClasses) {
                instance = injectExtension((T) wrapperClass.getConstructor(type).newInstance(instance));
            }
        }
        //返回层层包装之后的实例，完成链式调用
        return instance;
    } catch (Throwable t) {
        ...
    }
}
```

这里有个知识点，ExtensionLoader 在加载扩展点时，如果加载到的扩展点有拷贝构造函数，则判定为扩展点 Wrapper 类。将当前拓展点实例作为参数传入该包装类实例的构造函数，将该包装类实例化。这说明了，我们可以在这里做 **AOP ** 的逻辑处理。

#### 2、获取自适应拓展点

```java
public T getAdaptiveExtension() {
    Object instance = cachedAdaptiveInstance.get();
    if (instance == null) {
        if (createAdaptiveInstanceError == null) {
            synchronized (cachedAdaptiveInstance) {
                instance = cachedAdaptiveInstance.get();
                if (instance == null) {
                    try {
                        //关键方法，创建自适应拓展点实例，跟下去
                        instance = createAdaptiveExtension();
                        cachedAdaptiveInstance.set(instance);
                    } catch (Throwable t) {
                        createAdaptiveInstanceError = t;
                        throw new IllegalStateException("fail to create adaptive instance: " + t.toString(), t);
                    }
                }
            }
        } else {
            ...
        }
    }

    return (T) instance;
}
```

```java
private T createAdaptiveExtension() {
        try {
            //injectExtension方法作用同上，进行注入
            //我们跟进getAdaptiveExtensionClass方法
            return injectExtension((T) getAdaptiveExtensionClass().newInstance());
        } catch (Exception e) {
            throw new IllegalStateException("Can not create adaptive extension " + type + ", cause: " + e.getMessage(), e);
        }
    }
```

```java
private Class<?> getAdaptiveExtensionClass() {
    	//检查一下缓存中是否存在这个Class
        getExtensionClasses();
        if (cachedAdaptiveClass != null) {
            return cachedAdaptiveClass;
        }
    	//关键方法，我们跟进createAdaptiveExtensionClass方法
        return cachedAdaptiveClass = createAdaptiveExtensionClass();
}
```

```java
private Class<?> createAdaptiveExtensionClass() {
    	//下面的这个方法，通过字符串拼接，生成名为typeName+$Adaptive的实例代码
        String code = createAdaptiveExtensionClassCode();
        ClassLoader classLoader = findClassLoader();
    	//compiler接口有@SPI("javassist")修饰，即默认使用javasist进行
        com.alibaba.dubbo.common.compiler.Compiler compiler = ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.compiler.Compiler.class).getAdaptiveExtension();
        return compiler.compile(code, classLoader);
}
```

这段代码主要干了几件事。

- 调用 createAdaptiveExtensionClassCode 方法生成当前类的代码字符串
- 获取类加载器。
- 通过 ExtensionLoader 取得一个编译器，默认是javasist，进行对代码进行编译。

生成的代码大概是这样：

```java
package com.alibaba.dubbo.common.extensionloader.ext1;
import com.alibaba.dubbo.common.extension.ExtensionLoader;
public class SimpleExt$Adaptive implements com.alibaba.dubbo.common.extensionloader.ext1.SimpleExt {
public java.lang.String echo(com.alibaba.dubbo.common.URL arg0, java.lang.String arg1) {
if (arg0 == null) throw new IllegalArgumentException("url == null");
com.alibaba.dubbo.common.URL url = arg0;
String extName = url.getParameter("simple.ext", "impl1");
if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.common.extensionloader.ext1.SimpleExt) name from url(" + url.toString() + ") use keys([simple.ext])");
com.alibaba.dubbo.common.extensionloader.ext1.SimpleExt extension = (com.alibaba.dubbo.common.extensionloader.ext1.SimpleExt)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.extensionloader.ext1.SimpleExt.class).getExtension(extName);
return extension.echo(arg0, arg1);
}
public java.lang.String yell(com.alibaba.dubbo.common.URL arg0, java.lang.String arg1) {
if (arg0 == null) throw new IllegalArgumentException("url == null");
com.alibaba.dubbo.common.URL url = arg0;
String extName = url.getParameter("key1", url.getParameter("key2", "impl1"));
if(extName == null) throw new IllegalStateException("Fail to get extension(com.alibaba.dubbo.common.extensionloader.ext1.SimpleExt) name from url(" + url.toString() + ") use keys([key1, key2])");
com.alibaba.dubbo.common.extensionloader.ext1.SimpleExt extension = (com.alibaba.dubbo.common.extensionloader.ext1.SimpleExt)ExtensionLoader.getExtensionLoader(com.alibaba.dubbo.common.extensionloader.ext1.SimpleExt.class).getExtension(extName);
return extension.yell(arg0, arg1);
}
}
```

可以看到，类使用的都是完整路径。

我整理一下代码，去掉完整路径，然后再去掉一下判断逻辑，就非常清晰了。

```java
package com.alibaba.dubbo.common.extensionloader.ext1;
import com.alibaba.dubbo.common.extension.ExtensionLoader;
public class SimpleExt$Adaptive implements SimpleExt {
	public String echo(URL arg0, String arg1) {
        URL url = arg0;
        //根据URL获取到对应的拓展名
        String extName = url.getParameter("simple.ext", "impl1");
        //根据拓展名获取对应的拓展点实现类
        SimpleExt extension = (SimpleExt)ExtensionLoader.getExtensionLoader(SimpleExt.class)
                                .getExtension(extName);
        return extension.echo(arg0, arg1);
	}
    public String yell(URL arg0, String arg1) {
        URL url = arg0;
        //根据URL获取到对应的拓展名
        String extName = url.getParameter("key1", url.getParameter("key2", "impl1"));
        //根据拓展名获取对应的拓展点实现类
        SimpleExt extension = (SimpleExt)ExtensionLoader.getExtensionLoader(SimpleExt.class)
                                .getExtension(extName);
        return extension.yell(arg0, arg1);
    }
}
```

也就是说，生成的动态代理类，也是在运行时动态的从URL中获取参数，然后使用 **ExtensionLoader** 获取实现类进行调用。

#### 3、自动激活拓展点

还记得在获取普通拓展点时，我们是调用 **createExtension** 方法。

```java
private T createExtension(String name) {
	Class<?> clazz = getExtensionClasses().get(name);
    if (clazz == null) {
        throw findException(name);
    }
    ...
}
```

其中 **getExtensionClasses** 方法会调用 **loadExtensionClasses** 方法进行class信息的读取。

```java
private Map<String, Class<?>> loadExtensionClasses() {
    final SPI defaultAnnotation = type.getAnnotation(SPI.class);
    if (defaultAnnotation != null) {
        String value = defaultAnnotation.value();
        if ((value = value.trim()).length() > 0) {
            String[] names = NAME_SEPARATOR.split(value);
            if (names.length == 1) cachedDefaultName = names[0];
        }
    }
    Map<String, Class<?>> extensionClasses = new HashMap<String, Class<?>>();
    //根据传入类型，对 META-INF/services/、META-INF/dubbo/、META-INF/dubbo/internal/ 路径下的拓展点实现类进行加载
    loadDirectory(extensionClasses, DUBBO_INTERNAL_DIRECTORY);
    loadDirectory(extensionClasses, DUBBO_DIRECTORY);
    loadDirectory(extensionClasses, SERVICES_DIRECTORY);
    return extensionClasses;
}
```

```java
private void loadDirectory(Map<String, Class<?>> extensionClasses, String dir) {
    String fileName = dir + type.getName();
    Enumeration<java.net.URL> urls;
    ClassLoader classLoader = findClassLoader();
    if (classLoader != null) {
        urls = classLoader.getResources(fileName);
    } else {
        urls = ClassLoader.getSystemResources(fileName);
    }
    if (urls != null) {
        while (urls.hasMoreElements()) {
            java.net.URL resourceURL = urls.nextElement();
            //获取jar下对应文件的class，跟进去，关键是调用loadClass方法
            loadResource(extensionClasses, classLoader, resourceURL);
        }
    }
}
```

```java
private void loadClass(Map<String, Class<?>> extensionClasses, java.net.URL resourceURL, Class<?> clazz, String name) throws NoSuchMethodException {
        if (clazz.isAnnotationPresent(Adaptive.class)) {
            if (cachedAdaptiveClass == null) {
                cachedAdaptiveClass = clazz;
            } else if (!cachedAdaptiveClass.equals(clazz)) {
                throw new IllegalStateException("More than 1 adaptive class found: "
                        + cachedAdaptiveClass.getClass().getName()
                        + ", " + clazz.getClass().getName());
            }
        } else if (isWrapperClass(clazz)) {
            Set<Class<?>> wrappers = cachedWrapperClasses;
            if (wrappers == null) {
                cachedWrapperClasses = new ConcurrentHashSet<Class<?>>();
                wrappers = cachedWrapperClasses;
            }
            wrappers.add(clazz);
        } else {
            clazz.getConstructor();
            if (name == null || name.length() == 0) {
                name = findAnnotationName(clazz);
            }
            String[] names = NAME_SEPARATOR.split(name);
            if (names != null && names.length > 0) {
                //判断是否被@Active修饰的拓展点实现类，如果是，则使用cachedActivates缓存
                Activate activate = clazz.getAnnotation(Activate.class);
                if (activate != null) {
                    cachedActivates.put(names[0], activate);
                }
                for (String n : names) {
                    if (!cachedNames.containsKey(clazz)) {
                        cachedNames.put(clazz, n);
                    }
                    Class<?> c = extensionClasses.get(n);
                    if (c == null) {
                        extensionClasses.put(n, clazz);
                    } else if (c != clazz) {
                        throw new IllegalStateException("Duplicate extension " + type.getName() + " name " + n + " on " + c.getName() + " and " + clazz.getName());
                    }
                }
            }
        }
    }

```

而通过 **ExtensionLoader** 的 **getActivateExtension** 的方法获取的实现原理并不复杂，主要是：

- 去缓存尝试取。
- 根据Activate注解的值，从 **cachedActivates** 中拿到所有拓展进行匹配和排序。
- 根据用户的自定义拓展点配置，进行整体排序
- 返回拓展点集合

### 七、动态编译

这个技术可以通过操作Java字节码的方式，在JVM中生成新类或者对已经加载的类动态添加元素。如上文所说，我们Dubbo中默认使用的动态编译使用的 **Javasist**，我们先来看一段demo。

```java
//初始化类池
ClassPool pool = ClassPool.getDefault();
//创建类
CtClass ct = pool.makeClass("hello world");
//添加方法
CtMethod helloM=CtNewMethod.make("public void hello(){ System.out.println("hello world");}",ct);
ct.addMethod(helloM);
//创建类
Class aClass = ct.toClass();
//实例化
Object o = aClass.newInstance();
//反射调用
Method m = aClass.getDeclaredMethod("test",null);
m.invoke(o,null);
```

用起来还算简单清晰的，像是创建一个字段，声明一个实现接口或者上级父类，Javasist 都提供了对应的操作方法。

我们会把生成的拓展点代码字符串，经过一系列的正则匹配，取出引用的包数据，实现的接口，继承的父类等等类描述数据，调用 **Javasist** 提供的方法进行类的动态构造与实例创建。

### 八、最后

本篇我们分析了 Dubbo 框架中比较底层的技术，优秀的分层，抽象和 SPI 的自研提供了 Dubbo 极高的拓展性。

下一篇会把之前搁置的分布式任务调度框架的终篇先整理出来，Dubbo 这边下一篇计划是 **registry** 层这个知识点的分析和源码跟读。

喜欢的可以关注我的公众号「**江飞杰**」第一时间阅读（会更新的比较快），里面也有自己的一些和技术无关的读书笔记与生活随感，欢迎大家来关注。

![](https://user-gold-cdn.xitu.io/2019/11/17/16e793daa95189b7?w=430&h=430&f=jpeg&s=40918)