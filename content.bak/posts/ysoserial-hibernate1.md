+++
title = "Ysoserial-Hibernate1"
publishDate = 2021-01-26
tags = ["java", "反序列化"]
draft = false
+++

Hibernate 是 Java 的一个对象关系映射(ORM)框架，使用 GNU 开源协议。该利用链
以 HashMap 为入口点，通过 hibernate-core 包中的多个类及其方法构成调用路径，最终执行事
先写入到 TemplatesImpl 类中的命令。

<!--more-->


## 动态字节码 {#动态字节码}

ysoserial 在构造 payload 时，利用了 Java 的动态字节码生成的技术，所以我们先稍微
了解一下。

众所周知，Java 是一门需要编译的语言，JVM 读取并运行的是编译后的 `.class` 字节码文
件。但有时候我们想要在程序运行中动态修改一些代码，比如热补丁、接口升级、IDE 在调
试时读取或修改变量等等需求。如果每次都要把程序停掉再重新编译，显然不太现实，而
通过 Java 动态字节码技术，就可以直接修改字节码文件，再通过相关接口加载到 JVM 中，
实现运行时的代码修改。

常见的动态字节码修改有两种方式：

1.  ASM，可以直接操作字节码，执行效率高，相对的门槛也比较高，需要对 Java 的字节码
    文件有所了解，熟悉 JVM 的编译指令。
2.  Javassit，由东京技术学院的 Shigeru Chiba 创作的开源类库，提供了更抽象的 API 来对
    字节码进行操作。Hibernate1 利用链中也正是使用这个类库来构造 payload。


### Javassit demo {#javassit-demo}

下面的示例代码通过 Javassit 构造了一个 Someone 类，并添加了方法和属性。然后将该类实
例化，并调用其 toString 方法。
{{< code language="java" title="示例代码" expand="➕" collapse="➖" isCollapsed="true" >}}

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

{{< /code >}}

{{< figure src="/ox-hugo/2021-01-26_19-11-04_screenshot.png" >}}


## 构造 payload {#构造-payload}

我们先看一下最主要的 getObject 方法，这个方法完成了对要执行的命令的“包装”，并返回
包含该命令的对象。

```java
public Object getObject ( String command ) throws Exception {
    Object tpl = Gadgets.createTemplatesImpl(command);
    Object getters = makeGetter(tpl.getClass(), "getOutputProperties");
    return makeCaller(tpl, getters);
}
```

其中 makeGetter 和 makeCaller 都是内部函数，而 makeGetter 其实只起到路由作用，真
正的逻辑代码在内部函数 makeHibernate4Getter 和 makeHibernate5Getter 中。

{{< figure src="/ox-hugo/2021-01-22_16-29-20_screenshot.png" >}}

首先跟进一下 createTemplatesImpl 方法：

{{< figure src="/ox-hugo/2021-01-22_16-53-53_screenshot.png" >}}

该方法先对系统属性 properXalan 进行了判断——如果返回 false，则通过反射机制获取
TemplatesImpl、AbstractTranslet、TransformerFactoryImpl 三个类的全局限定类名，将
其作为参数，调用重载的 createTemplatesImpl 方法；如果返回 true，就使用上述三个类
的非全局限定类名作为参数调用重载方法。

在重载的 createTemplatesImpl 方法中，通过 Javassit 库创建一个 StubTransletPayload 类，
并将我们要执行的命令添加到这个类的静态代码块，当这个类被加载时，我们写入的命令就
会执行。

{{< figure src="/ox-hugo/2021-01-22_17-44-52_screenshot.png" >}}

可以看到在 116 行，调用 `CtClass.makeClassInitializer()` 创建一个空的静态代码块，
再通过 `insertAfter()` 将命令插入到静态代码块末尾。

代码的 119 和 120 行给 StubTransletPayload 设置了父类 AbstractTranslet，但其实没
有必要，因为 StubTransletPayload 本身就继承了 AbstractTranslet。

{{< figure src="/ox-hugo/2021-01-22_17-55-15_screenshot.png" >}}

方法的最后将 StubTransletPayload 实例转换成 byte 数组，赋给 templates 变量
的 `_bytecodes` 属性，并设置了 `_name` 和 `_tfactory` 属性。

返回的 templates 变量是 TemplatesImpl 类的实例，执行完 createTemplatesImpl 后，它的结
构大致如下图所示：

{{< figure src="/ox-hugo/templates.png" >}}

接下来执行的代码是 makeGetter 方法：

```java
Object getters = makeGetter(tpl.getClass(), "getOutputProperties");
```

首先通过系统属性判断当前的 hibernate 版本，ysoserial 使用的是 4.3 版本，所以直接跳
到 makeHibernate4Getter 函数。

{{< figure src="/ox-hugo/2021-01-24_22-06-00_screenshot.png" >}}

这一部分比较简单，通过反射机制创建了一个 `BasicPropertyAccessor$BasicGetter` 实例，
并赋值了 clazz、method、propertyName 属性，然后放到 Getter 数组中。类图大致如下：

{{< figure src="/ox-hugo/getter.png" >}}

