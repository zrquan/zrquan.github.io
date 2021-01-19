+++
title = "Ysoserial-Groovy1"
publishDate = 2021-01-19
tags = ["java", "反序列化"]
draft = false
+++

最近想要加强一下自己的代码能力和漏洞分析能力，于是想到学习 ysoserial 的代码。文章
主要记录自己调试的过程，理清利用链的思路，对一些基础知识不会做过多解释。

<!--more-->


## 源代码 {#源代码}

一些导入包、注释、声明之类的就忽略了，下面给出主要的 getObject 函数，该函数返回一
个恶意对象(AnnotationInvocationHandler)，当反序列化这个对象时就能执行命令。

```java
public InvocationHandler getObject(final String command) throws Exception {
		final ConvertedClosure closure = new ConvertedClosure(new MethodClosure(command, "execute"), "entrySet");

		final Map map = Gadgets.createProxy(closure, Map.class);

		final InvocationHandler handler = Gadgets.createMemoizedInvocationHandler(map);

		return handler;
}
```

可以看到代码很少，除了 `return` 一共就三行，在调试前先简要说一下每行代码的作用：

1.  创建一个 closure 对象，将我想执行的命令(command)保存到该对象的属性中。该类实现
    了 InvocationHandler 接口，所以可以作为动态代理的处理器；

2.  创建一个 map 对象，实则是一个动态代理对象，调用这个对象的方法时会代理
    到上面的 closure 对象的处理逻辑中(invoke)；

3.  创建最终的恶意对象，把上面的动态代理对象 map 保存到属性中，当反序列化 handler 时,
    就会通过层层调用最终执行 closure 里面的命令。


## 构造 payload {#构造-payload}

首先看一下 ConvertedClosure 类的构造函数：

{{< figure src="/ox-hugo/2021-01-18_17-58-12_screenshot.png" >}}

再看看其父类 ConversionHandler 的构造函数：

{{< figure src="/ox-hugo/2021-01-18_17-59-30_screenshot.png" >}}

接着跟进 MethodClosure 类的初始化过程：

{{< figure src="/ox-hugo/2021-01-18_18-12-17_screenshot.png" >}}

{{< figure src="/ox-hugo/2021-01-18_18-12-54_screenshot.png" >}}

以上过程的重点都是一些属性赋值的操作，而攻击反序列化过程就是要利用这些受我控制的
属性去影响一些代码片段，而这些代码片段需要构成完整的利用链。

这里先记录一下 closure 对象的一些关键属性，再接着看下去：

```java
closure.methodName = "entrySet";
closure.delegate = new MethodClosure(command, "execute");
closure.delegate.method = "execute";
closure.delegate.owner = command;
```

接着是调用 `Gadgets.createProxy` 方法，通过 `Proxy.newProxyInstance` 创建一个 Map 类型
的代理对象：

{{< figure src="/ox-hugo/2021-01-18_19-28-29_screenshot.png" >}}

newProxyInstance 的第二个参数(allIfaces)是代理类要实现的接口，这里只有 `Map.class`
，所以可以通过 `Class.cast` 将类型转换成 Map。

第三个参数(ih)是 InvocationHandler 类的对象，也就是上面的 closure，无论调用代理
对象的那个方法，都会执行 InvocationHandler 对象中重写的 invoke 方法。

最后再看看 `Gadgets.createMemoizedInvocationHandler` ：

{{< figure src="/ox-hugo/2021-01-18_19-46-26_screenshot.png" >}}

getFirstCtor 方法可以通过反射得到 AnnotationInvocationHandler 类的构造函数，之所以
通过反射机制，是因为 AnnotationInvocationHandler 的构造函数没有 `public` 修饰，不能
通过 `new` 直接访问。

从 AnnotationInvocationHandler 的构造函数看到，我传入的 map 对象将会赋值给
memberValues 属性：

{{< figure src="/ox-hugo/2021-01-18_19-54-56_screenshot.png" >}}

到这里为止，payload 的构造基本完成了，当这个 AnnotationInvocationHandler 对象被反序
列化时，上述设置好的属性就会影响过程中的一些代码片段，最终导致命令执行。


## 反序列化过程 {#反序列化过程}

反序列化的入口点自然是 readObject 方法，我们直接看看
`AnnotationInvocationHandler.readObject()` 都做了什么操作：

{{< figure src="/ox-hugo/2021-01-18_20-28-06_screenshot.png" >}}

看红色框部分就知道执行了 memberValues 属性的 entrySet 方法。

而这个 memberValues 则是我传入的动态代理对象 map，调用它的方法实则执行
`InvocationHandler.invoke()` 。

{{< figure src="/ox-hugo/2021-01-18_20-34-26_screenshot.png" >}}

从调用栈可以知道，invoke 方法在 ConvertedClosure 的父类 ConversionHandler 中，在执行
了一些版本判断相关的逻辑后，又跳到了 ConvertedClosure 的 invokeCustom 方法。

```java
public Object invokeCustom(Object proxy, Method method, Object[] args) throws Throwable {
    return this.methodName != null && !this.methodName.equals(method.getName()) ? null : ((Closure)this.getDelegate()).call(args);
}
```

这里 `&&` 的优先级高于三目运算符，当 `this.methodName` 既不等于空也不等于
`method.getName()` 时返回 null，不然就会执行
`((Closure)this.getDelegate()).call(args)` 。

我们回顾一下在构造 payload 时，设置的一些属性：

```java
closure.methodName = "entrySet";
closure.delegate = new MethodClosure(command, "execute");
closure.delegate.method = "execute";
closure.delegate.owner = command;
```

一开始就将 ConvertedClosure 对象的 methodName 属性设置为 entrySet，正好就是
`AnnotationInvocationHandler.readObject()` 中所执行的方法。

`this.getDelegate()` 会返回什么也很显而易见了，那么跟进一下 `call()` ：

{{< figure src="/ox-hugo/2021-01-18_22-19-16_screenshot.png" >}}

直接进入到 MetaClassImpl 类的 invokeMethod 方法中，紧接着调用一个重载
的 invokeMethod 方法。

这个函数代码比较多，但是我们只需要跟踪 object 参数，因为需要执行的命令在这个参数对
象的属性中。

{{< figure src="/ox-hugo/2021-01-18_22-25-49_screenshot.png" >}}

运行到以下代码，我想要执行的命令被拿了出来：

{{< figure src="/ox-hugo/2021-01-18_22-36-32_screenshot.png" >}}

ownerMetaClass 依然是 MetaClassImpl 类，所以这里递归调用了 invokeMethod 方法，不过
MethodClosure 对象变成了我要执行的命令，也就是 String。

因此这一次不会进入 `if (isClosure)` ，而是执行到以下 return 语句中

```text
return method != null ? method.doMethodInvoke(object, arguments) : this.invokePropertyOrMissing(object, methodName, originalArguments, fromInsideClass, isCallToSuper);
```

命令字符串从 doMethodInvoke 进入到 `ProcessGroovyMethods.execute` ，最终导致命令执
行：

{{< figure src="/ox-hugo/2021-01-18_22-47-03_screenshot.png" >}}
