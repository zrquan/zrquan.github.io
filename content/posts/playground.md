+++
title = "测试页面"
publishDate = 2021-01-09
tags = ["test"]
draft = true
+++

<!--more-->


## test {#test}

{{< code language="python" title="python demo" expand="➕" collapse="➖" isCollapsed="true" >}}

if __name__ == '__main__':
    print('hello, hugo.')

{{< /code >}}


## music {#music}

{{< music 188550 >}}


## foonote {#foonote}


### 面向属性编程 {#面向属性编程}

当我们可以控制对象的属性值，并利用这些属性值影响代码的执行过程时，这一过程就称之
为“面向属性编程（property-oriented programming，POP）”。POP 利用点指的是一个代码
片段，我们可以修改某些对象的属性来影响这个代码片段，使其满足我们的特定需求。许多
情况下，我们需要串联多个利用点才能形成完整的利用程序。我们可以将其看成高级
ROP（return-oriented programming，面向返回编程）技术，只不过 ROP 需要将某个值推送
到栈上，而我们通过 POP 利用点可以将某些数据写入到文件中。

这里非常重要的一点是，反序列化漏洞利用过程不需要将类或者代码发送给服务器来执行。
我们只是简单发送了类中的某些属性，服务器对这些属性比较敏感， **通过这些属性，我们就
能控制负责处理这些属性的已有代码。** 因此，漏洞利用成功与否取决于我们对代码的熟悉程
度，我们需要充分了解通过反序列化操作能被操控的那部分代码。这也是反序列化漏洞利用
过程中的难点所在。