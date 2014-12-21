##HBase Compaction策略

RegionServer这种类LSM存储引擎需要不断的进行Compaction来减少磁盘上数据文件的个数和删除无用的数据从而保证读性能。

RegionServer后台有一组负责flush region的线程(MemStoreFlusher)，每次从flushQueue中拿出一个flush region请求，会检查这个region是否有某个store包含的storefile个数超过配置hbase.hstore.blockingStoreFiles，默认7，如果超过,说明storefile个数已经到了会影响读性能的地步，那么就看这个flush region请求是否已经blockingWaitTime（hbase.hstore.blockingWaitTime,默认90s）没有执行了,如果是，这时候需要立即执行flush region，为了防止OOM。如果没有超过blockingWaitTime，那么先看看region是否需要分裂，如果不需要，则向后台的CompactionSplitThread请求做一次Compaction(从这里可以看出，split优先级比compaction高)，然后重新将这个flush region加入flushQueue，延后做flush.

重点看Compaction

Compaction以store为单位，CompactSplitThread会为region的每个store生成一个CompactionRequest.一个Compaction根据它包含的storefile的总大小，可以分为large compaction和small compaction，这两种compaction分别被两个不同的线程池处理。系统一般认为small compaction占大多数，所以上文中由于storefile过多系统自动触发的system compaction 默认放入small compaction池子中处理.

```
//系统自动触发的system compaction，selectNow参数为false，实际选取待compact的
filelist过程延后在CompactionRunner中做.
if (selectNow) {
    // 通过hbase shell触发的major compaction,selectNow为true.这里进行实际的选取待compact filelist操作
    compaction = selectCompaction(r, s, priority, request);
    if (compaction == null) return null; // message logged inside
}
// We assume that most compactions are small. So, put system compactions
//into small pool; we will do selection there, and move to large pool if //necessary.

long size = selectNow ? compaction.getRequest().getSize() : 0;

// 从这里可以看出，用户外部触发的compaction默认放入small compaction线程池中处理，并且
// system compaction 也会放入small compaction线程池中，后续真正执行
// system compaction时，会根据选出的storefile的总大小来决定最终放入large还是small线程池
ThreadPoolExecutor pool = (!selectNow && s.throttleCompaction(size))? largeCompactions : smallCompactions;
pool.execute(new CompactionRunner(s, r, compaction, pool));
```

看看执行compaction过程的CompactionRunner任务。

```
 // Common case - system compaction without a file selection. Select now.
 // system compaction 还没有选择待compact的filelist,为null
 if (this.compaction == null) {
   int oldPriority = this.queuedPriority;
   this.queuedPriority = this.store.getCompactPriority();
   if (this.queuedPriority > oldPriority) {
     // Store priority decreased while we were in queue (due to some other compaction?),
     // requeue with new priority to avoid blocking potential higher priorities.
     this.parent.execute(this);
     return;
   }
   try {
     // 选择storefile
     this.compaction = selectCompaction(this.region, this.store, queuedPriority, null);
   } catch (IOException ex) {
     LOG.error("Compaction selection failed " + this, ex);
     server.checkFileSystem();
     return;
   }
   if (this.compaction == null) return; // nothing to do
   // Now see if we are in correct pool for the size; if not, go to the correct one.
   // We might end up waiting for a while, so cancel the selection.
   assert this.compaction.hasSelection();
   // 判断这次compaction放入small还是large池中执行
   ThreadPoolExecutor pool = store.throttleCompaction(
       compaction.getRequest().getSize()) ? largeCompactions : smallCompactions;
   // system compaction应该放入large池
   if (this.parent != pool) {
     this.store.cancelRequestedCompaction(this.compaction);
     this.compaction = null;
     this.parent = pool;
     // 在large池子中执行
     this.parent.execute(this);
     return;
   }
 }
```

large compaction和small compaction分界线由hbase.regionserver.thread.compaction.throttle参数决定，如果没有设置，默认为2 \* hbase.hstore.compaction.max \* hbase.hregion.memstore.flush.size
全部取默认值等于2\*10\*128MB = 2.5GB

从以上可以看出，system compaction默认放入small池，当选出storefile list后，再根据size去判断最终放入small还是large线程池中执行.对于外部触发的compaction，放入small中执行.

