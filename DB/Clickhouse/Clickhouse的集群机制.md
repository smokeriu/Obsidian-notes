Clickhouse依赖分片和副本，提供了集群的概念，与Hadoop等不同的是，Clickhouse并不是主从架构的，Clickhouse的集群工作更多是针对逻辑层面的。

# 副本和分片

假设Clickhouse集群有N个节点（N1,N2...），每个节点都有一张表A。如果N1的表A和N2的表A中的数据：

- 完全不同，则**N1,N2互为分片**。
- 完全相同，则**N1,N2互为副本**。

# 副本

副本主要用途是保证数据安全性，避免数据丢失。在MergeTree中，通过添加`Replicated`前缀，其实就是具有副本能力的MergeTree。

> 除了这种方式，也有另外一种提供副本的实现方式。

ReplicatedMergeTree逻辑关系如图所示：

在MergeTree中，数据分区的从创建到完成会尽力两个区域

1. 数据会先被写入内存中。
2. 数据会写入tmp临时目录分区，待数据全部完成后，再将临时目录**重命名**为正式分区。

Replicated在上述基础上增加了Zookeeper部分，会在Zookeeper中创建一系列监听节点，实现多个实例之间的通信。

> 通信过程中，Zookeeper不会负责表数据的传输。

关于Replicated，有以下特点：

- 写时依赖Zookeeper：在执行Insert和Alter操作时，依赖Zookeeper实现多节点的副本同步。**查询时不依赖**。
    
- Zookeeper不负责数据传输。
    
- 表级别副本：副本数量/分布位置是表级定义的。
    
- 多主架构：Clickhouse不是主从架构，在任意一个副本上执行INSERT效果相同。
    
- Block数据块：`max_insert_block_size`是数据写入的基本单元。
    
    - 原子性：一个`Block`内的数据要么全部成功，要么全部失败。
        
    - 唯一性：避免`Block`级别的重复写入。会更具Block的信息生成Hash记录，忽略相同Hash记录的`Block`。
        

## Zookeeper

分布式协同能力依赖于Zookeeper，故需要为Server配置连接的Zookeeper。这部分请参考[官方配置](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/replication/)。

