
# 数据库shard
## 一、基本思想
Sharding的基本思想就要把一个数据库切分成多个部分放到不同的数据库(server)上，从而缓解单一数据库的性能问题。不太严格的讲，对于海量数据的数据库，如果是因为表多而数据多，这时候适合使用垂直切分，即把关系紧密（比如同一模块）的表切分出来放在一个server上。如果表并不多，但每张表的数据非常多，这时候适合水平切分，即把表的数据按某种规则（比如按ID散列）切分到多个数据库(server)上。当然，现实中更多是这两种情况混杂在一起，这时候需要根据实际情况做出选择，也可能会综合使用垂直与水平切分，从而将原有数据库切分成类似矩阵一样可以无限扩充的数据库(server)阵列。
* 垂直切分
垂直切分的最大特点就是规则简单，实施也更为方便，尤其适合各业务之间的耦合度非
常低，相互影响很小，业务逻辑非常清晰的系统。在这种系统中，可以很容易做到将不同业
务模块所使用的表分拆到不同的数据库中。
![enter image description here](http://hi.csdn.net/attachment/201101/24/0_12958577041KqK.gif)
* 水平切分
水平切分于垂直切分相比，相对来说稍微复杂一些。因为要将同一个表中的不同数据拆
分到不同的数据库中，对于应用程序来说，拆分规则本身就较根据表名来拆分更为复杂，后
期的数据维护也会更为复杂一些。
![enter image description here](http://hi.csdn.net/attachment/201101/24/0_1295857710BUth.gif)

##　二、详细说明
![enter image description here](http://ww1.sinaimg.cn/large/67a6a651gw1ducq4lmzyzj.jpg)

##　三、主键生成策略
一旦数据库被切分到多个物理结点上，我们将不能再依赖数据库自身的主键生成机制。一方面，某个分区数据库自生成的ID无法保证在全局上是唯一的；另一方面，应用程序在插入数据之前需要先获得ID,以便进行SQL路由。目前几种可行的主键生成策略有：
1. UUID：使用UUID作主键是最简单的方案，但是缺点也是非常明显的。由于UUID非常的长，除占用大量存储空间外，最主要的问题是在索引上，在建立索引和基于索引进行查询时都存在性能问题。
2. 结合数据库维护一个Sequence表：此方案的思路也很简单，在数据库中建立一个Sequence表，表的结构类似于：
[sql] view plaincopy在CODE上查看代码片派生到我的代码片
``` sql
CREATE TABLE `SEQUENCE` (  
    `tablename` varchar(30) NOT NULL,  
    `nextid` bigint(20) NOT NULL,  
    PRIMARY KEY (`tablename`)  
) ENGINE=InnoDB   
```
每当需要为某个表的新纪录生成ID时就从Sequence表中取出对应表的nextid,并将nextid的值加1后更新到数据库中以备下次使用。此方案也较简单，但缺点同样明显：由于所有插入任何都需要访问该表，该表很容易成为系统性能瓶颈，同时它也存在单点问题，一旦该表数据库失效，整个应用程序将无法工作。有人提出使用Master-Slave进行主从同步，但这也只能解决单点问题，并不能解决读写比为1:1的访问压力问题。
3.flickr开发团队在2010年撰文介绍了flickr使用的一种主键生成测策略，同时表示该方案在flickr上的实际运行效果也非常令人满意，原文连接：Ticket Servers: Distributed Unique Primary Keys on the Cheap 这个方案是我目前知道的最好的方案，它与一般Sequence表方案有些类似，但却很好地解决了性能瓶颈和单点问题，是一种非常可靠而高效的全局主键生成方案。
![enter image description here](http://ww1.sinaimg.cn/large/67a6a651gw1dujgqcx9ncj.jpg)

flickr这一方案的整体思想是：建立两台以上的数据库ID生成服务器，每个服务器都有一张记录各表当前ID的Sequence表，但是Sequence中ID增长的步长是服务器的数量，起始值依次错开，这样相当于把ID的生成散列到了每个服务器节点上。例如：如果我们设置两台数据库ID生成服务器，那么就让一台的Sequence表的ID起始值为1,每次增长步长为2,另一台的Sequence表的ID起始值为2,每次增长步长也为2，那么结果就是奇数的ID都将从第一台服务器上生成，偶数的ID都从第二台服务器上生成，这样就将生成ID的压力均匀分散到两台服务器上，同时配合应用程序的控制，当一个服务器失效后，系统能自动切换到另一个服务器上获取ID，从而保证了系统的容错。

关于这个方案，有几点细节这里再说明一下：
* flickr的数据库ID生成服务器是专用服务器，服务器上只有一个数据库，数据库中表都是用于生成Sequence的，这也是因为auto-increment-offset和auto-increment-increment这两个数据库变量是数据库实例级别的变量。
* flickr的方案中表格中的stub字段只是一个char(1) NOT NULL存根字段，并非表名，因此，一般来说，一个Sequence表只有一条纪录，可以同时为多张表生成ID，如果需要表的ID是有连续的，需要为该表单独建立Sequence表。
* 方案使用了mysql的LAST_INSERT_ID()函数，这也决定了Sequence表只能有一条记录。
* 使用REPLACE INTO插入数据，这是很讨巧的作法，主要是希望利用mysql自身的机制生成ID,不仅是因为这样简单，更是因为我们需要ID按照我们设定的方式(初值和步长)来生成。
* SELECT LAST_INSERT_ID()必须要于REPLACE INTO语句在同一个数据库连接下才能得到刚刚插入的新ID，否则返回的值总是0
* 该方案中Sequence表使用的是MyISAM引擎，以获取更高的性能，注意：MyISAM引擎使用的是表级别的锁，MyISAM对表的读写是串行的，因此不必担心在并发时两次读取会得到同一个ID(另外，应该程序也不需要同步，每个请求的线程都会得到一个新的connection,不存在需要同步的共享资源)。经过实际对比测试，使用一样的Sequence表进行ID生成，MyISAM引擎要比InnoDB表现高出很多！
* 可使用纯JDBC实现对Sequence表的操作，以便获得更高的效率，实验表明，即使只使用Spring JDBC性能也不及纯JDBC来得快！

实现该方案，应用程序同样需要做一些处理，主要是两方面的工作：

1. 自动均衡数据库ID生成服务器的访问
2. 确保在某个数据库ID生成服务器失效的情况下，能将请求转发到其他服务器上执行。

## 四、 分库分表
* 原理

首先，作为方案的基石，为了能使系统感知到Shard并基于Shard的分布进行路由计算，我们需要建立一个可以描述Sharding拓扑结构的编程模型。按照一般的切分原则，一个单一的数据库会首先进行垂直切分，垂直切分只是将关系密切的表划分在一起，我们把这样分出的一组表称为一个Partition。 接下来，如果Partition里的表数据量很大且增速迅猛，就再进行水平切分，水平切分会将一张表的数据按增量区间或散列方式分散到多个Shard上存储。在我们的方案里，我们使用增量区间与散列相结合的方式，全局上，数据按增量区间分布，但是每个增量区间并不是按照某个Shard的存储规模划分的，而是根据一组Shard的存储总量来确定的，我们把这样的一组Shard称为一个ShardGroup，局部上，也就是一个ShardGroup内，记录会再按散列方式均匀分布到组内各Shard上。这样，一条数据的路由会先根据其ID所处的区间确定ShardGroup，然后再通过散列命中ShardGroup内的某个目标Shard。在每次扩容时，我们会引入一组新的Shard，组成一个新的ShardGroup，为其分配增量区间并标记为“可写入”，同时将原有ShardGroup标记为“不可写入”，于是新生数据就会写入新的ShardGroup，旧有数据不需要迁移。同时，在ShardGroup内部各Shard之间使用散列方式分布数据读写，进而又避免了“热点”问题。最后，在Shard内部，当单表数据达到一定上限时，表的读写性能就开始大幅下滑，但是整个数据库并没有达到存储和负载的上限，为了充分发挥服务器的性能，我们通常会新建多张结构一样的表，并在新表上继续写入数据，我们把这样的表称为“分段表”（Fragment Table）。不过，引入分段表后所有的SQL在执行前都需要根据ID将其中的表名替换成真正的分段表名，这无疑增加了实现Sharding的难度，如果系统再使用了某种ORM框架，那么替换起来可能会更加困难。目前很多数据库提供一种与分段表类似的“分区”机制，但没有分段表的副作用，团队可以根据系统的实现情况在分段表和分区机制中灵活选择。总之，基于上述切分原理，我们将得到如下Sharding拓扑结构的领域模型：
![enter image description here](http://ww1.sinaimg.cn/large/67a6a651tw1dwtktxvkycj.jpg)

在这个模型中，有几个细节需要注意：ShardGroup的writable属性用于标识该ShardGroup是否可以写入数据，一个Partition在任何时候只能有一个ShardGroup是可写的，这个ShardGroup往往是最近一次扩容引入的；startId和endId属性用于标识该ShardGroup的ID增量区间；Shard的hashValue属性用于标识该Shard节点接受哪些散列值的数据；FragmentTable的startId和endId是用于标识该分段表储存数据的ID区间。
* 示例

让我们通过示例来了解这套方案是如何工作的。

* 阶段一：初始上线

假设某系统初始上线，规划为某表提供4000W条记录的存储能力，若单表存储上限为1000W条，单库存储上限为2000W条，共需2个Shard，每个Shard包含两个分段表，ShardGroup增量区间为0-4000W，按2取余分散到2个Shard上，具体规划方案如下：
![enter image description here](http://ww4.sinaimg.cn/large/67a6a651tw1dwtkua2zayj.jpg)

与之相适应，Sharding拓扑结构的元数据如下：
![enter image description here](http://ww3.sinaimg.cn/large/67a6a651tw1dwtkulpyalj.jpg)

* 阶段二：系统扩容

经过一段时间的运行，当原表总数据逼近4000W条上限时，系统就需要扩容了。为了演示方案的灵活性，我们假设现在有三台服务器Shard2、Shard3、Shard4，其性能和存储能力表现依次为Shard2<Shard3<Shard4，我们安排Shard2储存1000W条记录，Shard3储存2000W条记录，Shard4储存3000W条记录，这样，该表的总存储能力将由扩容前的4000W条提升到10000W条，以下是详细的规划方案：
![enter image description here](http://ww4.sinaimg.cn/large/67a6a651tw1dwtkusk6o1j.jpg)
图4. 二次扩容6000W存储规模的规划方案

相应拓扑结构表数据下：

![enter image description here](http://ww4.sinaimg.cn/large/67a6a651tw1dwtkuzlxeaj.jpg)
图5. 对应Sharding元数据

从这个扩容案例中我们可以看出该方案允许根据硬件情况进行灵活规划，对扩容规模和节点数量没有硬性规定，是一种非常自由的扩容方案。

* 增强

接下来让我们讨论一个高级话题：对“再生”存储空间的利用。对于大多数系统来说，历史数据较为稳定，被更新或是删除的概率并不高，反映到数据库上就是历史Shard的数据量基本保持恒定，但也不排除某些系统其数据有同等的删除概率，甚至是越老的数据被删除的可能性越大，这样反映到数据库上就是历史Shard随着时间的推移，数据量会持续下降，在经历了一段时间后，节点就会腾出很大一部分存储空间，我们把这样的存储空间叫“再生”存储空间，如何有效利用再生存储空间是这些系统在设计扩容方案时需要特别考虑的。回到我们的方案，实际上我们只需要在现有基础上进行一个简单的升级就可以实现对再生存储空间的利用，升级的关键就是将过去ShardGroup和FragmentTable的单一的ID区间提升为多重ID区间。为此我们把ShardGroup和FragmentTable的ID区间属性抽离出来，分别用ShardGroupInterval和FragmentTableIdInterval表示，并和它们保持一对多关系。
![enter image description here](http://ww1.sinaimg.cn/large/67a6a651tw1dwtlsrz0fij.jpg)

让我们还是通过一个示例来了解升级后的方案是如何工作的。

阶段三：不扩容，重复利用再生存储空间

假设系统又经过一段时间的运行之后，二次扩容的6000W条存储空间即将耗尽，但是由于系统自身的特点，早期的很多数据被删除，Shard0和Shard1又各自腾出了一半的存储空间，于是ShardGroup0总计有2000W条的存储空间可以重新利用。为此，我们重新将ShardGroup0标记为writable=true，并给它追加一段ID区间：10000W-12000W，进而得到如下规划方案：
![enter image description here](http://ww4.sinaimg.cn/large/67a6a651tw1dwtlt4c96yj.jpg)
图7. 重复利用2000W再生存储空间的规划方案

相应拓扑结构的元数据如下：
![enter image description here](http://ww4.sinaimg.cn/large/67a6a651tw1dwtltc5aknj.jpg)
图8. 对应Sharding元数据

小结

这套方案综合利用了增量区间和散列两种路由方式的优势，避免了数据迁移和“热点”问题，同时，它对Sharding拓扑结构建模，使用了一致的路由算法，从而避免了扩容时修改路由代码，是一种理想的Sharding扩容方案。