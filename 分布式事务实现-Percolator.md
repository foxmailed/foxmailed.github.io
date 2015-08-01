
Google为了解决网页索引的增量处理，以及维护数据表和索引表的一致性问题，基于BigTable实现了一个支持分布式事务的存储系统。这里重点讨论这个系统的分布式事务实现，不讨论percolator中为了支持增量计算而实现的Notifications机制。


该系统基于BigTable，支持[Snapshot Isolation](https://en.wikipedia.org/wiki/Snapshot_isolation)隔离级别，这个隔离级别不在ANSI定义的隔离级别范围内。简单来说，就是一个事务看到的是一个stable的数据库的快照。快照隔离相对于可串行化隔离级别的优点是更高的读性能，不需要加锁，MVCC基于BigTable的多版本机制。缺点是有write skew问题，简单来说，对于两个事务T1：b=a+1和T2:a=b+1，初始化a=b=0。串行化的情况下，结果只可能是(a=2,b=1)或者(a=1,b=2)，而在快照隔离级别下，结果可能是(a=1,b=1)。这在某些业务场景下是不能接受的。既然有多版本，就需要有版本号，percolator系统使用一个全局递增时间戳服务器来为事务产生时间戳，每个事务开始时拿一个时间戳t1，那么这个事务执行过程中可以读t1之前的数据。提交时再取一下时间戳t2，作为这个事务的提交时间戳。


现在说分布式事务。说起分布式事务第一个想到的就是两阶段提交，这个系统也不例外。客户端作为协调者coordinator，BigTable的tablet server作为参与者participant。 除了实际的表的每个Cell的数据存在BigTable中外，coordinator还将Cell锁信息，事务版本号存在BigTable中。简单来说，如果需要写列C，在BigTable中实际存在三列，分别为C:data,C:lock,C:write。由于BigTable实际上定位一个Value需要三个信息，rowkey，column和timestamp，所以实际上一个 column本身内部可以看成一个timestamp->value的map。那么：

　　1. c:write中存事务提交时间戳commit_ts=>start_ts。   

　　2. c:data这个map中存事务开始时间戳start_ts=>实际列数据

　　3. c:lock存start_ts=>(primary cell)，primary cell是rowkey和列名的组合，它在两阶段提交容错处理和事务冲突时使用，用来清理由于coordinator失败导致的分布式事务失败留下的锁信息。


举个没有任何冲突例子，假设一个分布式事务T1需要修改两个Cell，C1(Rowkey1:C1)和C2(Rowkey2:C2)，C1为primary cell，Value分别为Value1和Value2，并且两个Cell处于不同的tablet server，serverA和serverB。客户端commit之前首先将两个Cell都加入到客户端本地的一个数组中，最后事务commit(包括两阶段的prepare和commit)的时候才将所有Cell发向tablet server。


###没有检测到冲突的写事务流程：

prepare阶段：

      1. 分布式事务T1启动，从全局时间戳服务器获取事务启动时间戳记作t1_start_ts。
      2. 首先写primary cell C1，往C1:data中写入t1_start_ts=>value1，往C1:lock中写入t1_start_ts=>primary cell 表示加锁，同理，写serverB，往C2:data中写入t1_start_ts=>value2, 往C2:lock中写入t1_start_ts=>primary cell

commit阶段：

	1. 从全局时间戳服务器获取事务提交时间戳记作t1_commit_ts。 
	2. 启动一个C1所在的BigTable行事务，包含以下两个操作
		2.1 往primary cell C1:write写入t1_commit_ts=>t1_start_ts(这步是关键)
		2.2 将primary cell的lock release(delete C1:lock，时间戳为t1_start_ts)

	3. Commit这个BigTable 事务，这一步实际上将这个事务的数据对外可见，因为随后的一个读事务(事务启动时间戳记作t2_start_ts)读C1之前，会首先读C1:write的小于t2_start_ts的最大的版本的数据获得t1_start_ts，然后拿着t1_start_ts才能去C1:data中读取真正的数据。
	4. 将其他secondary cell C2:write中写入t1_commit_ts=>t1_start_ts，release C2的lock

###没有检测到冲突的读C1和C2的事务T3流程：
	
	1. 从全局时间戳服务器获取事务提交时间戳记作t3_start_ts
	2. 分别读C1和C2，读C:write的比t3_start_ts小的最大的一个事务提交时间戳的事务启动时间，然后拿这个事务启动时间去C:data中读真正的数据。

可以看出，一个Cell对外可见是通过写C:write来达到的，t1_commit_ts为事务提交版本号，t1_start_ts为t1这个事务修改后的数据版本号，真正读数据需要拿到t1_start_ts，而读t1_start_ts又需要首先拿到t1_commit_ts。

###协调者(Client)宕机容错

   假设C1上锁失败，C2上锁成功，那么分布式事务失败会将C2的锁残留在BigTable中。这个残留的锁由后续第一个读或写C2的事务来清除，满足什么样的条件才能清除？满足以下两个条件中的一个即可：1. 写这个lock的客户端在chubby上的结点没了，即客户端死了 2. C2:lock这把锁滞留时间太长了(lock内部保存最后更新时间即可)。  cleanup的操作就是直接delete C2:lock即可，时间戳为t2_start_ts_(percolator论文中此处有笔误)。但是如何知道残留下这个锁的事务是否已经提交？这就需要去读C2的primary cell的write字段，在这个例子里就是读C1:write，记残留下来的锁C2:lock的时间戳为lock_ts(percolator论文这里说的不清楚)，那么具体的判断事务是否提交的操作就是读取C1:write的[lock_ts,正无穷)的所有版本，判断是否有一个版本的值是lock_ts，如果有，则说明残留锁的事务已经提交。    

###事务冲突


Prepare阶段冲突：写C1之前需要首先读取C1:lock，如果有任何一个版本被加上了锁，那么这次分布式事务失败。还有一种冲突是，有其他事务在本事务开始之后commit修改了C1，从而修改了C1:write，这是一种Snapshot isolation需要避免的写写冲突。


Commit阶段冲突：分布式事务提交需要先提交primary cell，再提交其他cell，再提交primary cell时需要先检查自己是否还拿住了primary cell的锁，在这里是C1:lock，即t1_start_ts_版本是否已经被删除。做这个判断的原因是其他事务可以cleanup这个lock，如果它认为这个事务握有锁时间过长或者写入lock的客户端宕机太慢等原因。在这里，primary cell的lock字段是其他事务进行cleanup操作和当前事务提交操作的同步点。

 

###参考资料

[Large-scale Incremental Processing
Using Distributed Transactions and Notifications](http://static.googleusercontent.com/media/research.google.com/en/us/pubs/archive/36726.pdf)

[themis](https://github.com/XiaoMi/themis/)

 