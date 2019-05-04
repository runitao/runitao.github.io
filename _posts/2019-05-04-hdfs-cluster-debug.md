# HDFS 集群调试

在深入研究 Hadoop HDFS 细节时, 掌握如何配置日志系统和调试集群的技巧就是不可或缺的技能. 在这里面我要分享的是日志系统(log4j.properties)的设置, 调试 HDFS 客户端, NameNode 以及 DataNode 方法. 包括修改启动参数, 参数配置, 断点设置, 几个关键类的展示.

以下知识点都是基于 Linux 下的 Hadoop-2.7.1.

## 日志系统

有两种方式可以控制系统日志级别, Web 方式和修改 log4j.properties. 当我们需要定位线上系统问题时, 使用 Web 方式非常合适, 不需要重启服务, 还可以精确的控制具体类的日志输出级别. 在测试集群中, 修改 log4j.properties 会是一个更好的方式, 一方面日志系统不会因为重启而失效, 另一方面还能在通过日志系统了解系统的详细运作方式, 再有就是对日志整体的控制, 通过修改配置文件也是最简单直接的方式.

### 线上实时调整日志

修改 `http://<namenode>:50070/logLevel` 和 `http://<datanode>:50075/logLevel`修改 NameNode 和 DataNode 的日志. *注意, 使用完后, 务必记得恢复原状*

![logLevel](/assets/images/namenode-loglevel-web.png)


### 调整配置文件
#### 修改配置文件 etc/hadoop/log4j.properties

源码中自带一份比较完整的配置 `hadoop-common-project/hadoop-common/src/main/conf/log4j.properties`. 根据需要进行修改.

1. 直接调整 `hadoop.root.logger=DEBUG,console`, 设置全局 Debug 级别
2. 设置日志格式 `log4j.appender.RFA.layout.ConversionPattern=[%d{yyyy-MM-dd'T'HH:mm:ss.SSSXXX}] [%p] [%t] : %m%n`, 依照此修改所有 `log4j.appender.*.layoutlayout.ConversionPattern` 的格式
3. 设置具体类的日志级别 `log4j.logger.x.y.z=INFO|WARN`, 如 `log4j.logger.org.apache.hadoop.hdfs.StateChange=WARN`

#### 修改环境变量
如果修改了 log4j.properties, 日志没有如预期那样显示, 可能是环境变量或启动配置项导致的.

1. 查看 etc/hadoop/hadoop-env.sh 和 libexec/hadoop-config.sh 还有 sbin/hadoop-daemon.sh 和 bin/hdfs 有没有设置诸如 `-Dhadoop.root.logger` 或其它 `-D...logger` 之类的配置项, 一般这样的配置都是由环境变量 HADOOP_NAMENODE_OPTS, HADOOP_OPTS, HADOOP_ROOT_LOGGER 等设置, 如下
	```
	export HADOOP_NAMENODE_OPTS="-Dhadoop.security.logger=${HADOOP_SECURITY_LOGGER:-INFO,RFAS} -Dhdfs.audit.logger=${HDFS_AUDIT_LOGGER:-INFO,RFAAUDIT}"
	```

2. 检查系统环境变量(`env`)中是否设置了 HADOOP_ROOT_LOGGER


## 集群调试技巧
IDEA 开发环境配备 Proxifier 调试远程的 HDFS 服务非常方便, 一个 IDEA 可以同时调试客户端, NameNode, 多个 DataNode.

### 设置 Proxifier

1. 在 Proxies 页面根据自己的情况添加代理服务器
2. 在 Rules 页面添加规则, 指定应用程序(Applications) *idea*, 目标主机(Target Hosts) *10.\*; 172.21.\**, 目标端口(Target Ports) *Any*, 动作(Action) 走上一步设置的代理

### 调试
#### 设置调试参数

1. Java 的远程调试参数根据不同的 JDK 版本是有差别的, 用户可以通过 IDEA 获取对应 JDK 版本的调试参数. 当设置 `suspend=y` 时, 可以在调试启动前暂停服务的运行.
	```
	-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=9555		# JDK 5-8
	-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:9555	# JDK 9 and later
	```

2. 在 etc/hadoop/hadoop-env.sh 中设置 NameNode, DataNode, Client 的调试参数, 注意避免端口冲突.
	```
	# NameNode
	export HADOOP_NAMENODE_DEBUG_OPTS="-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=9555"
	export HADOOP_NAMENODE_OPTS="$HADOOP_NAMENODE_DEBUG_OPTS $HADOOP_NAMENODE_OPTS"
	# DataNode
	export HADOOP_DATANODE_DEBUG_OPTS="-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=9556"
	export HADOOP_DATANODE_OPTS="$HADOOP_DATANODE_DEBUG_OPTS $HADOOP_DATANODE_OPTS"
	# Client 
	export HADOOP_CLIENT_DEBUG_OPTS="-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=9557"
	export HADOOP_CLIENT_OPTS="$HADOOP_CLIENT_DEBUG_OPTS $HADOOP_CLIENT_OPTS"
	```

3. 启动服务或应用
	```
	hadoop-daemon.sh start namenode 	# namenode
	hadoop-daemon.sh start datanode 	# datanode
	hdfs dfs -put file /tmp/			# client
	```

#### 配置 IDEA 远程调试
![idea-debug-config](/assets/images/idea-debug-config.png)

#### 断点设置
在调试过程中, 有些情况不能阻塞所有线程. DataNode 会周期性的向 NameNode 进行心跳汇报并取走下发的指令, 处理心跳的线程(H)和当前线程(C)如果都加上了断点, 就可能导致心跳线程不能被执行或不能按预期的方式执行. 比如线程 C 在循环中设置断点, 线程 H 不能及时响应请求, 导致无效请求激增, 不能按预期的方式执行. 可以参考下图进行断点的精准控制.

![idea-debug-breakpoint.png](/assets/images/idea-debug-breakpoint.png)

#### 关键类
在调试 HDFS 集群时, 客户端, NameNode, DataNode 彼此之间的交互需要特别关注. 首先要了解彼此的交互方式是通过 RPC 还是 HTTP 方式. 其次要掌握调用链的关键路径. 在我的调试经验中, 发现如下几个类是 RPC 的客户端出口和服务端入口.

	```
	NameNodeRpcServer
	DatanodeProtocolClientSideTranslatorPB
	DatanodeProtocolServerSideTranslatorPB
	ClientNamenodeProtocolTranslatorPB
	ClientDatanodeProtocolServerSideTranslatorPB
	```

#### 参数配置项
1. 在调试过程中发现, DataNode 的心跳总是不能按预期从 NameNode 取走任务, 经调查发现 DataNode 日志报错 `[2019-04-19T14:31:17.666+08:00] [WARN] [BP-1516815134-172.21.XX.YY-1555507434409 heartbeating to /172.21.XX.ZZ:8021] : IOException in offerService
java.net.SocketTimeoutException: Call From HADDOP-H1-XX.YY/172.21.XX.YY to HADDOP-H1-XX.ZZ:8021 failed on socket timeout exception: java.net.SocketTimeoutException: 60000 millis timeout while waiting for channel to be ready for read. ch : java.nio.channels.SocketChannel[connected local=/172.21.XX.YY:7504 remote=HADDOP-H1-XX.ZZ/172.21.XX.ZZ:8021]; For more details see:  http://wiki.apache.org/hadoop/SocketTimeout`. 通过在 hdfs-site.xml 添加如下配置, 解决 IPC 超时问题

	```
	<property>
    	<name>ipc.ping.interval</name>
    	<value>9000000</value>
  	</property>
 	```

