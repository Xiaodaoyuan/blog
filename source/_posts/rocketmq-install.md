---
title: Rocketmq安装部署
date: 2019-12-17 19:34:20
categories: middleware
tags: rocketmq
---

### 获取源码,打包编译

1. 从github下载源码，git  clone  https://github.com/apache/rocketmq.git
2. 编译源码，进入到主目录，然后执行命令： mvn -Prelease-all -DskipTests clean install -U
3. 编译完之后我们只需要下图红框之内的目录进行部署。
![iamge](/images/rocketmq-install.png)
4. 上图中bin目录是存放部署启动的脚本，conf目录存放的是配置文件，lib目录是打包好之后的所有jar包。

### 启动Namesrv
<p style="text-indent:2em">首先设置好系统JAVA_HOME变量（必须使用64位JDK），然后在主目录下执行nohup sh bin/mqnamesrv &，启动成功之后可以查看到进程，端口默认是9876。(启动默认指定的堆内存是4G，我们自己测试时可以通过修改runserver.sh文件中参数适当改小)</p>

### 启动broker
&nbsp;&nbsp;&nbsp;broker集群主要有四种配置方式：单master，多master，多master多slave同步双写，多master多slave异步复制。（现在好像多了一种dledger的方式，暂时还没研究，不太明白）。  
&nbsp;&nbsp;&nbsp;Master和Slave的配置文件参考conf目录下的配置文件。Master与Slave通过指定相同的brokerName参数来配对，Master的BrokerId必须是0，Slave的BrokerId必须是大于0的数。  
&nbsp;&nbsp;&nbsp;启动命令：nohup sh bin/mqbroker -n localhost:9876  -c  conf/2m-2s-async/broker-a.properties  &  

### 其他
- **查看日志**  
namesrv和broker的日志默认存在下面所示目录中：  
tail -f ~/logs/rocketmqlogs/namesrv.log  
tail -f ~/logs/rocketmqlogs/broker.log
- **关闭服务器**  
 分别关闭broker和namesrv:  
 sh bin/mqshutdown broker    
 sh bin/mqshutdown namesrv 
  
-  **其他命令**  
查看集群情况： ./bin/mqadmin clusterList -n localhost:9876  
查看broker状态： ./bin/mqadmin brokerStatus -n localhost:9876 -b localhost:10911  
查看topic列表： ./bin/mqadmin topicList -n localhost9876  
查看topic状态： ./bin/mqadmin topicStatus -n localhost:9876 -t MyTopic (换成想查询的 topic)  
查看topic路由： ./bin/mqadmin topicRoute -n localhost:9876 -t MyTopic  

### Rocketmq-Console安装

1. 使用git命令下载项目源码，由于我们仅需要rocketmq-console，故下载此项目对应分支即可。`git clone -b release-rocketmq-console-1.0.0 https://github.com/apache/rocketmq-externals.git`
2. 进入项目文件夹并修改配置文件
    `cd rocketmq-externals/rocketmq-console/
    vi src/main/resources/application.properties`  (可以修改rocketmq.config.namesrvAddr，当然也可以后面启动时通过参数指定)
3. 执行maven命令打成jar包：`mvn clean package -Dmaven.test.skip=true`
4. 启动： `java -jar -n localhost:9876  rocketmq-console-ng-1.0.0.jar  &`
5. 浏览器访问http://localhost:8080就可以查看管理界面。

    