选定池子后，下面看每个store compaction具体的步骤.

两个步骤:

1. 根据某种策略生成compact的目标storefile集合
2. 进行compaction

这两步都在CompactionRunner这个runnable任务中完成。这里主要说第一个步骤：入口在HStore::requestCompaction.

首先创建storeEngine相应的CompactionContext，这个context用来存各种compact相关的信息，
最重要的就是CompactionRequest，作为上面第二个步骤的输入. HBase 0.98主要有两种存储引擎,DefaultStoreEngine和[StripeStoreEngine](https://issues.apache.org/jira/browse/HBASE-7667)，这里的存储引擎主要是管理磁盘上的storefile文件和flush 内存中的snapshot memstore到磁盘。StripStoreEngine比较特别，一个snapshot memstore刷到磁盘上有可能多于一个storefile文件，这里不讨论.大部分人都使用默认的storeEngine.

其次，创建完context后，然后调用compactionPolicy的selectCompaction()，将store下面的所有storefile传进去，供其挑选.HBase的compaction policy可通过配置项hbase.hstore.defaultengine.compactionpolicy.class配置，默认是ExploringCompactionPolicy.class
下面看selectCompaction()，主要有几个步骤：

1. 从store下面的storefiles中过滤掉比正在compacting的storefilelist中最新的storefile更老的storefile(输入的storefile按照如下规则排序)

	```
     public static final Comparator<StoreFile> SEQ_ID =
      Ordering.compound(ImmutableList.of(
          Ordering.natural().onResultOf(new GetSeqId()),
          Ordering.natural().onResultOf(new GetFileSize()).reverse(),
          Ordering.natural().onResultOf(new GetBulkTime()),
          Ordering.natural().onResultOf(new GetPathName())
      ));
	```

	seq id是storefile对应的snapshot memstore在flush时，从region内部的全局递增计
数器sequenceId中取到的，可以看到，seq id越大的storefile越新.对多个文件进行compact后产生的新的storefile的seq id被设置为多个文件中最大的seq id

2. 如果不是major compaction，就检查：如果配置了删除ttl到期的storefile，并且ttl是一个有限的值，那么这次compaction只会选ttl到期的storefile，如果目前确实存在ttl过期的storefile，则这次compaction选取的文件列表就是这些过期的storefile，选取文件流程结束,CompactionRequest产生。如果没有配置，则过滤掉文件大小大于配置值
hbase.hstore.compaction.max.size(默认是 Long.MAX_VALUE)的storefile(实际上，这块实现有问题)
3. 根据一些规则和参数，判断是否升级为major compaction，比较烦，直接贴代码

	```
 // Force a major compaction if this is a user-requested major compaction,
 // or if we do not have too many files to compact and this was requested
 // as a major compaction.
 // Or, if there are any references among the candidates.
 boolean majorCompaction = (
   (forceMajor && isUserCompaction)
   || ((forceMajor || isMajorCompaction(candidateSelection))
       && (candidateSelection.size() < comConf.getMaxFilesToCompact()))
   || StoreUtils.hasReferences(candidateSelection)
   );
	```

	- 如果不是，那么这次compaction是一个minor compaction，做以下几件事
    	- 过滤掉bulk load的storefile
    	- 应用ExploringCompactionPolicy重写的applyCompactionPolicy方法，挑选storefile的思想是：枚举所有的n个连续文件，n位于[hbase.hstore.compaction.min(默认3), hbase.hstore.compaction.max(默认10)]之间, 这是对于minor compaction的文件个数的限制。并且n个连续的文件大小总和不能超过hbase.hstore.compaction.max.size(默认MAX_VALUE)，并且n个文件的大小之间的"方差"不能太大. 最后选出n个文件，选择的原则是：删除最多的文件同时这些文件的
大小总和小(消耗更少的磁盘IO)
	- 检查是否选出来的storefile个数超过hbase.hstore.compaction.max，如果超过，并且
这只是minor compaction，则从storefile文件集合尾部将多余的storefile过滤掉，如果超过但是是major compaction并且是用户发起的，则不过滤.至此，这次compact的storefile集合产生，结束。

至此，第一个步骤结束，compact的目标storefile选出.

###参考资料

https://github.com/apache/hbase/tree/0.98
















