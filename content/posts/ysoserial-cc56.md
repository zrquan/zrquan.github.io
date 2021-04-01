+++
title = "Ysoserial-CommonsCollections5/6"
publishDate = 2021-04-01T00:00:00+08:00
tags = ["java", "unserialize"]
draft = false
+++

<!--more-->

在之前 [CommonsCollections1]({{< relref "ysoserial-cc1" >}}) 的文章中有分析怎么利用几个 Transformer 类来执行系统命令，以及通过 LazyMap#get 来触发。所以本文不再赘述这一部分内容，只要知道最终目的是执行
LazyMap#get 方法就行了。


## CommonsCollections5 {#commonscollections5}

这条链需要用到 BadAttributeValueExpException 中重写的 readObject 方法，我搜了一下没找到这个方法具体是在哪个版本加上去的，下面测试用的是 `JDK 8u212` 。

首先看一下 getObject 方法：

{{< figure src="/ox-hugo/2021-03-31_18-09-05_screenshot.png" >}}

前面构造 transformerChain 的过程和 cc1 一样，触发点同样在 lazyMap#get 方法中。区别主要在红框的部分——生成 TiedMapEntry 对象，并将 lazyMap 赋值给它的 map 属性；然后创建 BadAttributeValueExpException 对象，通过反射将 TiedMapEntry 对象赋值给它的私有属性 val。

{{< figure src="/ox-hugo/val.png" >}}

在进行反序列化时，执行 BadAttributeValueExpException 重写的 readObject 方法：

{{< figure src="/ox-hugo/2021-04-01_13-40-30_screenshot.png" >}}

如果没有开启安全管理器(`System.getSecurityManager() == null`)，会执行
TiedMapEntry#toString 方法。TiedMapEntry 的关键方法如下：

```java
public Object getValue() {
    return this.map.get(this.key);
}
...
public String toString() {
    return this.getKey() + "=" + this.getValue();
}
```

可见最终会执行我们构造的 lazyMap 的 get 方法，导致命令执行。


## CommonsCollections6 {#commonscollections6}

前半段和上面一样，先构造一个 TiedMapEntry 类对象，想办法触发它的 getValue 或者
toString 方法。

首先创建一个 HashSet 类对象，通过反射拿到它的私有属性 map：

{{< figure src="/ox-hugo/2021-04-01_14-21-07_screenshot.png" >}}

这个 map 属性是 HashMap 类的，再通过反射获取其 table 属性：

{{< figure src="/ox-hugo/2021-04-01_14-26-00_screenshot.png" >}}

获取数组的第一个非空元素 node，将构造好的 TiedMapEntry 对象赋值给它的 key 属性：

{{< figure src="/ox-hugo/2021-04-01_14-27-33_screenshot.png" >}}

构造好的 HashSet 对象结构如下：

{{< figure src="/ox-hugo/hashset.png" >}}

从上图可以看到 map 和 table 两个都是 transient 属性，也就是不会被序列化，那设置它们有什么意义呢？我们看一下 HashSet#writeObject 方法：

```java
private void writeObject(java.io.ObjectOutputStream s)
    throws java.io.IOException {
    // Write out any hidden serialization magic
    s.defaultWriteObject();

    // Write out HashMap capacity and load factor
    s.writeInt(map.capacity());
    s.writeFloat(map.loadFactor());

    // Write out size
    s.writeInt(map.size());

    // Write out all elements in the proper order.
    for (E e : map.keySet())
        s.writeObject(e);
}
```

关键在最后的 for 循环，主动将我们构造的 TiedMapEntry 对象写到输出流中，所以不受
transient 的影响。

反序列化 HashSet 时，在 readObjet 方法的最后会获取 TiedMapEntry 对象(key)，将它添加到一个 HashMap 中。

{{< figure src="/ox-hugo/2021-04-01_14-59-21_screenshot.png" >}}

跟进发现会通过 `key.hashCode()` 获取 key 的哈希值，而在 TiedMapEntry#hashCode 方法中触发了 lazyMap#get 方法：

{{< figure src="/ox-hugo/2021-04-01_23-27-34_screenshot.png" >}}

最终导致命令执行。


## 总结 {#总结}

之所以把 cc5 和 cc6 两条链放在一起讲，是因为它们和 cc1 一样都利用
ChainedTransformer 构造反射链来执行命令，而且都利用了 TiedMapEntry 类。

链的前半部分逻辑就比较简单了，比较有趣的知识点是 HashSet 的 writeObject 方法会序列化它的所有 key，而 HashMap 的 writeObject 会序列化它的所有 key 和 value，所以即使属性用 transient 修饰也不影响利用。
