
Spanner要满足的external consistency(有些文章叫Linearizability或者strict serializability) 是指：后开始的事务一定可以看到先提交的事务的修改。所有事务的读写都加锁可以解决这个问题，缺点是性能较差。特别是对于一些workload中只读事务占比较大的系统来说不可接受。为了让只读事务不加任何锁，需要引入多版本。在单机系统中，维护一个递增的时间戳作为版本号很好办。分布式系统中，机器和机器之间的时钟有误差，并且误差范围不确定，带来的问题就是很难判断事件(在本文，事件指分布式事务版本号)发生的前后关系。反应在Spanner中，就是很难给事务赋予一个时间戳作为版本号，以满足external consistency。在这样一个误差范围不确定的分布式系统时，通常，获得两个事件发生的先后关系主要通过在节点之间进行通信分析其中的因果关系(casual relationship),经典算法包括Lamport时钟等算法。然而，Spanner采用不同的思路，通过在数据中心配备原子钟和GPS接收器来解决这个误差范围不确定的问题，进而解决分布式事务时序这个问题。基于此，Spanner提供了TrueTime API，返回值实际为一个区间[t-ε,t+ε]，ε为时间误差，毫秒级，保证当前的真实时间位于这个区间。

  Spanner是一个支持分布式读写事务，只读事务的分布式存储系统，只读事务不加任何锁。和其他分布式存储系统一样，通过维护多副本来提高系统的可用性。一份数据的多个副本组成一个paxos group，通过paxos协议维护副本之间的一致性。对于涉及到跨机的分布式事务，涉及到的每个paxos group中都会选出一个leader，来参与分布式事务的协调。这些个leader又会选出一个大leader，称为coordinator leader，作为两阶段提交的coordinator，记作coordinator leader。其他leader作为participant。

  数据库事务系统的核心挑战之一是并发控制协议。Spanner的读写事务使用两阶段锁来处理。分布式读写事务请求到达coordinator leader后，coordinator leader运行两阶段提交协议，将读写请求发给participant，pariticpant和coordinator leader开始加读，写锁。最后commit的时候，读写锁解除。

  如第一段所述，给事务赋予一个时间戳版本号是这样一个分布式存储系统的核心。下面先说如何确定读写事务的版本号，再说只读事务。

  前面已经说了两阶段提交过程中两阶段锁的过程，这里就省略这些，只讨论两阶段提交过程中如何确定最后的读写事务的时间戳版本号。

###读写事务 

  读写事务开始时，coordinator leader 首先调用TrueTime API(记作事件e)，获得一个时间区间[t1-ε1,t1+ε1]，然后给所有的participants发送prepare消息，participants收到prepare消息后，调TrueTime API返回区间\[t3,t4](保证真实时间肯定在这个区间内)，然后取t3和这个participant目前已commit的最大事务版本号的较大值。记participant维护的已commit的最大事务时间戳为max_commit_timestamp,那么将max(max_commit_timestamp+1, t3)返回给coordinator leader。coordinator leader收集到所有participants返回的prepare时间戳后，从中选出一个最大的，记作maxpreparetimestamp，这个时间戳肯定在真实时间之后，并且比所有的participants的已commit的最大事务版本号都要大。随后coordinator再次调用一次TrueTime API(记作事件f)，获得时间区间[t2-ε2,t2+ε2]。为了保证这个分布式事务的时间戳确实位于这个分布式事务执行过程中的某个点(事件e Happens Before 事件f)，coordinator leader必须为这个分布式事务选择一个介于[t1+ε1,t2-ε2]的时间戳。显然这需要t1+ε1 < t2-ε2.可以看出，Spanner必须等到t1+ε1 < t2-ε2成立之后，才能提交这个事务(也就是说coordinator需要不断的调用TrueTime API,直到返回的时间下界比t1+ε1更大)，大概需要等待2*ε。考虑两个事务T1和T2，时序上T2在T1 commit之后start，由于上面保证了每个事务的commit timestamp都介于事务的start和commit之间，那么显然，T1的commit timestamp小于T2的commit timestamp，从而保证了Linearizability。另外，这个分布式事务的时间戳还必须满足一个条件，就是大于maxpreparetimestamp。从单机的角度来看，如果本地已提交事务版本号比要读的版本号大，就可以读。这就要保证后面提交的单机事务和分布式事务版本号都要比现在已提交事务的版本号更大。否则，读的时候可能会有事务插进来，导致都到的数据可能不是一个快照。


另外，一台机器在收到快照读时，有可能需要阻塞。举个例子，在分布式系统中，即有分布式事务也有单机事务，以A，B，C为例，A为coordinator leader，B和C为participant，以B为例，假设B当前维护的本机最大的commit时间戳为100，现在从A来了一个分布式事务T1的prepare请求，B返回了101给A，在这个分布式事务commit之前，B机器来了一个单机事务T2，并且先于T1提交，时间戳为105，而A可能为这个分布式事务指定的版本号为104. 显然，如果在T1提交前来了一个大于101比如110的快照读事务，这个快照读事务必须被阻塞住直到T1提交才能向客户端返回结果，因为B不知道T1这个还未commit的事务最后的时间戳是多少。

###只读事务

调用TrueTime API，将右区间作为只读事务的版本号即可。 


###两阶段提交的错误处理。

  
  两阶段提交协议由于协调者和参与者的故障可能会有严重的可用性问题。Spanner的两阶段提交实现基于Paxos协议，每个participant和coordinator本身产生的日志都会通过Paxos协议复制到自身的Paxos group中，从而解决可用性问题。同样以A，B，C三份数据为例，他们分别有三个副本，记作(A1，A2，A3)，(B1，B2，B3)，(C1，C2，C3)，每组作为一个Paxos group，内部通过paxos协议保证一致性。假设，A1，B1，C1分别为各自paxos group的leader，A1为coordinator leader。

  Prepare阶段：A1给B1和C1发送prepare消息后，假设B1挂了，A1等待超时，A1给C1发送rollback。B1后续回滚分为两种情况：1. B1在持久化prepare消息之前挂了，B1恢复后可自行回滚 2. 如果B1持久化prepare消息之后挂了，B1自身可以回放日志得知事务未决，主动联系(A1,A2,A3)。A1给B1，C1发送prepare消息之后，自己挂了，同样，A1通过回放日志可以得知。实际上，A1本身挂了之后，A2和A3通过选主协议马上会选出一个新的leader，不至于影响到可用性。

   Commit阶段：A1给B1，C1发送commit消息，B1 commit成功，C1挂了，C1起来后，如果C1之前没有持久化commit消息，则A1主动要求C1继续commit。如果C1之前已经持久化了commit消息，则自己commit。如果C1由于某些原因，始终commit不成功，则由上层业务进行回补操作。