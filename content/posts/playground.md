+++
title = "测试页面"
publishDate = 2021-01-09T00:00:00+08:00
tags = ["test"]
draft = true
+++

<!--more-->


## Details and summary {#details-and-summary}

```python
if __name__ == '__main__':
    print('hello, hugo.')
```

<details>
<summary>
Why is this in **green**?
</summary>
<p class="details">

You will learn that later below in CSS section.
</p>
</details>


## java code block {#java-code-block}

<details>
<summary>
示例代码
</summary>
<p class="details">

```java
import javassist.*;
import java.lang.reflect.Constructor;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

public class demo {
    public static void main(String[] args) {
        ClassPool classPool = ClassPool.getDefault();

        //定义Someone类
        CtClass ccSomeone = classPool.makeClass("Someone");

        try {
            //定义成员变量name
            CtClass fieldType = classPool.get("java.lang.String");
            CtField cfName = new CtField(fieldType, "name", ccSomeone);
            cfName.setModifiers(Modifier.PRIVATE);  //用private修饰name
            ccSomeone.addField(cfName, CtField.Initializer.constant("init"));  //添加name到Someone中，初始值为init

            //定义构造方法
            CtClass[] parameters = new CtClass[]{classPool.get("java.lang.String")};  //参数为String类型
            CtConstructor constructor = new CtConstructor(parameters, ccSomeone);
            String body = "{this.name=$1;}";  //方法体，$1表示的第一个参数
            constructor.setBody(body);
            ccSomeone.addConstructor(constructor);  //设置为Someone的构造方法

            //setName和getName方法
            ccSomeone.addMethod(CtNewMethod.setter("setName",cfName));
            ccSomeone.addMethod(CtNewMethod.getter("getName",cfName));

            //toString 方法
            CtClass returnType = classPool.get("java.lang.String");  //返回类型为 String
            String methodName = "toString";
            CtMethod cmToString = new CtMethod(returnType, methodName, null, ccSomeone);
            cmToString.setModifiers(Modifier.PUBLIC);  //用 public 修饰
            String methodBody = "{return \"My name is \"+$0.name;}";  //方法体，$0 表示 this
            cmToString.setBody(methodBody);
            ccSomeone.addMethod(cmToString);

            //获取 Someone 类的实例，设置 name 属性，调用 toString 方法
            Class clazz = ccSomeone.toClass();
            Constructor cons = clazz.getConstructor(String.class);
            Object someone = cons.newInstance("zrquan");  //通过构造方法设置 name
            Method toString = clazz.getMethod("toString");
            System.out.println(toString.invoke(someone));
        } catch (NotFoundException | CannotCompileException | NoSuchMethodException e) {
            e.printStackTrace();
        } catch (IllegalAccessException e) {
            e.printStackTrace();
        } catch (InstantiationException e) {
            e.printStackTrace();
        } catch (InvocationTargetException e) {
            e.printStackTrace();
        }
    }
}
```
</p>
</details>


## foonote test {#foonote-test}


### 面向属性编程[^fn:1] {#面向属性编程}

当我们可以控制对象的属性值，并利用这些属性值影响代码的执行过程时，这一过程就称之为“面向属性编程（property-oriented programming，POP）”。POP 利用点指的是一个代码片段，我们可以修改某些对象的属性来影响这个代码片段，使其满足我们的特定需求。许多情况下，我们需要串联多个利用点才能形成完整的利用程序。我们可以将其看成高级 ROP（return-oriented
programming，面向返回编程）技术，只不过 ROP 需要将某个值推送到栈上，而我们通过 POP 利用点可以将某些数据写入到文件中。

这里非常重要的一点是，反序列化漏洞利用过程不需要将类或者代码发送给服务器来执行。我们只是简单发送了类中的某些属性，服务器对这些属性比较敏感， **通过这些属性，我们就能控制负责处理这些属性的已有代码。** 因此，漏洞利用成功与否取决于我们对代码的熟悉程度，我们需要充分了解通过反序列化操作能被操控的那部分代码。这也是反序列化漏洞利用过程中的难点所在[^fn:2]。

中文中文中文中文中文中文中文中文中文中文中文中文中文中文中文中文中文中文中文中文中文中文中文 english 中文中文中文中文中文中文中文中文中文中文中文中文中文中文中文中文中文中
english


### empty {#empty}

null[^fn:3]


## plantuml {#plantuml}

{{< figure src="/ox-hugo/plantuml-test.png" >}}


## fuji shortcodes {#fuji-shortcodes}

{{< figure src="/ox-hugo/2021-01-26_15-49-33_screenshot.png" >}}

[^fn:1]: <https://www.anquanke.com/post/id/86641>
[^fn:2]: test
[^fn:3]: null
