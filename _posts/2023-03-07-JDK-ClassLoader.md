---
layout: post
title: "JDK ClassLoader"
subtitle:   "类加载机制"
author:     "Walklown"
date: 2023-03-12 17:45:13 -0400
background: 'img/posts/06.jpg'
tags:
    - JDK
    - ClassLoader
    - 双亲委派
---

### 简介

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Java类加载器（Java Classloader）是Java运行时环境（Java Runtime Environment）的一个部件，负责动态加载Java类到Java虚拟机的内存空间中。类通常是按需加载，即第一次使用该类时才加载。由于有了类加载器，Java运行时系统不需要知道文件与文件系统。

### 双亲委派机制

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;指类加载器（Java Classloader）之间构成父子关系，加载类的时候总是会先询问父加载器是否已经加载过该类。如果父级加载器已经加载过，则不进行重复加载。  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<font color=red>首先，保证了java核心库的安全性。</font>如果你也写了一个java.lang.String类，那么JVM只会按照上面的顺序加载jdk自带的String类，而不是你写的String类。  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<font color=red>其次，还能保证同一个类不会被加载多次。</font>

### JDK不同版本的类加载器差异

#### 早于JDK 1.2-单纯的ClassLoader

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在JDK 1.2之前，JDK通过ClassLoader实现类加载机制。开发者可以依靠重写`loadClass`实现实现必要的功能。

#### JDK 1.2-“双亲委派机制”

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;JDK 1.2之后引入“双亲委派”方式来实现类加载器的层次调用，以尽可能保证JDK的系统API不会被Ext库的扩展类加载器破坏，系统API与Ext库的API也不会被用户定义的类加载器所破坏。  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;但一些使用场景会打破这个惯例来实现必要的功能。如：`java.lang.ClassLoader`类，新增`findClass`用于提供自定义接入点，开发者也仍可以重写`loadClass`实现以破坏“双亲委派机制”<font color=red></font>。

* BootClassLoader(Bootstrap ClassLoader) [%JAVA_HOME%/lib (rt.jar)]（C++编写，控制台打印不出）
* ExtClassLoader(Extension ClassLoader) [%JAVA_HOME%/jre/lib/ext]
    * ExtClassLoader.parent = BootClassLoader
* AppClassLoader(Application ClassLoader) [%CLASSPATH%]
    * AppClassLoader.parent = ExtClassLoader

#### JDK 1.6-违背“双亲委派机制”的SPI机制

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;JDK1.6引入SPI了机制。当父加载器加载接口类（如`java.sql.Driver`）时，会主动通过子加载器加载实现类，如com.mysql.cj.jdbc.Driver，如：`BootClassLoader`加载`java.sql.DriverManager`类时，会执行`loadInitialDrivers`来加载应用配置的`java.sql.Driver`实现类，此时由于com.mysql.cj.jdbc.Driver是应用类，不能使用当前上下文类加载器，所以使用了`ClassLoader.getSystemClassLoader()`获取类加载器（默认是AppClassLoader）。

#### JDK 9-模块化改造

https://openjdk.org/jeps/261

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;JDK9新增了模块化特性，包的目录结构发生了变化，不再使用系统库、Ext库结构。类加载器也因此进行了相应的调整，不再需要区分`BootstrapClassLoader`、`ExtClassLoader`。

![JDK ClassLoader.jpg](https://walklown.github.io/img/Images/JDK%20ClassLoader.jpg)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;JDK11中，BootClassLoader在库代码和虚拟机中都有实现，但为了兼容性，它在 ClassLoader API 中仍由 null 表示。 它定义了核心 Java SE 和 JDK 模块。  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;扩展类加载器（extension class loader）不再是 URLClassLoader 的实例，而是内部类的实例。 它不再通过扩展机制加载类，该机制已被 JEP 220 删除。但是，它确实定义了选定的 Java SE 和 JDK 模块，下面将详细介绍。 在其新角色中，此加载器称为Platform ClassLoader，可通过新的 ClassLoader::getPlatformClassLoader 方法使用，Java SE 平台 API 规范也需要它。  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;AppClassLoader不再是 URLClassLoader 的实例，而是内部类的实例。 它是既不是 Java SE 也不是 JDK 模块的命名模块的默认加载器。

> jdk.internal.loader.ClassLoaders:
>
> /**  
>  \* Bootstrap ClassLoader  
>  \**/  
> private static final BootClassLoader BOOT_LOADER;
>
>  /**  
>  \* Platform ClassLoader  
>  \**/  
> private static final PlatformClassLoader PLATFORM_LOADER;
>
>  /**  
>  \* Application ClassLoader  
>  \**/   
> private static final AppClassLoader APP_LOADER;

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;保留PlatformClassLoader不仅是为了兼容性，也是为了提高安全性。 <font color=red>BootClassLoader加载的类型被隐式授予所有安全权限 (AllPermission)，但其中许多类型实际上并不需要所有权限</font>。 我们通过将它们定义到PlatformClassLoader而不是BootClassLoader，并通过在默认安全策略文件中授予它们实际需要的权限，取消了不需要所有权限的模块的特权。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;提供工具或导出工具 API 的 JDK 模块被定义到AppClassLoader。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;三个内置的类加载器协同工作来加载类，如下所示：

AppClassLoader
1. 首先搜索为所有内置加载器定义的命名模块。   
   1.1. 如果在某个类加载器中有匹配的模块，则该加载器将加载该类。  
   1.2. 如果该加载器定义的命名模块中找不到类，则AppClassLoader将委托给其父级。  
   1.2.1. 如果其父类仍未找到该类，则AppClassLoader将搜索类路径。 在类路径中找到的类将作为此加载程序的未命名模块的成员进行加载。

PlatformClassLoader
1. 搜索为所有内置加载器定义的命名模块。
2. 如果为这些加载器之一定义了合适的模块，则该加载器将加载该类。 （因此，PlatformClassLoader现在可以委托给AppClassLoader，这在upgrade module路径上的模块依赖于application module路径上的模块时非常有用。）
3. 如果该加载器定义的命名模块中找不到类，然后PlatformClassLoader委托给它的父级。

BootClassLoader
1. 搜索为自己定义的命名模块。
2. 如果在引导加载器定义的命名模块中找不到类，则BootClassLoader通过 -Xbootclasspath/a 选项搜索添加到引导类路径的文件和目录。 在引导程序类路径中找到的类作为此加载程序的未命名模块的成员进行加载。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;AppClassLoader和PlatformClassLoader当在加载器定义的模块中找不到类时，会委托给它们各自的父加载器，以确保仍会搜索bootstrap类路径。


### [热部署（OSGi,  Open Services Gateway initiative）](https://www.osgi.org/)

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;OSGi 是 Eclipse 基金会下的一个开放规范和开源项目。 它是 OSGi 联盟（以前称为开放服务网关计划-`Open Services Gateway initiative`）所做工作的延续。  
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;OSGi 技术由一组规范、每个规范的实现以及每个规范的一组测试兼容性工具包组成，它们共同为 Java 定义了一个动态模块系统。其特点包括：
* 应用程序或组件以部署包的形式出现，可以远程安装、启动、停止、更新和卸载，而无需重新启动。
* 非常详细地指定了 Java 包/类的管理。 应用程序生命周期管理是通过支持远程下载管理策略的 API 实现的。
* 服务注册表使 bundle 能够检测新服务的添加或服务的删除，并相应地进行调整。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;OSGi 实现的类加载机制没有完全遵循自下而上的委托，有很多平级之间的类加载器查找。

