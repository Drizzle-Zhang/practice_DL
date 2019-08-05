# 【Task5】Spark常用API

## 1. spark集群搭建
解压并移动到相应目录
```Bash
tar -zxvf spark-2.1.1-bin-hadoop2.7.tgz
```
修改/etc/profie，增加如下内容：
```
export SPARK_HOME=/local/zy/tools/spark-2.1.1-bin-hadoop2.7
export PATH=$PATH:$SPARK_HOME/bin
```
复制spark-env.sh.template成spark-env.sh，并添加以下内容
```
[zy@node7 spark-2.1.1-bin-hadoop2.7]$ cd conf/
[zy@node7 conf]$ ls
docker.properties.template  metrics.properties.template   spark-env.sh.template
fairscheduler.xml.template  slaves.template
log4j.properties.template   spark-defaults.conf.template
[zy@node7 conf]$ cp spark-env.sh.template spark-env.sh
[zy@node7 conf]$ vi spark-env.sh
###
export HADOOP_HOME=/local/zy/tools/hadoop-2.7.3
export HADOOP_CONF_DIR=/local/zy/tools/hadoop-2.7.3/etc/hadoop
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.151-1.b12.el7_4.x86_64/jre
# 设置Master的主机名
export SPARK_MASTER_IP=10.152.255.52
export SPARK_LOCAL_IP=10.152.255.52
export SPARK_MASTER_HOST=10.152.255.52
# 提交Application的端口，默认就是这个，万一要改呢，改这里
export SPARK_MASTER_PORT=7077
# 每一个Worker最多可以使用的cpu core的个数，我虚拟机就一个;真实服务器如果有32个，你>可以设置为32个
export SPARK_WORKER_CORES=20
# 每一个Worker最多可以使用的内存，我的虚拟机就2g
# 真实服务器如果有128G，你可以设置为100G
export SPARK_WORKER_MEMORY=10g
#export SCALA_HOME=/home/heitao/Soft/scala-2.11.0
export SPARK_HOME=/local/zy/tools/spark-2.1.1-bin-hadoop2.7
#export SPARK_DIST_CLASSPATH=$(/home/heitao/Soft/hadoop-2.7.3/bin/hadoop classpath)
###
```
复制slaves.template成slaves，并添加以下内容
```Bash
cp slaves.template slaves
###
node7
node8
node9
###
```
将配置好的spark文件复制到Slave1和Slave2节点。
```Bash
scp -r ./spark-2.1.1-bin-hadoop2.7 zy@10.152.255.53:/local/zy/tools
scp -r ./spark-2.1.1-bin-hadoop2.7 zy@10.152.255.54:/local/zy/tools
```
然后再修改其他节点的bashrc文件,增加Spark的配置，过程同Master一样<br>
再修改spark-env.sh，将export SPARK_LOCAL_IP=xxx.xxx.xxx.xxx改成node8和node9对应节点的IP。<br><br>
在Master节点启动集群
```Bash
[zy@node7 spark-2.1.1-bin-hadoop2.7]$ ./sbin/start-all.sh 
```
查看集群是否启动成功
```Bash[zy@node7 spark-2.1.1-bin-hadoop2.7]$ jps
19408 SecondaryNameNode
19121 NameNode
20722 ResourceManager
177040 Jps
176721 Worker
176661 Worker
176429 Master
[zy@node8 ~]$ jps
4881 Jps
183451 DataNode
184153 NodeManager
4655 Worker
```
node7在Hadoop的基础上新增了：Master<br>
node8,node9在Hadoop的基础上新增了：Worker<br><br>

