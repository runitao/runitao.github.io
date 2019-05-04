# HDFS ��Ⱥ����

�������о� Hadoop HDFS ϸ��ʱ, �������������־ϵͳ�͵��Լ�Ⱥ�ļ��ɾ��ǲ��ɻ�ȱ�ļ���. ����������Ҫ���������־ϵͳ(log4j.properties)������, ���� HDFS �ͻ���, NameNode �Լ� DataNode ����. �����޸���������, ��������, �ϵ�����, �����ؼ����չʾ.

����֪ʶ�㶼�ǻ��� Linux �µ� Hadoop-2.7.1.

## ��־ϵͳ

�����ַ�ʽ���Կ���ϵͳ��־����, Web ��ʽ���޸� log4j.properties. ��������Ҫ��λ����ϵͳ����ʱ, ʹ�� Web ��ʽ�ǳ�����, ����Ҫ��������, �����Ծ�ȷ�Ŀ��ƾ��������־�������. �ڲ��Լ�Ⱥ��, �޸� log4j.properties ����һ�����õķ�ʽ, һ������־ϵͳ������Ϊ������ʧЧ, ��һ���滹����ͨ����־ϵͳ�˽�ϵͳ����ϸ������ʽ, ���о��Ƕ���־����Ŀ���, ͨ���޸������ļ�Ҳ�����ֱ�ӵķ�ʽ.

### ����ʵʱ������־

�޸� `http://<namenode>:50070/logLevel` �� `http://<datanode>:50075/logLevel`�޸� NameNode �� DataNode ����־. *ע��, ʹ�����, ��ؼǵûָ�ԭ״*

![logLevel](/assets/images/namenode-loglevel-web.png)


### ���������ļ�
#### �޸������ļ� etc/hadoop/log4j.properties

Դ�����Դ�һ�ݱȽ����������� `hadoop-common-project/hadoop-common/src/main/conf/log4j.properties`. ������Ҫ�����޸�.

1. ֱ�ӵ��� `hadoop.root.logger=DEBUG,console`, ����ȫ�� Debug ����
2. ������־��ʽ `log4j.appender.RFA.layout.ConversionPattern=[%d{yyyy-MM-dd'T'HH:mm:ss.SSSXXX}] [%p] [%t] : %m%n`, ���մ��޸����� `log4j.appender.*.layoutlayout.ConversionPattern` �ĸ�ʽ
3. ���þ��������־���� `log4j.logger.x.y.z=INFO|WARN`, �� `log4j.logger.org.apache.hadoop.hdfs.StateChange=WARN`

#### �޸Ļ�������
����޸��� log4j.properties, ��־û����Ԥ��������ʾ, �����ǻ�������������������µ�.

1. �鿴 etc/hadoop/hadoop-env.sh �� libexec/hadoop-config.sh ���� sbin/hadoop-daemon.sh �� bin/hdfs ��û���������� `-Dhadoop.root.logger` ������ `-D...logger` ֮���������, һ�����������ö����ɻ������� HADOOP_NAMENODE_OPTS, HADOOP_OPTS, HADOOP_ROOT_LOGGER ������, ����
	```
	export HADOOP_NAMENODE_OPTS="-Dhadoop.security.logger=${HADOOP_SECURITY_LOGGER:-INFO,RFAS} -Dhdfs.audit.logger=${HDFS_AUDIT_LOGGER:-INFO,RFAAUDIT}"
	```

2. ���ϵͳ��������(`env`)���Ƿ������� HADOOP_ROOT_LOGGER


## ��Ⱥ���Լ���
IDEA ���������䱸 Proxifier ����Զ�̵� HDFS ����ǳ�����, һ�� IDEA ����ͬʱ���Կͻ���, NameNode, ��� DataNode.

### ���� Proxifier

1. �� Proxies ҳ������Լ��������Ӵ��������
2. �� Rules ҳ����ӹ���, ָ��Ӧ�ó���(Applications) *idea*, Ŀ������(Target Hosts) *10.\*; 172.21.\**, Ŀ��˿�(Target Ports) *Any*, ����(Action) ����һ�����õĴ���

### ����
#### ���õ��Բ���

1. Java ��Զ�̵��Բ������ݲ�ͬ�� JDK �汾���в���, �û�����ͨ�� IDEA ��ȡ��Ӧ JDK �汾�ĵ��Բ���. ������ `suspend=y` ʱ, �����ڵ�������ǰ��ͣ���������.
	```
	-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=9555		# JDK 5-8
	-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:9555	# JDK 9 and later
	```

2. �� etc/hadoop/hadoop-env.sh ������ NameNode, DataNode, Client �ĵ��Բ���, ע�����˿ڳ�ͻ.
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

3. ���������Ӧ��
	```
	hadoop-daemon.sh start namenode 	# namenode
	hadoop-daemon.sh start datanode 	# datanode
	hdfs dfs -put file /tmp/			# client
	```

#### ���� IDEA Զ�̵���
![idea-debug-config](/assets/images/idea-debug-config.png)

#### �ϵ�����
�ڵ��Թ�����, ��Щ����������������߳�. DataNode �������Ե��� NameNode ���������㱨��ȡ���·���ָ��, �����������߳�(H)�͵�ǰ�߳�(C)����������˶ϵ�, �Ϳ��ܵ��������̲߳��ܱ�ִ�л��ܰ�Ԥ�ڵķ�ʽִ��. �����߳� C ��ѭ�������öϵ�, �߳� H ���ܼ�ʱ��Ӧ����, ������Ч������, ���ܰ�Ԥ�ڵķ�ʽִ��. ���Բο���ͼ���жϵ�ľ�׼����.

![idea-debug-breakpoint.png](/assets/images/idea-debug-breakpoint.png)

#### �ؼ���
�ڵ��� HDFS ��Ⱥʱ, �ͻ���, NameNode, DataNode �˴�֮��Ľ�����Ҫ�ر��ע. ����Ҫ�˽�˴˵Ľ�����ʽ��ͨ�� RPC ���� HTTP ��ʽ. ���Ҫ���յ������Ĺؼ�·��. ���ҵĵ��Ծ�����, �������¼������� RPC �Ŀͻ��˳��ںͷ�������.

	```
	NameNodeRpcServer
	DatanodeProtocolClientSideTranslatorPB
	DatanodeProtocolServerSideTranslatorPB
	ClientNamenodeProtocolTranslatorPB
	ClientDatanodeProtocolServerSideTranslatorPB
	```

#### ����������
1. �ڵ��Թ����з���, DataNode ���������ǲ��ܰ�Ԥ�ڴ� NameNode ȡ������, �����鷢�� DataNode ��־���� `[2019-04-19T14:31:17.666+08:00] [WARN] [BP-1516815134-172.21.XX.YY-1555507434409 heartbeating to /172.21.XX.ZZ:8021] : IOException in offerService
java.net.SocketTimeoutException: Call From HADDOP-H1-XX.YY/172.21.XX.YY to HADDOP-H1-XX.ZZ:8021 failed on socket timeout exception: java.net.SocketTimeoutException: 60000 millis timeout while waiting for channel to be ready for read. ch : java.nio.channels.SocketChannel[connected local=/172.21.XX.YY:7504 remote=HADDOP-H1-XX.ZZ/172.21.XX.ZZ:8021]; For more details see:  http://wiki.apache.org/hadoop/SocketTimeout`. ͨ���� hdfs-site.xml �����������, ��� IPC ��ʱ����

	```
	<property>
    	<name>ipc.ping.interval</name>
    	<value>9000000</value>
  	</property>
 	```

