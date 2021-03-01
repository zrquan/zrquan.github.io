+++
title = "playground"
publishDate = 2021-01-09T00:00:00+08:00
tags = ["test"]
draft = true
+++

本文用于测试

<!--more-->


## Details and Summary {#details-and-summary}

<details>
<summary>
python code
</summary>
<p class="details">

```python
if __name__ == '__main__':
    print('hello, hugo.')
```
</p>
</details>


## foonote test {#foonote-test}


### 面向属性编程[^fn:1] {#面向属性编程}

当我们可以控制对象的属性值，并利用这些属性值影响代码的执行过程时，这一过程就称之为“面向属性编程（property-oriented programming，POP）”。POP 利用点指的是一个代码片段，我们可以修改某些对象的属性来影响这个代码片段，使其满足我们的特定需求。许多情况下，我们需要串联多个利用点才能形成完整的利用程序。我们可以将其看成高级
ROP（return-oriented programming，面向返回编程）技术，只不过 ROP 需要将某个值推送到栈上，而我们通过 POP 利用点可以将某些数据写入到文件中。

这里非常重要的一点是，反序列化漏洞利用过程不需要将类或者代码发送给服务器来执行。我们只是简单发送了类中的某些属性，服务器对这些属性比较敏感，通过这些属性，我们就能控制负责处理这些属性的已有代码。因此，漏洞利用成功与否取决于我们对代码的熟悉程度，我们需要充分了解通过反序列化操作能被操控的那部分代码。这也是反序列化漏洞利用过程中的难点所在[^fn:2]。


### empty {#empty}

null[^fn:3]


## plantuml {#plantuml}

{{< figure src="/ox-hugo/plantuml-test.png" >}}


## fuji {#fuji}

{{< figure src="/ox-hugo/2021-01-26_15-49-33_screenshot.png" >}}

~~测试测试测试~~

test **test** _test_


## asciinema {#asciinema}

<script id="asciicast-325730" src="https://asciinema.org/a/325730.js" async></script>

[^fn:1]: <https://www.anquanke.com/post/id/86641>
[^fn:2]: test
[^fn:3]: null