**Reference:**<br>
1. [Spark伪分布式环境搭建 + jupyter连接spark集群 ](https://mp.weixin.qq.com/s?__biz=MzI3Mjg1OTA3NQ==&mid=2247483893&idx=1&sn=84496036abf5c302806f2daa9655bd6a&chksm=eb2d6b59dc5ae24fae6d483547778fc7bbe1054094ca97b4c152fdd79bbc8b0ed3e14ffefbfd&mpshare=1&scene=1&srcid=&sharer_sharetime=1564900973740&sharer_shareid=8ac76a2e8d1b620817577ca68d2d215f&key=1a2eded5d1d2d5f7e5ff6b1d226694b4aed37afd13c5472df38cb96541ce31181690e47492a199c87dcdff8410f92dd3c4fb25dc3c00b4c5dba4e30a0ef826d8c81f4db1037d3fbcffbc2f31f3cdb1ef&ascene=1&uin=MTExNjkzNDEwNg%3D%3D&devicetype=Windows+10&version=62060833&lang=zh_CN&pass_ticket=Ika06k3RtS5H%2BXm0gmTpvebwTIuC5uQymoAZxQ6aQvMyKbEjXvF2WCwOqYhWuCiN)<br>


## 2. 初步认识Spark （解决什么问题，为什么比Hadoop快，基本组件及架构Driver/）
### 定义
Spark是Apache的一个顶级项目，是一个快速、通用的大规模数据处理引擎

### Spark比Hadoop快的主要原因
**1.消除了冗余的HDFS读写**<br>
Hadoop每次shuffle操作后，必须写到磁盘，而Spark在shuffle后不一定落盘，可以cache到内存中，以便迭代时使用。如果操作复杂，很多的shufle操作，那么Hadoop的读写IO时间会大大增加。<br>
**2.消除了冗余的MapReduce阶段**<br>
Hadoop的shuffle操作一定连着完整的MapReduce操作，冗余繁琐。而Spark基于RDD提供了丰富的算子操作，且reduce操作产生shuffle数据，可以缓存在内存中。<br>
**3.JVM的优化**<br>
Spark Task的启动时间快。Spark采用fork线程的方式，Spark每次MapReduce操作是基于线程的，只在启动。而Hadoop采用创建新的进程的方式，启动一个Task便会启动一次JVM。Spark的Executor是启动一次JVM，内存的Task操作是在线程池内线程复用的。每次启动JVM的时间可能就需要几秒甚至十几秒，那么当Task多了，这个时间Hadoop不知道比Spark慢了多少。<br>

### Spark基本组件
![](https://github.com/Drizzle-Zhang/practice/blob/master/big_data_basis/supp_Task5/elements.png)<br>
核心部分是RDD相关的，就是我们前面介绍的任务调度的架构，后面会做更加详细的说明。<br>

**SparkStreaming：**<br>
基于SparkCore实现的可扩展、高吞吐、高可靠性的实时数据流处理。支持从Kafka、Flume等数据源处理后存储到HDFS、DataBase、Dashboard中。<br><br>

**MLlib：**<br>
关于机器学习的实现库。<br><br>

**SparkSQL：**<br>
Spark提供的sql形式的对接Hive、JDBC、HBase等各种数据渠道的API，用Java开发人员的思想来讲就是面向接口、解耦合，ORMapping、Spring Cloud Stream等都是类似的思想。<br><br>

**GraphX：**<br>
关于图和图并行计算的API。<br><br>
 
**RDD(Resilient Distributed Datasets) 弹性分布式数据集**<br>
RDD支持两种操作：转换（transiformation）和动作（action）<br>
转换就是将现有的数据集创建出新的数据集，像Map；动作就是对数据集进行计算并将结果返回给Driver，像Reduce。<br>
RDD中转换是惰性的，只有当动作出现时才会做真正运行。这样设计可以让Spark更见有效的运行，因为我们只需要把动作要的结果送给Driver就可以了而不是整个巨大的中间数据集。<br>
缓存技术（不仅限内存，还可以是磁盘、分布式组件等）是Spark构建迭代式算法和快速交互式查询的关键，当持久化一个RDD后每个节点都会把计算分片结果保存在缓存中，并对此数据集进行的其它动作（action）中重用，这就会使后续的动作（action）变得跟迅速（经验值10倍）。例如RDD0àRDD1àRDD2，执行结束后RDD1和RDD2的结果已经在内存中了，此时如果又来RDD0àRDD1àRDD3，就可以只计算最后一步了。<br><br>



### Spark架构
![](https://github.com/Drizzle-Zhang/practice/blob/master/big_data_basis/supp_Task5/structure.png)<br>
**ClusterManage**r<br>
负责分配资源，有点像YARN中ResourceManager那个角色，大管家握有所有的干活的资源，属于乙方的总包。<br><br>

**WorkerNode**<br>
是可以干活的节点，听大管家ClusterManager差遣，是真正有资源干活的主。<br><br>

**Executor**<br>
是在WorkerNode上起的一个进程，相当于一个包工头，负责准备Task环境和执行Task，负责内存和磁盘的使用。<br><br>

**Task**<br>
是施工项目里的每一个具体的任务。<br><br>

**Driver**<br>
是统管Task的产生与发送给Executor的，是甲方的司令员。<br><br>

**SparkContext**<br>
是与ClusterManager打交道的，负责给钱申请资源的，是甲方的接口人。<br><br>


整个互动流程是这样的：<br>
![](https://github.com/Drizzle-Zhang/practice/blob/master/big_data_basis/supp_Task5/structure2.jpeg)<br>
1 甲方来了个项目，创建了SparkContext，SparkContext去找ClusterManager申请资源同时给出报价，需要多少CPU和内存等资源。ClusterManager去找WorkerNode并启动Excutor，并介绍Excutor给Driver认识。<br>

2 Driver根据施工图拆分一批批的Task，将Task送给Executor去执行。<br>

3 Executor接收到Task后准备Task运行时依赖并执行，并将执行结果返回给Driver。<br>

4 Driver会根据返回回来的Task状态不断的指挥下一步工作，直到所有Task执行结束。<br>


**Reference:**<br>
1. [Spark基础全解析](https://blog.csdn.net/vinfly_li/article/details/79396821)<br>
2. [spark为什么比hadoop的mr要快？(https://www.cnblogs.com/wqbin/p/10217994.html)<br>
3. [Spark原理详解](https://blog.csdn.net/yejingtao703/article/details/79438572)<br>


## 










