===================
浅析Spark的运行机制
===================

:info: 期望发展出一个系列，记录自己对Spark等通用大数据框架的理解
:author: Marc

背景
====
在使用oozie调pyspark任务时，一开始想当然地认为spark action无法指定driver和executor的python环境，所以直接使用了shell action，期望在shell中配置python环境并使用spark-submit命令提交spark job，但没想到shell脚本在执行时却报出找不到spark-submit命令的错误。在搞明白为什么的过程中，我对oozie、Hadoop、Spark的原理也算是有了新的认识。

下面首先讲讲对oozie、HDFS、YARN和Spark的理解，然后分析spark-submit命令找不到的原因。

一、oozie是什么
===============
仅从我的角度，未参考oozie官方文档，oozie应该有三个主要组成部分：

1. oozie server
2. oozie client
3. HDFS

本质上oozie是一个CS模式的应用，client端发出请求，server端响应请求。在我们的环境中：

1. oozie client就是堡垒机，或者说堡垒机上由oozie command line启动的进程，其发出的请求内容是从指定时间起、到指定时间止、按指定频率启动若干个coordinator
2. oozie server就是由-oozie指定的那台机器，或者说那台机器上运行的server进程，它负责实际启动各coordinator实例
3. HDFS存放一切与被调任务相关的资源（代码、依赖包、配置文件等）

所谓调度是什么
--------------
server调度任务的核心是协调两个方面：

1. 时序逻辑，即在什么时间启动什么任务
2. 资源（内存、CPU），即保证各任务拿到期望的资源

oozie server本质上只管理时序逻辑，资源调度交给YARN来负责。

所谓coordinator是什么
---------------------
oozie server协调时序逻辑的核心是coordinator。

coordinator相当于一个模版，定义了在各时间片上实例化workflow的方式，所谓实例化就是将workflow中的变量名替换为实际值，尤其对那些时间变量，当时间片固定后就都有了确定的值。
实例化是调度的第一步，接下来还要检查input-events中指定的依赖数据是否都已生成，若都已生成，才会真正开始执行workflow。

如何执行workflow
-----------------
执行workflow就是按顺序执行各个action，例如shell action、java action、pig action、hive action、spark action等。
oozie并不会在server机器上直接run这些action，这会使server负载过重。
oozie的方式是，在每次启动action前，先启动一个只有1个mapper的mapreduce任务，将启动action所需的资源全部发送到该mapper上，然后以该mapper为master启动action任务。这实际上是借助YARN的力量实现了负载均衡。

以Spark action为例，server首先启动一个mapreduce job，将Spark环境、Spark程序和其他资源文件等发送到唯一的mapper上，然后以该mapper为master运行spark-submit命令，若是yarn-client模式，则该mapper就是driver，若是yarn-cluster模式，则会在该mapper之外再分配一台机器作为driver。

二、HDFS是什么
==============
HDFS是分布式系统的持久化层，相当于我们在单机上的文件系统。HDFS的职责是存储，义务是容灾。

存储必须满足对任一节点上数据的即时读写，容灾必须满足对任一时刻突发故障的即时恢复，这都要求HDFS要在任一节点上以守护进程的形式驻留，从而定期与master节点交换信息。


三、YARN是什么
==============
YARN是一个很完善的资源调度器，mapreduce只是它的一个应用，任何涉及分布式资源调度的应用都可以寻求YARN的帮助。


四、Spark是什么
===============
Spark不同于HDFS，后者是存储引擎，前者是计算引擎。

存储是一种服务，必须在所有节点上持续驻留，保持活跃，其生命期可以认为是无限的。
而计算是一种ad-hoc行为，只在有需求的时候实施部署，当计算完成后，除了被持久化的数据外，其他资源都会被注销，不会留下任何痕迹，其生命期是有限的。
因此，Spark没有必要也不会被部署并持续驻留在集群内的全部节点上。

实际上，只要保证提交任务的机器上有Spark环境即可，例如堡垒机或oozie server。

通过oozie的spark action提交spark任务
------------------------------------
oozie会将server机器上的Spark环境（jar包）和其他相关资源发送到实际启动任务的mapper上，相当于在该mapper上部署了一套spark环境，然后再在该mapper上通过spark-submit提交spark任务。

通过spark-submit提交spark任务
-----------------------------
无论如何，spark任务最终都是通过spark-submit命令提交的。
按照我的理解，该命令的执行分为以下几步：

1. 跟YARN等资源管理器请求资源，获得若干worker
2. 将当前机器上的Spark环境分发到各worker上，相当于在各worker上即时部署了一套spark环境，但这些spark环境仅存在于当前container中，当任务结束container被回收后，各worker依然恢复未部署状态
3. 在各worker上启动executor进程，开始执行spark任务，控制权交给driver

五、oozie的shell action中无法使用spark-submit命令的原因
=======================================================
结合上述知识，我们来看使用shell action提交spark任务的问题。

当我们使用shell action时，我们实际上是在mapper上运行shell脚本，而该脚本的内容就是执行spark-submit命令，这要求mapper上必须有spark环境。
而mapper作为集群中的一个普通节点，加之我们在shell action配置和shell脚本中并没有做向mapper分发spark环境的操作，因此该mapper上势必不存在spark环境，也就自然找不到spark-submit命令。

作为解决之道，使用spark action自然是最好的选择，oozie会帮我们向mapper分发spark环境。
而如果一定要使用shell action的话，理论上就必须手动在shell action或shell脚本中完成向mapper分发spark环境的操作，当然这只是推断，是否可行还有待实验。
