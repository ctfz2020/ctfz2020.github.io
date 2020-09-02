---
title:  自定义类加载器引发的bug
tags:
  - 类加载器
  - Hadoop
---

最近遇到一个诡异的问题, hadoop服务加载了native文件, 客户端也指定了`java.library.path`属性,但是程序还是报未加载native包异常

<!--more-->

### 现象如下

在服务器上输入`hadoop checknative`命令, 显示可以加载native包

```shell
[~]$ hadoop checknative
20/09/02 11:04:43 WARN bzip2.Bzip2Factory: Failed to load/initialize native-bzip2 library system-native, will use pure-Java version
20/09/02 11:04:43 INFO zlib.ZlibFactory: Successfully loaded & initialized native-zlib library
Native library checking:
hadoop:  true /opt/hadoop/lib/native/libhadoop.so
zlib:    true /lib64/libz.so.1
snappy:  true /opt/hadoop/lib/native/libsnappy.so.1
lz4:     true revision:10301
bzip2:   false 
openssl: true /usr/lib64/libcrypto.so
```

程序启动参数

```shell
-Djava.library.path=/opt/hadoop/lib/native
```

程序日志

```log
2020-09-02T10:45:39 WARN [xxx] org.apache.hadoop.util.NativeCodeLoader - Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
```

### 分析

首先服务器可以加载native包,说明so文件没问题,

排除了版本问题, 因为服务器和客户端都是2.6.0版本.

然后程序中打印了`java.library.path`参数,是正确的.

没有其它思路, 只能从hadoop的源码入手了, 找到打印日志的源码, 可以看到加载代码在静态代码块中.

```java
public class NativeCodeLoader {
    static {
    // Try to load native hadoop library and set fallback flag appropriately
    if(LOG.isDebugEnabled()) {
      LOG.debug("Trying to load the custom-built native-hadoop library...");
    }
    try {
      System.loadLibrary("hadoop");
      LOG.debug("Loaded the native-hadoop library");
      nativeCodeLoaded = true;
    } catch (Throwable t) {
      // Ignore failure to load
      if(LOG.isDebugEnabled()) {
        LOG.debug("Failed to load native-hadoop with error: " + t);
        LOG.debug("java.library.path=" +
            System.getProperty("java.library.path"));
      }
    }
    
    if (!nativeCodeLoaded) {
      LOG.warn("Unable to load native-hadoop library for your platform... " +
               "using builtin-java classes where applicable");
    }
  }
  ...
}
```

静态代码块做了一件事情, 就是调用`System.loadLibrary()`方法加载hadoop本地方法库, 如果执行失败会打印警告日志, 加载过程出现异常会打印debug级别的日志, 所以可以通过日志判断执行情况

### 调试

打开日志的debug级别, 再次运行程序, 日志情况如下

首先打印了加载日志

```log
2020-09-02T11:20:24 DEBUG [xxx] org.apache.hadoop.util.NativeCodeLoader - Trying to load the custom-built native-hadoop library...
2020-09-02T11:20:24 DEBUG [xxx] org.apache.hadoop.util.NativeCodeLoader - Loaded the native-hadoop library
```

然后过了一段时间又打印了几条

```log
2020-09-02T11:20:28 DEBUG [xxx] org.apache.hadoop.util.NativeCodeLoader - Trying to load the custom-built native-hadoop library...
2020-09-02T11:20:28 DEBUG [xxx] org.apache.hadoop.util.NativeCodeLoader - Failed to load native-hadoop with error: java.lang.UnsatisfiedLinkError: Native Library /data/br/base/druid/hadoop-dependencies/native/libhadoop.so already loaded in another classloader
2020-09-02T11:20:28 DEBUG [xxx] org.apache.hadoop.util.NativeCodeLoader - java.library.path=/opt/hadoop/lib/native
2020-09-02T11:20:28 WARN [xxx] org.apache.hadoop.util.NativeCodeLoader - Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
```

### 分析

通过日志可以得知静态代码块执行了两次, 那么产生了两个问题

1. 为什么静态代码块会执行两次
2. 为什么第二次执行失败

静态代码块的问题涉及到类加载机制了, 所以我重温了这部分知识点

#### 类加载机制

java虚拟机通过类加载器把编译后的class文件加载到内存, 在虚拟机中类加载器+类本身确定唯一性, 也就是说当类被不同的类加载器加载时,可能会出现多次初始化.

为什么说可能呢? 因为java虚拟机为了避免类加载器滥用, 出现各种奇怪的现象, 推荐类加载器使用双亲委派模型

> 双亲委派模型: 当一个类加载器接收到类加载请求时，会先请求其父类加载器加载，依次递归，当父类加载器无法找到该类时（根据类的全限定名称），子类加载器才会尝试去加载。

![classloader-02](/assets/classloader-02.png)

启动类加载器：在HotSpot VM中这个加载器是由c++实现，负责将存放在`JAVA_HOME`下lib目录中的类库，比如rt.jar。因此，启动类加载器不属于Java类库，无法被Java程序直接引用，用户在编写自定义类加载器时，如果需要把加载请求委派给引导类加载器，那直接使用null代替即可。

