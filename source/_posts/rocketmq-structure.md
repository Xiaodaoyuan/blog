---
title: Rocketmq源码解析-1.Rocketmq介绍
date: 2019-12-18 20:16:13
categories: middleware
tags: rocketmq
---

### Rocketmq优势：
1. rocketmq原生就支持分布式，而activemq原生存在单点性。
2. rocketmq可以严格的保证消息的顺序，而activemq不能保证。
3. rocketmq可以提供亿级消息的堆积能力，不过这不是重点，重点是堆积了亿级消息还能保持低延迟写入。
 <!--more-->
4. 丰富的消息推拉模式（push和pull）：push好理解，在消费者端设置消息listener回调；pull需要应用主动的调用拉消息的方法从broker拉取消息，这里存在一个记录消费位置的问题，如果不记录会存在重复消费的问题。
5. 一般mq分布式协调使用zookeeper，rocketmq自己实现了一个nameserver，更加轻量级，性能更好。
6. 消息失败重试机制、高效的订阅者水平扩展能力、强大的api、事务机制等等。
7. rocketmq有group的概念，通过group机制可以实现天然的消息负载均衡。

### Rocketmq部署模式：
1. 单master模式：无需多言，一旦单个broker重启或宕机，一切都结束了！显然，线上不可以使用。
2. 多master模式：全是master，没有slave。一个broker宕机了，应用没有影响，缺点在于宕机的master上未被消费的消息在master没有恢复之前不可以订阅。
3. 多master多slave模式（异步复制）：高可用，采用异步复制的方式，主备之间短暂延迟，ms级别。master宕机，消费者可以从slave上消费，但是master的宕机会导致丢失掉极少量的消息。
4. 多master多slave模式（同步双写）：和上面的区别在于采用的是同步方式，也就是master、slave都写成功才会向应用返回成功。可见不论是数据，还是服务都没有单点，都非常可靠！缺点在于同步的性能比异步稍低。

### Rocketmq项目结构：
![image](/images/rocketmq-project-structure.png)

* acl：访问权限控制模块
* broker：消息代理模块，串联 Producer/Consumer 和 Store 
* client：客户端模块，包含producer和consumer，负责消息的发送和消费
* common：公共模块，供其他模块使用
* distribution：包含一些 sh 脚本和 配置，主要供部署时使用
* example：一些rocketmq使用的示例代码
* filter：过滤器模块
* logappender：接入日志需要的appender模块
* logging：日志模块
* namesrv：rocketmq的注册中心，broker，produce，consumer以及topic信息会在namesrv中注册
* openmessaging：忽略，没了解过
* remoting：远程调用模块，基于Netty实现，produer，consumer，broker之间通过这个模块通信。
* srvutil：解析命令行的工具类ServerUtil
* store：消息存储模块
* test：测试用例模块
* tools：一些工具类，基于它们可以写一些 sh 工具来管理、查看MQ系统的一些信息

对于这些模块我们不需要都去研究源码，只需要挑几个重点去关注。这里面比较重要的是broker，client，common，namesrv，remoting，store这几个模块。

### Rocketmq逻辑部署架构
![image](/images/rocketmq-deploy.png)

这是 Rocketmq 的逻辑部署结构(参考《RocketMQ原理简介 v3.1.1》)，包括 producer/broker/namesrv/consumer 四大部分。namesrv 起到注册中心的作用，部署的时候会用到 rocketmq-namesrv/rocketmq-common/rocketmq-remoting 三个模块的代码；broker 部署的时候会用到 rocketmq-broker/rocketmq-store/rocketmq-common/rocketmq-remoting 四个模块的代码；producer 和 consumer 会用到 rocketmq-client/rocketmq-common/rocketmq-remoting 三个模块的代码，这里虽然将它们分开画了，但实际上一个应用往往既是producer又是consumer。