最后执行的是 `makeCaller(tpl, getters)` ，通过反射机制进行一系列生成实例、赋值属性
的操作，然后返回一个 HashMap 对象。

{{< figure src="/ox-hugo/2021-01-25_13-50-41_screenshot.png" >}}

Reflections 类封装了一些反射机制的操作，注意到生成 PojoComponentTuplizer 实例时调用
的是 createWithoutConstructor 方法，实际上会使用 Object 类的构造方法，所以实例中的属
性基本都是 null。接着将之前构造的 `BasicPropertyAccessor$BasicGetter` 赋值给 getters 属性。

{{< figure src="/ox-hugo/2021-01-25_13-57-05_screenshot.png" >}}

生成 ComponentType 实例时，使用的是其父类 AbstractType 的默认构造方法，然后赋值
了 componentTuplizer、propertySpan、propertyTypes 三个属性。

{{< figure src="/ox-hugo/2021-01-25_14-03-05_screenshot.png" >}}

`Gadgets.makeMap(v1, v2)` 将两个相同的 TypedValue 实例写入到 HashMap 中，payload 的构造
到此就完成了，当目标应用反序列化这个 HashMap 对象时，我们写入的命令就会执行。

HashMap 的类图大致如下：

{{< figure src="/ox-hugo/hashmap.png" >}}


## 反序列化过程 {#反序列化过程}

使用 jdk 反序列化 HashMap 对象，自然会调用其 readObject 方法，所以以该方法作为入口点，
调试分析 HashMap 对象反序列化的过程。

如下图，在 readObject 方法中，局部变量 mappings 的值为 size 属性的值 2：

{{< figure src="/ox-hugo/2021-01-26_15-46-03_screenshot.png" >}}

一直执行到方法最后的 for 循环，从注释可以知道在这个循环中取出所有 key 和 value，并保
存到 HashMap 对象中。这里用到了上面的 mappings 变量，如果 mappings 变量是 0，就无法进入
循环了。

{{< figure src="/ox-hugo/2021-01-26_15-49-33_screenshot.png" >}}

跟进最后一行的 `hash(key)` ，这里的 key 是 TypedValue 类的对象。

{{< figure src="/ox-hugo/2021-01-26_16-08-16_screenshot.png" >}}

调用 `TypedValue.hashCode` 方法：

```java
public int hashCode() {
    return (Integer)this.hashcode.getValue();
}
```

其中 `this.hashcode` 属性是一个匿名内部类：

```java
private void initTransients() {
    this.hashcode = new ValueHolder(new DeferredInitializer<Integer>() {
        public Integer initialize() {
            return TypedValue.this.value == null ? 0 : TypedValue.this.type.getHashCode(TypedValue.this.value);
        }
    });
}
```

可以看到匿名内部类中有一个 initialize 方法，这个方法会在 getValue 方法中被调用。从上
面的类图可以很清楚地看到，TypedValue 的 value 属性和 type 属性分别是 TemplatesImpl 对象
和 ComponentType 对象。

继续跟进代码的执行过程：

{{< figure src="/ox-hugo/2021-01-26_16-47-43_screenshot.png" >}}

{{< figure src="/ox-hugo/2021-01-26_16-49-14_screenshot.png" >}}

执行到 PojoComponentTuplizer 的 getPropertyValue 方法，并获取我们之前构造
的 `BasicPropertyAccessor$BasicGetter` 对象，执行其 get 方法，参数是我们传入
的 TemplatesImpl 对象。

{{< figure src="/ox-hugo/2021-01-26_16-52-20_screenshot.png" >}}

{{< figure src="/ox-hugo/2021-01-26_16-56-32_screenshot.png" >}}

通过反射机制调用 `TemplatesImpl.getOutputProperties` 方法：

{{< figure src="/ox-hugo/2021-01-26_17-00-48_screenshot.png" >}}

{{< figure src="/ox-hugo/2021-01-26_17-01-13_screenshot.png" >}}

跟进 newTransformer 方法：

{{< figure src="/ox-hugo/2021-01-26_17-03-17_screenshot.png" >}}

然后进入我们的最终的目标方法 getTransletInstance：

{{< figure src="/ox-hugo/2021-01-26_17-09-42_screenshot.png" >}}

之所以在开始设置 TemplatesImpl 的 `_name` 属性，就是为了通过这里的 if 判断。

此时 `_class` 的值是 null，调用 defineTransletClasses 方法将 `_bytecodes` 中的每
个 `byte[]` 数组还原成一个 Class 对象，写到 `_class` 中。

{{< figure src="/ox-hugo/2021-01-26_17-17-14_screenshot.png" >}}

可见此时的 `_class[0]` 是我们通过 Javassit 库构造的 StubTransletPayload 类，它的静态
代码块保存着我们要执行的命令。

紧接着实例化了这个类，成功执行命令。

{{< figure src="/ox-hugo/2021-01-26_17-22-19_screenshot.png" >}}

这里还留意到 `_class[1]` 也是我们构造的 Foo 类，然而这个类似乎对过程没什么影响，不太
明白构造这个类的用意🤔

{{< figure src="/ox-hugo/2021-01-26_17-25-06_screenshot.png" >}}
