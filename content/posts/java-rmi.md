+++
title = "Java RMI"
tags = ["java"]
draft = true
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


### Server --> Registry {#server-registry}

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


### Client --> Server {#client-server}

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


### Server/Client --> Registry {#server-client-registry}


### Registry --> Server/Client {#registry-server-client}


### Server --> Client {#server-client}


### Client --> Server {#client-server}


## JEP 290 {#jep-290}
