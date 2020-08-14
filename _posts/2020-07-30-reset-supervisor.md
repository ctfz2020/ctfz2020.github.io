---
title: 重置数据源入库进度
tags:
  - Java
  - Druid
  - Kafka
---

对于实时流式数据入库, druid是借助kafka来完成的. 待入库的数据首先是发送到kafka, druid从kafka消费数据, 进入到druid中.

kafka数据模型是队列模式, 数据发送给kafka是按照发送顺序存储, 消费时按照顺序消费. 这里就会存在一个问题,当消费者重置消费进度时, 有两种选择:

<!--more-->

1. 重置到队列头部

2. 重置到队列尾部

   ![image-20200729104952893](/assets/reset-supervisor-1.png)

## 什么时候需要重置消费进度

#### 更换TOPIC

durid入库是以Supervisor为单位, 每个Supervisor会消费一个kafka topic, 并且保存了该topic的消费进度.

当Supervisor消费的topic进行变更, 此时消费进度会失效, 入库流程会停止, 需要重置进度以恢复入库流程. 这种情况一般是要重置到kakfa队列头,防止新topic上丢失部分数据.

#### kafka最近的数据出现丢失

kafka最近的数据会存储在内存中, 当生产者的ack设置为1时, kafka副本异步同步, 不可避免会出现最近数据丢失的情况,此时生产者进度会变为之前的偏移量

![image-20200729110727362](/assets/reset-supervisor-2.png)

如果消费者在数据丢失前已经消费到部分数据, 当数据出现丢失, druid Supervisor会发现消费的进度超过了kafka生产者的进度, 就会终止入库流程, 此时需要重置消费进度到kafka持有的偏移量范围内.

#### 数据积压过多

某些情况下, 消费者会积压过多的数据, 比如:

* 入库流程停止一段时间后才发现; 
* 压测时生产者速度大幅提升, 等压测结束消费者一直没有跟上生产进度

这些情况想丢掉积压的数据, 直接入库最近的数据, 需要重置消费进度到kafka队列的尾部.

查看数据积压条数的步骤为

登录overlord页面, url为 http://overlord节点ip:8090, 点击Ingestion标签,搜索目标数据,然后点击右侧

![image-20200729114752819](/assets/reset-supervisor-3.png)

Supervisors模块支持搜索数据源名称, 在右侧点击放大镜按钮

![image-20200729115003762](/assets/reset-supervisor-4.png)

弹框中点击`status`, `aggregateLag`参数表示总积压量, `minimumLag`参数表示kafka各分区积压量.

> Supervisor分配的task处于RUNNING状态时, 才会显示对应的分区积压.

## 怎样重置消费进度

### 设置useEarliestOffset属性

根据需求设置重置偏移量为队头还是队尾属性, 如果是队头需要设置useEarliestOffset属性为true, 如果是队尾则需要设置为false, 默认为false.

### 重置操作

在overlord页面Ingestion标签里面搜索目标数据源,然后点击右侧的小扳手

![image-20200814191630791](/assets/reset-supervisor-5.png)

在弹窗中点击reset按钮

![image-20200814191946996](/assets/reset-supervisor-6.png)