扩展类加载器：由sun.misc.Launcher$ExtClassLoader实现，负责加载`JAVA_HOME`下lib\ext目录下的，或者被`java.ext.dirs`系统变量所指定的路径中的所有类库，开发者可以直接使用扩展类加载器。

应用类加载器：由sun.misc.Launcher$AppClassLoader实现的。由于这个类加载器是ClassLoader中的getSystemClassLoader方法的返回值，所以也叫系统类加载器。它负责加载`java.class.path`上所指定的类库，可以被直接使用。如果未自定义类加载器，默认为该类加载器。

当自定义类加载器满足双亲委派模型时, 是不会出现类被初始化多次的现象, 如果不遵守就会就会出现多次.

举个例子

```java
package io.github.ctfz2020.classloader;
import java.io.ByteArrayOutputStream;
import java.io.File;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.MalformedURLException;
import java.net.URL;
import java.net.URLClassLoader;
import java.util.Arrays;
import java.util.List;
import java.util.stream.Collectors;

public class ClassLoaderTest {
	static {
		System.out.println("init static code");
	}
	
	public static void main(String[] args) throws MalformedURLException, ClassNotFoundException {
		String className = ClassLoaderTest.class.getName();
		
		System.out.println("------------- Use CustomClassLoader ---");
		ClassLoader customClassLoader = new CustomClassLoader();
		//加载类
		customClassLoader.loadClass(className);
		//初始化
		Class.forName(className, true, customClassLoader);
		
		System.out.println("------------- Use URLClassLoader ------");
		String classpath = System.getProperty("java.class.path");
		//将classpath路径转换为文件形式
		List<File> jars = Arrays
                         .asList(classpath.split(";"))
                         .stream()
                         .map(cp -> new File(cp))
                         .collect(Collectors.toList());
		//将文件转换为URL
		URL[] urls = jars
                     .stream()
                     .map(ClassLoaderTest::fileToURL)
                     .collect(Collectors.toList())
                     .toArray(new URL[0]);
		
		//创建url classloader, 指定父classloader为null
		ClassLoader urlClassLoader = new URLClassLoader(urls, null);
		//初始化
		urlClassLoader.loadClass(className);
		Class.forName(className, true, urlClassLoader);
	}
	
	public static class CustomClassLoader extends ClassLoader{
		@Override
		public Class<?> loadClass(String name) throws ClassNotFoundException {
			InputStream inputStream = getClass().getResourceAsStream(name.substring(name.lastIndexOf('.') + 1) + ".class");
			if(inputStream == null) {
				return super.loadClass(name);
			}
			ByteArrayOutputStream output = new ByteArrayOutputStream();
			copy(inputStream, output);
			byte[] bytes = output.toByteArray();
			
			return defineClass(name, bytes, 0, bytes.length);
		}
		
		public void copy(InputStream input, OutputStream output) {
			try {
				int len = 0;
				byte[] tmp = new byte[128];
				while((len = input.read(tmp)) != -1) {
					output.write(tmp, 0, len);
				}
			} catch (IOException e) {
				e.printStackTrace();
			}
		}
	}
	
	public static URL fileToURL(File file) {
		try {
			return file.toURI().toURL();
		} catch (MalformedURLException e) {
			return null;//不推荐这样写
		}
	}
}
```

运行结果

```shell
init static code
-------------Use CustomClassLoader---
init static code
-------------Use URLClassLoader------
init static code
```

可以看到`ClassLoaderTest`类中的静态代码块执行了3次, 第一次是默认的虚拟机加载用户指定classpath的AppClassLoader, 第二次是通过重写loadClass破坏双亲委派模型的CustomClassLoader, 第三次是没有指定父加载器的URLClassLoader.

什么情况下会这样使用呢?

当程序中由于某种原因引用了某个类库的多个版本时, 为了防止不同版本直接出现冲突, 需要使用自定义不同的classloader去加载不同的版本, 此时就不需要实现双亲委派模型, 本程序就是出于这个目的编写的自定义类加载器.

#### System.loadLibrary方法

当程序使用JNI技术时, 需要通过`System.loadLibrary`方法加载本地类库, 这个方法最终调用了`ClassLoader`的静态方法`loadLibrary0()`,`ClassLoader`类在`java.lang`包下面,是由启动类加载器(Bootstrap ClassLoader)加载, 其它加载器无法加载的, 所以`ClassLoader`类在虚拟机中只有一个

```java
private static Vector<String> loadedLibraryNames = new Vector<>(); 

private static boolean loadLibrary0(Class<?> fromClass, final File file) {
    ...
    synchronized (loadedLibraryNames) {
        if (loadedLibraryNames.contains(name)) {
            throw new UnsatisfiedLinkError
                ("Native Library " + name +
                 " already loaded in another classloader");
        }
     ...
   }
 ...
}
```

可以看到方法中使用集合`loadedLibraryNames`判断是否已经存在相同的本地库名称, 由于类唯一并且是静态集合对象, 所以无论哪个类加载器加载的类调用`System.loadLibrary`都会执行这一步, 多次加载一定会抛异常.

### 结论

JNI类库只能加载一次, 静态代码块不一定只执行一次, 想要通过静态代码块实现本地类库加载不是一个好的处理方式, 很有可能引发bug.

