+++
title = "Ysoserial-CommonsCollections1"
tags = ["java", "反序列化"]
draft = true
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

[^fn:1]: <https://xz.aliyun.com/t/7031>