> 在新版本中，clickhouse提供了clickhouse-keeper，来替代zookeeper，参考[clickhouse-keeper](https://clickhouse.com/clickhouse/keeper)和[clickhouse-keeper和zookeeper](https://clickhouse.com/docs/en/operations/tips#zookeeper)。
> 本文仍然基于zookeeper，更多内容参考[[Clickhouse-keeper]]。

## 定义副本

引擎选择Replicated时，会有专门的配置项用于配置副本相关的内容。

 ENGINE = ReplicatedMergeTree('zk_path','replica_name')

- `zk_path` - 元数据在zk中的地址。
    
    - 官方推荐写法：`/clickhouse/tables/{shards}/table_name`
        
        - `shards` - 即分片编号。一般采用数字`01,02,03`。一张表可以有多个分片，每个分片有自己的副本。

- `replica_name` - zk中的副本名称。可以设置成服务器的域名称。

## 目录结构

在理解副本的工作原理前，需要熟悉Zookeeper中各个节点的用途。这些节点可以分为：

- 元数据：
    
    - `/metadata` - 元数据信息，如主键、分区键等。
        
    - `/columns` - 列字段信息，如字段名称、字段类型。
        
    - `/replicas` - 副本名称。对应`replica_name`。
        
- 判断标识：
    
    - `/leader_election` - 主副本选举。`Merge`、`Delete`和`Update`操作会先在主副本上完成后，在同步到其他副本。
        
    - `/blocks` - 记录数据块的`Hash`摘要，以及对应的`partition_id`信息。
        
        - `Hash`摘要：判断Block是否重复。
            
        - `partition_id`信息：快速找到需要同步的数据分区。
            
    - `/block_numbers` - 按照分区的写入顺序，以相同的顺序记录`partition_id`。副本在本地merge时，会依照相同的`block_numbers`顺序进行。
        
    - `/quorum` - 记录`quorum`数量。当至少有参数`insert_quorum`数量副本写入成功时，才选操作成功。
        

> 上面这些暂时有个印象即可。

- 操作日志：
    
    - `/log` - 常规操作的日子记录：`INSERT`、`MERGE`和`DROP PARTITION`。
        
        - 副本节点会监听这些log节点，当有新的指令加入时，他会把指令驾到各自的任务队列中执行。
            
        - _每条指令_是从0开始的递增：`/log/log-000...000, /log/log-000...001`。
            
    - `/mutations` - `MUTATION`操作的日志节点：`ALTER DELETE`和`ALTER UPDATE`。
        
        - _每条指令_是从0开始的递增：`/mutations/000...000, /mutations/000...001`。
            
    - `/replicas/{replica_name}/*` - 各副本各自的节点下的监听节点。用于指导副本在本地执行具体的任务指令。
        
        - `/queue` - 任务队列节点。当监听的`/log`和`/mutations`出现变化时，执行任务会加入该节点。
            
        - `/log_pointer` - `/log`的指针节点。记录最后一次执行log日志的下标信息。
            
            - `log_pointer: 4`时，对应了`/log/log-0...03`。
                
        - `/mutation_pointer` - `/mutations`的指针节点。记录最后一次执行mutations日志的下标信息。
            
            - `mutation_pointer: 000000`时，对应了`/mutations/000000`。
                

为`/log`和`/mutations`添加子节点，其实就起到了发送指令的目的。所有副本实例都会监控这两个节点。当子节点被添加时就能被感应到。这一类子节点被统一抽象为Entry。

> 注意到指针节点是存储在副本的节点上的，即每个副本由于业务繁忙问题，可能存储不同的指针。但如果停止数据插入，最终，他们会趋于一致。

## Entry对象

LogEntry对应了`/log`节点，而MutationEntry则对应了`/mutations`节点。

### LogEntry

用于封装`/log`的子节点信息。拥有下述熟悉：

- `source replica` - 这条log来自于哪个副本。即`replica_name`
    
- `type` - 指令类型，主要有：
    
    - `get` - 从远程副本下载分区。
        
    - `merge` - 合并分区
        
    - `mutate` - MUTATION操作
        
- `block_id` - 当前分区的BlockID，对应于`/blocks`路径下子节点的名称。
    
- `partition_name` - 当前分区目录的名称。
    

### MutationEntry

用于封装`/mutations`的子节点信息。拥有下述熟悉：

- `source replica` - 这条log来自于哪个副本。即`replica_name`
    
- `commands` - 操作指令。主要有Update和Delete。
    
- `mutation_id` - 操作版本号。
    
- `partition_name` - 当前分区目录的名称。
    

## 副本协同核心流程

在大致了解Zookeeper的组成后，可以开始理解副本协同流程。核心流程主要有：INSERT、MERGE、MUTATION和ALTER四种，对应着数据插入，分区合并，数据修改，元数据修改。

其中，INSERT和ALTER查询时分布式执行的，借助ZK，多副本之间会自动协同，但他们不会使用ZK存储任何分区数据。

而其他的，包括SELECT、CREATE、DROP、RENAME和ATTACH则不支持分布式执行，所以现阶段建表需要在各个节点都建。

现假设我们有两个节点，拥有1分片1副本的情况

### INSERT

1. 在A，B节点分别创建一个副本实例，即建表。区别是`replica_name`不一样。此时会进行下述操作：
    
    1. 在`/replicas`节点下注册自己的副本实例`{replica_name}`
        
    2. 启动监听任务，监听`/log`节点。
        
    3. 参与副本选举，向`/leader_election`插入子节点。第一个插入成功的会成为主副本。
        
2. 向A实例中写入数据。此时会触发下述操作：（向B中写入是一样的）
    
    1. 在本地创建新分区。
        
    2. 想`/blocks`下创建子节点`block_id` : `/blocks/20211116_xxx_yyy`，即`分区名称_计算值`。如果一组数据，拥有同样的计算结果，则不会插入。值得注意的是，`block_id`是基于一次插入计算得到的。一次插入可以插入多条数据。一次插入的所有数据都一样，这会有一样的`block_id`。这意味着：
        
        1. 如果我们两次插入一条一摸一样的数据，则第二次插入会被忽略。即表中只会有一条。
            
        2. 但如果我们第二次插入两条数据（数据内容与第一次一样），也会有不同的`block_id`。即表中会有3条数据。
            
        3. 上述两种情况的两次可以隔很远，并不是说要相邻。
            
3. 副本实例推送Log日志。
    
    1. 此时会在`/log`下创建子节点：`/log/log-000...000`，并在`log-000...000`节点中写入数据。
        
    2. 通过`get`命令，可以看到这个节点下的数据。
        
        1. 其中`source replica`指向了操作的来源。即第二步中写入数据的实例名称。
            
        2. `block_id`反应了这条数据对应的哪条数据。
            
        3. `type`是get类型
            
        4. 结合其他信息，就能够得到这次插入的数据在哪，其他节点应该去哪取数据。
            
4. 其余副本拉取Log日志。
    
    1. 副本自身存储的了自身的`/log_pointer` ，其节点值记录了该副本执行到了第几个`/log`下子节点。
        
    2. 如果有超过指针的新节点，这会讲LogEntry拉取到自己副本的`/queue`中，排队执行。
        
5. 副本向其他副本发送下载请求。（以B为例）
    
    1. 当`/queue`等到本例的LogEntry时，由于Type时get类型。则会向其他副本发送下载请求，选择逻辑：
        
        1. 从`/replicas`节点拿到所有副本节点，本例有两个`{replica_name}`。
            
        2. 选择其中`log_pointer`下标最大，且`/queue`子节点数量最少的`{replica_name}`
            
            1. `log_pointer`下标大意味着**数据最完整**。
                
                1. 这个选择规则意味着总是同步最老的需要同步的数据。新数据需要等老数据同步完，`log_pointer`才会往后移。
                    
            2. `/queue`子节点数量最少意味着**负担最小**。
                
    2. 选择好节点后，向该节点发起HTTP请求。下载数据。
        
6. 响应下载。
    
7. 本地写入。
    
    1. 数据会先写入临时目录下：`tmp_fetch_partitionId`。
        
    2. 写入完成后，重命名为分区本来的目录名称。
        

### MERGE

无论MERGE操作从哪个副本发起，其合并计划都会交由主副本来制定。这样能够避免不同副本协同进度不一造成的不统一问题。

我们假定操作由非主副本B发起。

1. B尝试触发Merge，建立与主副本的通信。
    
2. 主副本接受通信。
    
3. 主副本制定MERGE计划，并向`/log`节点下创建对应的Log日志。
    
    1. 主副本判断哪些分区需要合并。这些信息会被记录在Log日志中。包含了：要合并哪些分区，最后生成什么分区。
        
    2. 此时`LogEntry`中的`type`为`merge`。
        
4. 各节点拉取Log日志，并加入自身的`/queue`中。
    
5. 各节点完成合并。
    

### MUTATION

Mutation在Clickhouse中指`UPDATE`和`DELETE`操作。与MERGE类似，无论Mutation操作从哪个副本发起，其合并计划都会交由主副本来制定。

我们假定操作由非主副本B发起。

1. 我们在B副本上尝试删除一条数据。主要会发生两件事：
    
    1. 创建`MUTATION ID`
        
    2. 操作转换为`MutationEntry`日志，并推送到`/mutations/00000000`下。
        
2. 所有副本均会监听`/mutations`节点。但自由主副本会做出响应。
    
3. 主副本响应Mutation日志，并推送Log日志。
    
    1. 主副本将`MutationEntry`日志转换为`LogEntry`日志，并推送到`/log`下。
        
    2. 此时`LogEntry`中的`type`为`mutate`。（也可能为其他的，如drop）。
        
4. 各副本拉取Log日志，并执行。
    
    1. 各副本根据Log日志的情况，执行mutate操作。
        
    2. 具体执行时，会根据`LogEntry`，来判断需要根据哪个`MutationEntry`来执行。
        

### ALTER

这里的ALTER指的是表的元数据的修改，如增加、删除表字段。不包含`ALTER UPDATE`和`ALTER DELETE`这类数据修改。

1. 修改共享元数据。
    
    1. 在一个节点执行ALTER操作后，会修改ZK上的`/metadata`和`/columns`节点。
        
    2. 提升节点版本号。
        
2. 所有节点监听元数据变更，并自己执行本地修改。
    
    1. 对比版本号，根据ZK的元数据，修改本地元数据。
        
3. 确认所有副本完成修改。
    

# 分片

想要启用分片，需要在配置文件中进行配置。一般来说，与刚刚讲到的副本不同，刚刚我们控制副本个数的方法是在 N 台机器上运行建表语句，并将PATH指向同一个ZK节点。这种情况，其实相当于我们只有一个分片。

实际工作中，我们会利用`ON CLUSTER {cluster_name}`的方式，执行分布式DDL。而`{cluster_name}`决定了副本和分片的设置。具体的配置请参考官网，简单来说，我们需要为一个`{cluster_name}`配置他的分片，为每个分片配置副本属性。这些属性需要在每个节点保持一致

同时为了方便实用，还可以为每台机器配置各自的`<macros>`属性：

 <macros>  
   <shard>01</shard>  
   <replica>ssiu01</replica>  
 </macros>

这么做的好处是我们可以在使用分布式语句时，可以通过宏的方式，使得不同节点指向自己的ZK节点：

 ```sql
 CREATE TABLE {table_name} ON CLUSTER {cluster_name}   
 (...)  
 ENGINE = ReplicatedMergeTree('/path/to/tables/{shard}/table_name','{replica}')  
 ORDER BY id;
```

上面使用的`{shard}`和`{replica}`，就对应了这个配置属性，这使得每台机器会根据配置，监听自己负责{shard}和{replica}。

## 数据结构

默认情况下，分布式DDL会使用ZK的节点：

 <distributed_ddl>  
   <path>/clickhouse/task_queue/ddl</path>  
 </distributed_ddl>

这个节点下也会有一些其他的监听节点，最主要的一个是`/query-[seq]`。每次分布式DDL都会在该节点新增一条操作日志。即[seq]会一直递增。

`/query-[seq]`记录的日志信息由DDLLogEntry承载，他拥有如下几个核心属性：

- DDL查询语句。
    
- hosts记录了哪些主机需要执行该DDL。
    
- initiator记录了初始化该DDL的主机。
    

`/query-[seq]`下有两个状态节点：

- `/query-[seq]/active` - 在任务执行过程中，该节点会临时保存当前集群内，状态为active的节点。
    
- `/query-[seq]/finished` - 每当某个host节点执行完毕，便会在该节点写写入记录。
    

> 实际测试中发现，这两个节点存在，但其中并没有书中提及的相应数据。暂时先忽略这个问题。

## 执行流程

DDL的执行流程相对简单。初始化节点向`clickhouse/task_queue/ddl`下推送DDLLogEntry。其余节点监控该日志，通过`hosts`和`initiator`来判断本机是否需要执行。

`/query-[seq]/finished`则用于监控哪些节点已经完成执行。保证所有节点元数据统一。

# 分布式表

截止目前，我们已经创建好带分片和副本的数据表。

> 假设我们集群拥有4个节点2个分片，其中分片1在机器01和03上有副本，分片2在机器02和04上有副本。

如果我们直接向机器01的实例写入数据，就会发现数据仅仅出现在01/03这两台机器上，因为他们互为副本，而多分片的效果并没有体现，这就要引入分布式表。

在Clickhouse中，按照分布式来区分的话，有两类表：

- 本地表：通常以`_local`进行命名。
    
    - 本地表是承载数据的载体，可以使用非`Distributed`的任意表引擎。
        
    - 一张本地表对应了一个数据分片。
        
- 分布式表：通常以`_all`进行命名。
    
    - 只能使用`Distributed`表引擎。本身不存储数据。
        
    - 他与本地表是一种映射关系。通过分布式表，可以操作多张本地表。
        

## 定义

Distributed也是一类表引擎，所以和一般建表一样：

 ENGINE = Distributed(cluster , database, table, [,sharding_key[, policy_name]])

> 创建Distributed时也应当使用`ON CLUSTER`关键字，否则分布式表仅仅能在单个节点使用（读/写）。

前三组参数不需要解释，

- `sharding_key`是一个选填参数，可以翻译成分片键，也可以理解为分布键。
    
    - 在往分布式表中插入数据时，会按照`sharding_key`和`config.xml`中配置的分片权重(weight)一同决定写入分布式表时的路由, 即数据最终落到哪个物理表上。权重越大，写入其中的数据越多。
        
    - 可以是表中一列的原始数据(如`site_id`)，也可以是函数调用的结果, 如采用了随机值`rand()`。
        
    - 该键要尽量保证数据均匀分布，一个常用的操作是采用区分度较高的列的哈希值，如`intHash64(user_id)`。
        
- `plicy_name` - 用于决定中间数据存储在哪。一般需要配合[storage_configuration](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/mergetree/#table_engine-mergetree-multiple-volumes_configure)使用。
    

Distributed表引擎提供了一组设置。下面的只是分类大致介绍，具体的情况可参见官网[setting](https://clickhouse.com/docs/en/engines/table-engines/special/distributed/)。

- `fsync_*` - 默认不开启，不符合本分片的数据会暂存在本地，再根据规则异步分布到其他分片中。如果开启，则会等待所有分片数据插入才算成功。
    
- `bytes_*` - 默认不开启，当单次插入过多数据时会导致异常或延迟。
    

Distributed表引擎和Hive类似，是读时检查的机制，即建表时如果Distributed表的结构和`_local`表结构不一样，则不会报错，仅当无法完成读取时才会报错（这意味着其实两者表结构可以不完全一致，但并不建议这么做。）

## 写入

写入流程如图所示：

> 注：我测试版本存储目录为: `/{table_name}/shard{x}_replica{y}/[increase_num].bin`

以我的版本为例，简单来说，经由分片算法得到的属于本机数据，会直接写入本机的local表中，而不属于本机的数据，则会写入本机的临时目录，按照数据所属的`分片_副本`为目录进行存放。所以每台设备的临时目录下，会有`分区*副本-1`个目录。

而本机会有一个监听任务，去监控这个文件夹的变化，一旦有新文件，便会启动向其他机器发送数据的任务。而数据发送到所在机器后，还会尽力分区重命名的步骤。

### 关于副本

副本的数据传输有两种模式，一种便是通过`ReplicatedMergeTree`自身的机制进行数据传输。而另一种则是通过Distributed进行复制。本节写入流程中的流程说明的便是了这种形式。

不过这种形式由于副本的传输也是有连接的节点负责，会造成网络IO瓶颈，所以可以利用`ReplicatedMergeTree`的机制，让Distributed表只负责分片的传输，分片自己负责副本的传输。采用这种方式，需要在shard上添加配置`<internal_replication>true</internal_replicatio>`。

最终配置如下：

 <shard>  
   <internal_replication>true</internal_replication>  
   <replica>  
     <!--replica1-->  
   </replica>  
   <replica>  
     <!--replica2-->  
   </replica>  
 </shard>

## 查询

当查询Distributed时，引擎会一次查询每个分片的数据，然后汇总结果返回。

### 副本路由规则

首先，当分片存在副本时，会涉及到副本的选择问题。由`load_balancing`控制：

> 背景知识：节点内部有一个`errors_count`计数，服务器发生宕机此数会`+1`

- `random` - clickhouse会在`errors_count`最小的副本中随机选择一个。
    
- `nearest_hostname` - 在选择`errors_count`最小的副本的基础上，对host名称逐位比较，选择相差最小的。
    
- `in_order` - 在选择`errors_count`最小的副本的基础上，根据配置的`replica`定义顺序选择。
    
- `first_or_random` - 在选择`errors_count`最小的副本的基础上，选择配置的`replica`定义中最靠前的。如果最靠前的不可用，则在剩余里随机选择。
    

### 多分片查询

提高查询性能最好的手段就是分片查询，即将查询分布到各个节点，分布计算压力。

在不涉及聚合/联合查询时，这种查询很简单， 只要将查询条件中的分布表替换成local表即可。

#### 聚合查询

聚合查询会直接下发到各个节点，首先在远程服务器聚合，之后返回聚合函数的中间状态给查询请求的服务器。再在请求的服务器上进一步汇总数据。

例如查询如下：

```sql
SELECT name,avg(price) FROM info_all WHERE info = 'info1' GROUP BY name
```

对于avg来说，我们不能直接只返回一个avg结果，而应该返回`id,sum(price),count(price)`组成的结果对，再在本地完成结果的聚合。具体返回什么数据由函数决定。不过远程服务器会完成绝大多数的聚合操作，保证数据查询的快速。

#### 联合查询

假设我们有这么一条SQL：

```sql
SELECT u.name,i.price  
    FROM hadoop.info_all i JOIN hadoop.user_all u  
ON i.id=u.id;
```

其实仅仅观察log，我们会发现右侧的分布式表的查询翻倍了，原因其实很简单，因为我们关联的右表也是一个分布式表，则其实Clickhouse内部的转换可以看成：

```sql
SELECT u.name,i.price  
    FROM hadoop.info_all i JOIN hadoop.user_all u  
ON i.id=u.id;  
```
 ​  
- 将主表替换为本地表,推送给远端  
 ​  
```sql
SELECT u.name,i.price  
    FROM hadoop.info_local i JOIN hadoop.user_all u  
ON i.id=u.id;  
```
 ​  
 - 远端机器将子句的分布式表，转换成local表。向其他机器发送查询请求  
 ​  
```sql
SELECT id,name FROM user_local
``` 
 ​  
 - 得到sub_query后，再结合本机的local查询结果，得到最终结果，最后汇总  
 ​  
```sql
SELECT u.name,i.price  
    FROM hadoop.info_local i JOIN sub_query u  
ON i.id=u.id;
```

所以，查询请求被放大了N\*N倍。而类似于大数据中常见的`hash-join`，这部分在某些情况其实不需要让每台机器重复执行相同的运算，我们可以使用`GLOBAL`关键字，将子查询转换为一次查询。

```sql
SELECT u.name,i.price  
    FROM hadoop.info_all i GLOBAL JOIN hadoop.user_all u  
ON i.id=u.id;
```

这样，对右表user_all的查询会被精简到一次，将查询的最终结果**全部**加载到内存中，分发到各个各个节点。

从这里也可以看出，GLOBAL JOIN的查询会被全部加载到内存中并分发到各个节点。如果右表过大，会造成大量的资源损耗。故GLOBAL JOIN也**只适合右表是小表**的场景。

如果联合查询时两张表都是大表怎么办，目前只能根据Clickhouse的特性，对右表进行设计。

创建分布式表时，Clickhouse会根据`sharding_key`来进行数据的分配，一个设计准则就是让右表的查询不发往其他节点。例如当我们join条件是`a.id=b.id`时，如果b表的id作为sharding_key。保证一个id只存在于一个分片上，就可以避免将查询N\*N倍的放大。