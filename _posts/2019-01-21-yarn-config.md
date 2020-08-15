---
title: yarn服务resourceManager进程内存溢出
tags:
  - Yarn
  - OOM
---

新搭建的yarn集群resourceManager进程几次发生内存泄漏问题, 调高内存只是延长内存溢出日期,并没有真正解决问题. <!--more-->

根据对resourceManager的功能分析,该程序仅有调度功能并且不实际执行任务,也没有涉及到数据的操作, 根据之前的认知该进程属于对内存大小要求不多的程序.昨天经过对resourceManager进程的dump文件进行分析,终于找到了内存泄漏的原因,以下是具体的分析过程.

## 分析步骤

想要分析内存, 首先要等程序运行一段时间, 等到gc日志中显示剩余内存不多时进行以下操作:

1. 获取dump文件

​    执行`jmap -dump:live,format=b,file=dump.log <pid>`,输入resourceManager的进程id,在当前目录会得到dump.log文件

  2.安装分析工具

​    在eclipse中安装Memory Analysis工具, 可以分析dump文件

  3.对dump进行分析

​    Memory Analysis插件可以简单的分析出哪个对象占用内存最多

![img](/assets/yarn-config-1.jpg)

可以看到`RMActiveServiceContext`对象占用了97%的内存,并且是被`ConcurrentHashMap`的一个实例占用的,点击最下面的Details按钮

查看`Shortest Paths To the Accumulation Point`一栏,可以看到具体的对象关系,前面的黑色字体为变量名称,后面为对象地址

![img](/assets/yarn-config-2.jpg)

 这个图是对象引用的最短路径,抛开jdk里面的对象,可以看到RMActiveServiceContext对象的applications变量占用了最多的内存,查看源码,得知这个对象是保存job运行状态的map,

 然后找到该对象添加和删除的地方

  添加:

![img](/assets/yarn-config-3.jpg)

 移除:

![img](/assets/yarn-config-4.jpg)

每次创建application对象,都会添加到applications map里面,另外有个地方会检测完成的application数量和maxCompletedAppsInMemory关系,

 如果超过限制才会移除application,查看源码找到maxCompletedAppsInMemory赋值的地方,是读取的配置文件中`yarn.resourcemanager.max-completed-applications`

 属性的值,程序中默认是保留10000个

![img](/assets/yarn-config-5.jpg)



周末在测试环境调小最大保留数后,使用测试脚本提交了10000个application进行压测,resourceManager的内存基本没有提高了 

增加配置项步骤:

   1.修改yarn-site.xml配置文件

​    由于业务日志都是druid系统内,yarn的日志只是运行状态的记录,并没有多少参考价值,所以可以把application最大保存数量调到500,增加配置项如下:

```xml
<property>
  <name>yarn.resourcemanager.max-completed-applications</name>
  <value>500</value>
</property>
```

  2.重启yarn集群

  注:resourceManager和nodeManager进程都属于调度程序,内存设置在512m~1g之间即可,不需要设置很大.