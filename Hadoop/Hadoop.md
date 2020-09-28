# Hadoop



### 分布式文件存储系统——HDFS
HDFS是Hadoop下的分布式文件系统，具有高容错、高吞吐量等特性，可以部署在低成本的硬件上。

构建于单个磁盘之上的一般文件系统，会将文件划分为文件块，并将这些文件块存储到磁盘块上（文件块大小是磁盘块的整数倍）。

而HDFS中也有这种**[数据块](# HDFS中使用数据块的好处)**（block）的概念。**集群中的所有硬件设备都会被分为block，而文件则会被分为块大小的多个分块(chunk)，这些分块被保存为多个副本存储在集群中的多个节点中**。HDFS的block一般比较大（128M），这是为了**最小化寻址开销**，这样传输一个由多个块组成的大文件的时间主要取决于磁盘的传输速率。

#### 1、HDFS设计原理
##### 1.1 HDFS架构

![image-20200918153654839](D:%5CJob%5Clearn-BigData%5CHadoop%5CHadoop.assets%5Cimage-20200918153654839.png)

HDFS遵循主/从架构，即一个namenode(NN)和多个datanode(DN):

- **namenode**：管理**文件系统的命名空间**，维护着文件系统树以及整棵树内所有**文件和目录**，例如打开、关闭、重命名文件和目录等操作。这些信息以：**命名空间镜像文件**和**编辑日志文件**的形式永久保存在本地磁盘中。另外，namenode记录着每个文件中各个块所在的数据节点信息。整个HDFS可存储的文件数受限于namenode的内存大小。

  - 存储元数据
  - 文件操作：client进行文件读写操作时，会询问namenode文件的存放地址，然后直接和datanode连接。
  - 文件块副本的复制规则：机架感知策略。
  - 接收datanode的心跳信号
- **datanode**：是文件系统的工作节点。它们根据需要**存储并检索数据块**（一般是受客户端或namenode的调度），并且定期向namenode发送他们所存储的数据块列表。
  - 以数据块的形式存储HDFS文件
  - 响应clien的读写请求
  - 向namenode定期汇报心跳信号、数据块信息、块缓存信息等



##### 1.2 文件系统命名空间

HDFS的文件系统命名空间的层次结构与大多数文件系统类似（如Linux），支持目录和文件的创建、移动、删除和重命名等操作，支持配置用户和访问权限，但**不支持[硬链接和软链接](# Linux的硬链接与软链接)**。



##### 1.3 数据复制

由于Hadoop被设计运行在廉价的机器上，这意味着硬件是不可靠的，因此，为了保证容错性，HDFS提供了数据复制机制。HDFS将每个文件存储为一系列**块**，每个块有多个副本来保证容错，块的大小和复制因子可以自行配置（默认情况下，块大小是128M，复制因子是3）。

![image-20200918153718359](D:%5CJob%5Clearn-BigData%5CHadoop%5CHadoop.assets%5Cimage-20200918153718359.png)



##### 1.4 数据复制的实现原理

大型的HDFS实例常常部署在多个机架上的多台服务器上，不同机架上的两台服务器通过交换机进行通讯。通常情况下，同一机架上的服务器间的网络带宽大于不同机架上服务器之间的的带宽。因此，HDFS采用**机架感知副本放置策略**，常见情况下，复制因子为3时：

写入程序位于datanode上时，优先将写入文件的一个副本放在该datanode上，否则随机放在随机datanode上。之后在另一个远程机架上的任意一个节点上存放另一个副本，并在该机架上的另一个节点上放置最后一个副本。此策略可以减少机架之间的写入流量，从而提高写入性能。
![image-20200918153735051](D:%5CJob%5Clearn-BigData%5CHadoop%5CHadoop.assets%5Cimage-20200918153735051.png)

当复制因子大于3，则随机确定第4个和之后的副本放置位置，不允许同一个datanode上具有同一个块的多个副本。



##### 1.5 副本的选择

为了最大限度地减少带宽消耗和读取延迟，HDFS在执行读取请求时，优先读取距离读取器最近的副本。



##### 1.6 架构的稳定性

- 心跳机制和重新复制

  每个datanode会定期向namenode发送心跳消息，如果超过指定时间没有收到心跳消息，则将datanode标记为死亡。由于数据不再可用，可能会导致某些块的复制因子小于指定值，namenode会在必要的时候重新复制。

- 数据的完整性

  当整个系统需要处理的数据量达到Hadoop的处理极限时，数据被损坏的概率是很大的。因此，HDFS需要提供数据完整性机制来保证数据的完整性，具体来说就是利用CRC（循环冗余校验）。

  当客户端创建HDFS文件时，它会计算文件的**每个数据块的校验和**，并将校验和存储在HDFS的命名空间下的隐藏文件中。当客户端检索文件内容时，它会验证从每个datanode接收的数据是否与存储在HDFS命名空间中的校验和匹配。若匹配失败，则证明数据已经损坏，此时客户端会选择其他datanode上的副本。

- [元数据的磁盘故障](# 元数据的磁盘故障)
  namenode上包含了元数据（在内存中），因此当namenode损坏后，整个文件系统的所有文件都会丢失。因此，namenode的容错性非常重要，为此Hadoop为此提供了两种机制：**备份元数据**（**FsImage**（HDFS系统中存储的文件路径）和**EditLog**（HDFS系统中记录所有操作的日志文件））和在另一个单独的物理计算机上运行一个**辅助namenode**。

- 支持快照

  快照支持在特定时刻存储数据副本，在数据意外损坏时，可以通过回滚操作恢复到健康的数据状态。
  
  

#### 2、HDFS的特点
##### 2.1 高容错
##### 2.2 高吞吐量
##### 2.3 大文件支持
##### 2.4 简单一致性模型
##### 2.5 跨平台移植性



#### 3、HDFS常用命令

#### 4、HDFS文件写入详述

<img src="D:%5CJob%5Clearn-BigData%5CHadoop%5CHadoop.assets%5Cimage-20200923160641860.png" alt="image-20200923160641860" style="zoom: 50%;" />



1：client向nn请求上传文件；

2：nn会检测上传的权限；

3：nn检查完权限后，会向client发送允许上传的请求；

4：client会将文件划分block，然后向nn请求上传第一个block；

5：nn收到请求后，检测datanode的空闲状态并按照机架感知策略选出可以上传的3个主机；

6：nn将选出的dn列表发送给client；

7：client会根据接收到的dn列表，与离client最近的dn1单独建立pipeline。然后，dn1会依次往后建立pipeline；

8：client将blk1传输给dn1，以packet（64k）为单位并沿着pipeline把blk1传递给dn1，并依次沿管道往后传；

9：每个dn在接收到数据时，会缓存在本地；

10：每接收一个packet，管道最后一个dn会返回一个应答，并沿着管道往前传到client；

第一个block传递完后，会回到第4步，开始上传blk2。



#### 5、HDFS文件读取详述

![image-20200923165417862](D:%5CJob%5Clearn-BigData%5CHadoop%5CHadoop.assets%5Cimage-20200923165417862.png)

[^1]: 黑马程序员：大数据基础

1：client向nn请求下载文件；（RPC远程调用）

2：nn权限检查。权限满足后，nn要检查文件block列表，在集群中就近选择block副本对应的主机列表；

3：返回block主机列表；

4：client根据收到的主机列表，与对应的nn建立pipeline；

5：client开始以packet为单位读取数据，所有pipeline并行读取；

6：将多个block合并成完整的文件；



#### 参考

##### Linux的硬链接与软链接

##### HDFS中使用数据块的好处

对分布式文件系统中的块进行抽象会带来很多好处。

- 一个文件的大小可以大于网络中任意一个磁盘的容量，组成文件的所有数据块可以被分散在集群中的多个磁盘上；
- 使用数据块作为存储单元，大大简化了存储子系统的设计。由于数据块的大小是固定的，可以很容易知道单个磁盘能存储多少个块。且数据块只需要存储大块的数据，而文件的元数据可以单独存储，方便管理；
- 每个块可以备份到多个地方，保证了文件系统的容错性和数据的完整性。

##### 元数据的磁盘故障

为保证namenode的容错性，Hadoop提供了两种机制：**元数据备份**和**辅助namenode**。

- **元数据备份**

  元数据信息包括文件的创建时间、大小’权限以及块列表等。由于元数据都存储在内存中，因此需要将元数据持久化保存到硬盘中，也就是`FsImage`和`EditLog`两个文件。

  `FsImage`相当于将namenode中的元数据持久化到硬盘的一个镜像文件，但它不会与nn进行同步更新，所以不一定备份到最新的元数据状态。

  `EditLog`记录了最近一段时间对元数据的操作日志。

  由于`FsImage`文件并不是最新的元数据状态，因此需要配合`EditLog`中的日志记录，才能在nn重启后完整地恢复元数据。

- **辅助namenode**

  频繁的元数据操作会导致`EditLog`文件过大，`SecondaryNameNode`会定期将namenode的`FsImage`和`EditLog`文件拷贝过来，将两个文件合并成新的`FsImage`，然后将原来的`EditLog`文件清空。接着，利用这个新的`FsImage`替换nn原来的`FsImage`。

  `SecondaryNameNode`的触发条件一般为时间和文件大小，默认值分别为1h和64M。它一般运行在另外一台单独的主机上，在namenode损坏后，`SecondaryNameNode`可以快速且完整的恢复namenode。

  

### 分布式计算框架——MapReduce

#### 1、MapReduce设计思想

MapReduce是一种用于数据处理的编程模型，本质上是并行运行的。可以将大规模数据分析任务分发给拥有多机器的数据中心。MapReduce任务主要包括两个部分：Map任务和Reduce任务。这两个任务高度抽象了大数据任务的并行计算过程。



>MR的设计思想是，从HDFS获得输入数据，将输入的一个大的数据集分割成多个小数据集，然后并行计算这些小数据集（Map），最后将每个小数据集的结果进行汇总（Reduce），得到最终的结果并将其输出到HDFS中。

<img src="D:%5CJob%5Clearn-BigData%5CHadoop%5CHadoop.assets%5C1565182220824.png" alt="1565182220824" style="zoom:150%;" />



MapReduce作业通过将输入的数据集拆分为独立的块，这些块由map以并行的方式处理，框架对map的输出进行排序，然后输入到reduce中。MR框架专门用于<key,value>键值对处理。

![image-20200918172730657](D:%5CJob%5Clearn-BigData%5CHadoop%5CHadoop.assets%5Cimage-20200918172730657.png)

MapReduce作业(job)是客户端需要执行的一个工作单元，包括：输入数据、MapReduce程序和配置信息。

Hadoop将MapReduce的输入数据划分为等长的小数据块，称为输入分片（分片），然后为每个分片构建一个map任务。Hadoop在存储有输入数据（HDFS中的数据）的节点上运行map任务，可以无需使用集群带宽资源，这就是所谓的“数据本地化优化”。然后将排序过的map输出数据发送到运行reduce任务的节点上进行合并，然后执行reduce函数。



#### 2、MapReduce的工作原理



#### 3、MapReduce在Yarn中的工作流程

![img](D:%5CJob%5Clearn-BigData%5CHadoop%5CHadoop.assets%5C20180430221233503)

1. **客户端提交MR应用到Yarn的RM。**RM主要对资源进行管理和分配，并监控ApplicationMaster的运行情况，不负责具体task的计算。如果AM运行失败，会重新分配资源（container），重启AM。
2. **RM选取一个NM，并在NM中分配一个container。**这个NM在container中新建一个AM，并于RM通信。
3. **AM创建后，立即在RM中完成注册，使得RM可以监控AM的运行情况。**AM开始运行MR应用，并为各个task向RM申请其他的NM。
4. **AM收到RM分配的NM，并与他们完成通信。**
5. **AM请求新分配的NM使用他们的container完成各个task的计算任务。**AM在task计算过程中，始终监控他们的运行情况，并与RM通信。

当应用程序完成时，AM会像RM申请注销自己。



#### 4、MapReduce编程流程





### 集群资源管理器——YARN

Hadoop 2.x引入了新的框架YARN(Yet Another Resource Nagotiator)，负责集群资源管理和任务调度。所谓资源管理，就是按照一定的策略将资源（内存，CPU）分配给各个应用程序使用，并采取一定的隔离机制防止应用程序之间彼此抢占资源而相互干扰。

由于YARN的通用性，YARN可以作为其他高层分布式计算框架的资源管理系统，比如Spark、Storm、MapReduce等。



#### 1、YARN的基本架构及组件

Yarn整体还是输入master/slave模型，主要依赖三个组件来实现功能：**ResourceManager**、**NodeManager**和**ApplicationMaster**。ResourceManager管理整个集群的资源分配和任务调度，NodeManager管理单个节点的container和task，ApplicationMaster管理单个应用程序的状态。

<img src="https://img2018.cnblogs.com/blog/1271254/201910/1271254-20191008161506376-61248447.png" alt="img" style="zoom:80%;" />

##### 1.1 ResourceManager

ResourceManager拥有系统所有资源分配的决定权，负责**集群**中所有应用程序的资源分配。它的主要职责是：

- 接收来自Client的请求
- 启动和管理各个应用程序的ApplicationMaster
- 接收来自ApplicationMaster的资源申请，并为其分配Container
- 管理NodeManager，接收来自NodeManager的资源和节点健康情况汇报

主要关注竞争的应用程序之间分配系统中的可用资源，并不关注每个应用程序的状态管理。可主要分为两个组件：**Scheduler**和**ApplicationManager**。Scheduler是一个纯调度器，只负责调度Container，不关心应用程序的监控及运行状态等信息。ApplicationManager主要负责接收job提交的请求，为应用分配第一个Container来运行ApplicationMaster并监控。



##### 1.2 NodeManager

集群中每个**节点**上都有一个NodeManager，用来管理该节点上的资源和任务。主要负责：

- 接收RM的请求，分配Container给应用的某个任务
- 和RM交换信息以确保整个集群平稳运行。RM通过收集每个NM的报告信息来追踪整个集群的健康状态，而NM负责监控自身的健康状态。
- 管理每个Container的生命周期
- 管理每个节点的日志
- 执行Yarn上应用的一些额外任务，比如MapReduce的shuffle过程

Container是Yarn框架的计算单元，是具体执行应用task（如map task、reduce task）的基本单位。一个节点会运行多个Container，但一个Container不会跨节点。Container由NodeManager监控，ResourceManager调度。



##### 1.3 ApplicationMaster

ApplicationMaster主要负责**应用程序**的管理，它为应用向ResourceManager申请资源，并将资源分配给所管理的应用程序的task。

每个应用程序都对应一个ApplicationMaster，它负责：

- 与ResourceManager的Scheduler协商分配合适的Container，
- 和NodeManager协同工作来跟踪应用程序的状态，监控它们的进度。



#### 2、YARN作业的调度流程

![image-20200919153110561](D:%5CJob%5Clearn-BigData%5CHadoop%5CHadoop.assets%5Cimage-20200919153110561.png)

1）Client提交作业到YARN上

2）ResourceManager选择一个NodeManager，启动一个Container并运行一个ApplicationMaster实例

3）ApplicationMaster根据应用程序需要向ResourceManager请求更多的Container资源

4）ApplicationMaster通过获得的Container资源进行分布式计算



#### 3、YARN工作原理详述

