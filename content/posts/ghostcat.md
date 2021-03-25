+++
title = "Ghostcat漏洞分析"
publishDate = 2021-03-25T00:00:00+08:00
tags = ["cve"]
draft = false
+++

编号：CNVD-2020-10487/CVE-2020-1938

描述：由于 Tomact AJP 协议的设计缺陷，攻击者可以通过漏洞读取 webapp 目录下的任意文件，或者通过文件包含 getshell。

影响版本：Tomcat 9/8/7/6

<!--more-->


## Tomcat Connector {#tomcat-connector}

Connector 是 Tomcat 用来处理外部连接的核心组件，它将底层的 socket 信道封装成
Request 和 Response 两个对象，再交给 Container(servlet 容器)进行处理。

Connector 的源码在 `org.apache.coyote` 目录下，默认有两个——分别处理 HTTP 协议和 AJP
协议。

{{< figure src="/ox-hugo/tomcat.png" >}}


## Apache Jserv Protocol {#apache-jserv-protocol}

AJP 协议可以简单理解为 HTTP 协议的二进制性能优化版本，它能降低 HTTP 请求的处理成本，因此主要在需要集群、反向代理的场景被使用，默认开放在 8009 端口。

浏览器无法直接处理 AJP 协议，所以通常用于 Tomcat 和其他 web 服务器的通信，比如
Apache 的 proxy\_ajp 模块就可以代理 AJP 协议。

{{< figure src="/ox-hugo/ajp.png" >}}


## POC {#poc}

