---
title: Rocketmq源码解析-2.Namesrv
date: 2019-12-19 20:10:25
categories: middleware
tags: rocketmq
---

### Namesrv简介
Namesrv可以理解为一个注册中心，类似于kafka的zookeeper，但是比zk更加轻量，主要包含两块功能：
	
* 管理一些KV配置信息。
* 管理broker和topic的注册信息。  

<!--more-->

![image](/images/rocketmq-namesrv.png)

---------------------
### Namesrv启动过程
> 启动过程主要涉及NamesrvStartup和NamesrvController两个类。

> 执行sh mqnamesrv命令会启动NamesrvStartup类中的main方法，首先会执行createNamesrvController方法，解析命令行中的参数到各种config对象中（主要是NettyServerConfig和NamesrvConfig）。然后会使用这两个config对象创建NamesrvController实例对象。接下来执行NamesrvController 对象的initialize()、start()方法，并且配置ShutdownHook。initialize()方法中会依次执行加载所有kv配置、创建NettyServer、创建processor线程池、注册processor、使用scheduledExecutorService启动各种scheduled task（包括broker的心跳检测）。
start()方法会执行启动NettyServer。

> 不仅Namesrv的启动过程是这样，其他的组件启动过程也是startup/config/controller这样一个流程。

---------------------
### Namesrv主要组件
-  **KVConfigManager**

> 定义一个HashMap configTable存储配置信息。键为namespace，值为真正存储kv信息的map，这样就可以将同样namespace的配置放入同一个map。
    
```
private final HashMap<String/* Namespace */, HashMap<String/* Key */, String/* Value */>> configTable =
            new HashMap<String, HashMap<String, String>>();
```
> 使用读写锁控制配置信息的加载和读取。

```
private final ReadWriteLock lock = new ReentrantReadWriteLock();
```
KVConfigManager类中load()方法用于启动namesrv时通过configpath读取配置文件，再将配置存入map，另外还提供添加，删除，获取配置方法用于后续操作

- **RouteInfoManager**

> 定义五个map分别存储topic、broker、cluster、brokerliveinfo、filter信息。


```
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    private final HashMap<String/* topic */, List<QueueData>> topicQueueTable;
    private final HashMap<String/* brokerName */, BrokerData> brokerAddrTable;
    private final HashMap<String/* clusterName */, Set<String/* brokerName */>> clusterAddrTable;
    private final HashMap<String/* brokerAddr */, BrokerLiveInfo> brokerLiveTable;
    private final HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;
```

- **DefaultRequestProcessor**

 namesrv启动时会注册processor

```
private void registerProcessor() {
        if (namesrvConfig.isClusterTest()) {
            this.remotingServer.registerDefaultProcessor(new ClusterTestRequestProcessor(this, namesrvConfig.getProductEnvName()), this.remotingExecutor);
        } else {
            this.remotingServer.registerDefaultProcessor(new DefaultRequestProcessor(this), this.remotingExecutor);
        }
    }
```
> 当namesrv有请求过来时，会使用DefaultRequestProcessor去处理请求，处理过程会在线程池this.remotingExecutor中执行，通过processRequest方法处理请求，根据request中不同code进行不同处理。

```
switch (request.getCode()) {
            case RequestCode.PUT_KV_CONFIG:
                return this.putKVConfig(ctx, request);
            case RequestCode.GET_KV_CONFIG:
                return this.getKVConfig(ctx, request);
            case RequestCode.DELETE_KV_CONFIG:
                return this.deleteKVConfig(ctx, request);
            case RequestCode.REGISTER_BROKER:
                Version brokerVersion = MQVersion.value2Version(request.getVersion());
                if (brokerVersion.ordinal() >= MQVersion.Version.V3_0_11.ordinal()) {
                    return this.registerBrokerWithFilterServer(ctx, request);
                }
                else {
                    return this.registerBroker(ctx, request);
                }
            case RequestCode.UNREGISTER_BROKER:
                return this.unregisterBroker(ctx, request);
            case RequestCode.GET_ROUTEINTO_BY_TOPIC:
                return this.getRouteInfoByTopic(ctx, request);
            case RequestCode.GET_BROKER_CLUSTER_INFO:
                return this.getBrokerClusterInfo(ctx, request);
            case RequestCode.WIPE_WRITE_PERM_OF_BROKER:
                return this.wipeWritePermOfBroker(ctx, request);
            case RequestCode.GET_ALL_TOPIC_LIST_FROM_NAMESERVER:
                return getAllTopicListFromNameserver(ctx, request);
            case RequestCode.DELETE_TOPIC_IN_NAMESRV:
                return deleteTopicInNamesrv(ctx, request);
            case RequestCode.GET_KVLIST_BY_NAMESPACE:
                return this.getKVListByNamespace(ctx, request);
            case RequestCode.GET_TOPICS_BY_CLUSTER:
                return this.getTopicsByCluster(ctx, request);
            case RequestCode.GET_SYSTEM_TOPIC_LIST_FROM_NS:
                return this.getSystemTopicListFromNs(ctx, request);
            case RequestCode.GET_UNIT_TOPIC_LIST:
                return this.getUnitTopicList(ctx, request);
            case RequestCode.GET_HAS_UNIT_SUB_TOPIC_LIST:
                return this.getHasUnitSubTopicList(ctx, request);
            case RequestCode.GET_HAS_UNIT_SUB_UNUNIT_TOPIC_LIST:
                return this.getHasUnitSubUnUnitTopicList(ctx, request);
            default:
                break;
        }
```

---------------------
### 其他

   namesrv是无状态的，可以任意水平扩展，每一个broker都与所有namesrv保持长链接(有个scheduled task会按一定频率给所有namesrv做register broker的操作)，所以namesrv之间没有主从关系，他们之间也不需要复制数据。client(producer/consumer)会随机选择一个namesrv进行连接。  

  client和broker中的namesrv地址有以下四种获取方式： 
 
 1. 通过命令行或者配置文件设置namesrv地址。  
 2. 在启动之前通过指定java选项rocketmq.namesrv.addr。  
 3. 设置NAMESRV_ADDR环境变量，brokers和clients会去读取这个环境变量。
 4. 通过一个定时任务每两分钟去一个web服务中获取并更新namesrvaddr的列表。