在我们的项目当中，使用定时任务是避免不了的，我们在部署定时任务时，通常只部署一台机器。部署多台机器时，同一个任务会执行多次。比如给用户发送邮件定时任务，每天定时的给用户下发邮件。如果部署了多台，同一个用户将发送多份邮件。只部署一台机器，可用性又无法保证。Elastic-Job框架可以帮助解决定时任务在集群部署情况下的协调调度问题，保证任务不重复不遗漏的执行。
Elastic-Job是当当网在2015年开源的一个分布式调度解决方案，由两个相互独立的子项目Elastic-Job-Lite和Elastic-Job-Cloud组成。在这里，主要对Elastic-Job-Lite技术做分析。

分布式定时任务调度平台Elastic-Job技术详解
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200415155606749.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L215c3dlZXQxMTE=,size_16,color_FFFFFF,t_70)
Elastic-Job-Lite架构图

Elastic-Job-Lite框架使用zookeeper作为注册中心，从架构图可以看出，Elastic-Job-Lite框架通过监听器感知zookeeper数据的变化，并做相应的处理；运维平台也仅是通过读取zk数据展现作业状态，或更新zk数据修改全局配置。运维平台和Elastic-Job-Lite没有直接关系，完全解耦合。Elastic-Job-Lite并不直接提供数据处理的功能，框架只会将分片项分配至各个运行中的作业服务器，分片项与真实数据的对应关系需要开发者在应用程序中自行处理。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200415155805237.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L215c3dlZXQxMTE=,size_16,color_FFFFFF,t_70)
Elastic-Job-Lite节点图

   想要了解Elastic-Job-Lite技术原理，应该先了解它在zookeeper上的节点情况。注册中心在定义的命名空间下，创建作业名称节点，用于区分不同作业，所以作业一旦创建则不能修改作业名称，如果修改名称将视为新的作业。作业名称节点下又包含5个数据子节点，分别是config, instances, sharding, servers和leader。config节点保存作业配置信息，以JSON格式存储；instances节点保存作业运行实例信息，子节点是当前作业运行实例的主键；sharding节点保存作业分片信息，子节点是分片项序号，从零开始，至分片总数减一；servers节点保存作业服务器信息，子节点是作业服务器的IP地址；leader节点保存作业服务器主节点信息，分为election，sharding和failover三个子节点，分别用于主节点选举，分片和失效转移处理。更详细的节点说明请看上图或Elastic-Job官方文档。

  Elastic-Job-Lite源码由两个入口，一个是JobSchedule类的init方法，当启动服务时，程序会进入该方法，进行任务的初始化等操作；另一个是LiteJob类，它实现了Quartz框架的Job接口，当定时任务启动时，会进入该实现类，完成失效转移项执行、重新分片、获取并执行本机任务项、错过任务重触发等操作。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200415155840436.png)
作业启动流程图

   Elastic-Job-Lite初始化的入口是JobSchedule，应用服务器启动时，会调用JobSchedule的init方法，开启作业启动流程。首先添加或更新作业配置信息，并将配置信息持久化到zk上；接着创建quartz调度器，作业的调度执行依赖quartz技术；然后启动所有的监听器，包括leader选举监听、失效转移监听、分片监听等，并发起主节点选举，将leader节点信息set到leader/election/instance节点下；然后将服务器信息、实例信息注册到zk上，并且在leader/sharding下创建necessary节点，作为重新分片的标记；最后由quartz调度器根据cron表达式调度执行。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200415155854434.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L215c3dlZXQxMTE=,size_16,color_FFFFFF,t_70)
作业执行流程图

  Elastic-Job-Lite执行器的入口是实现了Job接口的LiteJob类，当任务调度执行时，进入LiteJob类的execute方法。在这里完成一系列的操作，包括获取失效转移分片项，如果没有分配的失效转移项，则判断是否需要重新分片，然后获取分配给自己的分片项，然后判断当前分片项是否正在running，如果否，则执行任务项；如果是，则在sharding/[item]下添加misfire节点，标示该分片项错过执行，等待分片项执行结束后，再触发misfire的分片项执行。

总结：

1. Elastic-Job-Lite框架的任务处理、执行等都是针对的分片项，也就是说quartz调度执行面向的是定时任务的分片项，而不是定时任务本身。如果我们不打算对定时任务分片，那么可以把分片数设为1，这样在sharding节点下创建1个分片项0，0分片项将会被分配给一个instance，并启动执行。

2. Elastic-Job-Lite是提供失效转移功能的，即当正在执行的任务项遇到进程退出或机器宕机等故障，该任务项应该转移到某空闲服务器执行。但是，该功能存在bug：（1）失效转移只能在同一机器上不同实例间完成，跨机器无效；（2）没有判断机器宕机时任务项是否正在执行状态，而是只要遇到宕机，即使任务项没有开始执行，也被转移到其他机器上执行一遍，导致重复执行。

3. Elastic-Job-Lite是预分片，不是动态分片，即Elastic-Job-Lite是在服务启动时就完成分片项的创建和分配，并保存在zk节点上，而不是定时任务在每次启动时，根据机器的处理能力，重新分配任务项，例如：任务比较繁忙的机器不参与新一轮的任务项分配。这样做的目的是对zk弱依赖，如果进行真正的动态分片，对于秒级的定时任务将会产生很大影响，因为每次任务启动，应用服务器都要跟zookeeper进行通信，并执行重新分片的逻辑，频繁的通信，对于秒级定时任务，会错过很多次执行