部署好环境后，在 github 找个[脚本](https://github.com/YDHCUI/CNVD-2020-10487-Tomcat-Ajp-lfi)复现一下漏洞。

{{< figure src="/ox-hugo/2021-03-23_22-53-01_screenshot.png" >}}

可见通过 AJP 协议请求 `/asdf` 路径，却读取了 web.xml 文件的内容。

用 wireshark 看一下 AJP 请求的内容：

{{< figure src="/ox-hugo/2021-03-23_22-54-13_Snipaste_2021-03-23_18-36-54.png" >}}

红框中的三个由用户控制的参数就是导致文件读取的罪魁祸首，下面通过调试 Tomcat 看看怎么处理它们的。


## 文件读取 {#文件读取}

处理请求的类是 `org.apache.coyote.ajp.AbstractAjpProcessor` ，进入 prepareRequest
方法，这里是漏洞的入口点，此时的调用栈如下：

{{< figure src="/ox-hugo/2021-03-23_21-47-36_screenshot.png" >}}

来到关键的 while 语句，在这里对上面提到的三个参数进行设置：

{{< figure src="/ox-hugo/2021-03-23_23-01-39_screenshot.png" >}}

`request.setAttribute()` 就是简单的 HashMap 赋值操作，循环结束后 attributes 属性如下：

{{< figure src="/ox-hugo/2021-03-23_23-04-45_screenshot.png" >}}

AJP Connector 封装好 request 对象后，会将其传给 Container 处理。

{{< figure src="/ox-hugo/2021-03-24_14-13-34_screenshot.png" >}}

Container 即 servlet 容器，相关设置可以在 web.xml 文件找到，下面是默认开启的两个
servlet 容器：

```xml
<servlet>
    <servlet-name>default</servlet-name>
    <servlet-class>org.apache.catalina.servlets.DefaultServlet</servlet-class>
    ...
</servlet>

<servlet>
    <servlet-name>jsp</servlet-name>
    <servlet-class>org.apache.jasper.servlet.JspServlet</servlet-class>
    ...
</servlet>

<!-- The mapping for the default servlet -->
<servlet-mapping>
    <servlet-name>default</servlet-name>
    <url-pattern>/</url-pattern>
</servlet-mapping>

<!-- The mappings for the JSP servlet -->
<servlet-mapping>
    <servlet-name>jsp</servlet-name>
    <url-pattern>*.jsp</url-pattern>
    <url-pattern>*.jspx</url-pattern>
</servlet-mapping>
```

当请求目标的后缀是 jsp 或者 jspx 时由 JspServlet 处理请求，其他情况则由 DefaultServlet
处理。因此为了确保由 DefaultServlet 处理我们的请求，而使用 `/asdf` 这种大概率不存在的路径。

代码执行到 DefaultServlet#doGet 方法中：

{{< figure src="/ox-hugo/2021-03-24_14-46-56_screenshot.png" >}}

一直跟进到 getRelativePath 方法，关键代码如下：

{{< figure src="/ox-hugo/2021-03-24_14-51-56_screenshot.png" >}}

上图的第一个 if 语句拼接了 path\_info 和 servlet\_path 参数得到 `/WEB-INF/web.xml` ，而进入 if 的条件是 request\_uri 不为空(不一定要是 `/`)，这就是 poc 设置该参数的原因。

获取路径后调用 getResource 方法读取文件：

{{< figure src="/ox-hugo/2021-03-24_16-01-27_screenshot.png" >}}

{{< figure src="/ox-hugo/2021-03-24_16-02-07_screenshot.png" >}}

红框中的 validate 方法用来检查文件的路径，确保路径以 `/` 开头，且不会跨越到 webapp
目录以外。代码如下：

{{< figure src="/ox-hugo/2021-03-24_16-09-10_screenshot.png" >}}

从上图可以看到代码会校验路径是否以 `/` 开头，且标准化了路径分隔符，但是怎么防止目录穿越呢？在处理路径分隔符的部分调用了 RequestUtil#normalize 方法，我们跟进一下：

{{< figure src="/ox-hugo/2021-03-24_16-40-44_screenshot.png" >}}

方法开头会替换 windows 系统的 `\\` ，然后根据情况在开头和末尾加上 `/` ，在后面执行以下
while 语句：

```java
while (true) {
    int index = normalized.indexOf("/../");
    if (index < 0) {
        break;
    }
    if (index == 0) {
        return null;  // Trying to go outside our context
    }
    int index2 = normalized.lastIndexOf('/', index - 1);
    normalized = normalized.substring(0, index2) + normalized.substring(index + 3);
}
```

上述代码会将 `/../` 和前一个目录名抵消，如果最后 `/../` 出现在开头，即尝试逃出
webapp 目录，就会返回 null。理解这段代码的关键在于理解 lastIndexOf 方法，它返回第一个参数字符串最后出现的位置，第二个参数是表示位置的整数，可以理解为”只搜索该位置前的内容“。

```java
"aaabbbaaa".lastIndexOf("aaa");     // 6
"aaabbbaaa".lastIndexOf("aaa", 4);  // 0
```

校验完路径后，返回 getResource 方法读取文件资源。

{{< figure src="/ox-hugo/2021-03-24_16-59-36_screenshot.png" >}}


## 文件包含 {#文件包含}

前面说过 Tomcat 默认的 servlet 容器有两个，当访问的是 jsp 文件时会由 JspServlet
来处理请求。这个处理实际就是解析 jsp 文件，但前提是这个 jsp 文件是存在的，所以首先要想办法上传到 webapp 目录下。

我们先在 webapp 目录下创建一个 test.jsp：

```jsp
<%Runtime.getRuntime().exec("calc.exe");%>
```

修改一下 poc，发送的 AJP 请求如下：

{{< figure src="/ox-hugo/2021-03-24_18-13-11_screenshot.png" >}}

AJP Connector 的处理和之前类似，我们直接跟进到 JspServlet 类中。这次处理请求的方法是 service，关键代码如下：

{{< figure src="/ox-hugo/2021-03-24_18-20-01_screenshot.png" >}}

这里拼接 path\_info 和 servlet\_path 参数得到文件路径，接着调用 serviceJspFile 方法解析目标 jsp 文件。

{{< figure src="/ox-hugo/2021-03-24_18-23-41_screenshot.png" >}}

所以找到目标文件的关键在于 path\_info 和 servlet\_path 参数，如果上传的 jsp 文件在 upload
目录下，servlet\_path 就要设置为 `/upload/` ，而 URL 的路径并不重要，只要后缀是 jsp 或者
jspx 即可。

再跟进一下 serviceJspFile 方法：

{{< figure src="/ox-hugo/2021-03-24_18-30-29_screenshot.png" >}}

在 JspServletWrapper#service 方法中编译好目标 jsp 文件，获取对应的 servlet 对象，再执行它的 service 方法弹出计算器。

{{< figure src="/ox-hugo/2021-03-24_18-36-19_screenshot.png" >}}


## 总结 {#总结}

导致文件读取漏洞的是 AJP 请求中的三个用户可控参数：

1.  javax.servlet.include.request\_uri：不为空即可
2.  javax.servlet.include.path\_info：文件名
3.  javax.servlet.include.servlet\_path：根路径

而当请求资源是 jsp 或 jspx 文件时，由于 JspServlet 类会编译执行目标文件，导致文件包含漏洞，不过前提是能把恶意文件上传到 webapp 目录下。
