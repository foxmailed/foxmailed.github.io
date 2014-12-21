这篇分析一下namenode 写edit log的过程。

关于namenode日志,集群做了如下配置
```
  <property>
    <name>dfs.nameservices</name>
    <value>sync</value>
    <description>Logical name for this new nameservice</description>
  </property>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file://home/wudi/hadoop/nn</value>
  </property>
  <property>
    <name>dfs.namenode.shared.edits.dir</name>
    <value>qjournal://host1:port1;host2:port2;host3:port3/sync</value>
  </property>
```
这个配置是说namenode写edit log需要往两个地方写，第一个是/home/wudi/hadoop/nn,namenode本地文件系统,另外一个qjournal,这是一个共享的edit log directory,namenode往多个JournalNode写edit log，JournalNode之间运行Paxos协议，保证多点写时也能对edit log达成一致。实际上，我的集群上起了3个JournalNode进程。

总体来说，namenode多线程写edit log，edit log维护双buffer，一个用于填充数据，另外一个
用于flush。往buffer中写edit log需要事先加锁，写完后检查如果buffer中数据大小达到阈值，
则进行sync,将buffer真正写出. 或者，线程主动调用sync，主动将buffer写出去.sync时，也要加锁，和往buffer中写edit log是同一把锁，拿住锁后，切buffer，然后解锁，在锁外面将buffer写出去。
在我的配置中，需要写两个地方，一个是namenode本地的存edit log的目录file://home/wudi/hadoop/nn，另外一个是qjournal，往三个JournalNode进程并行写.

下面看看代码：

FSEditLog的初始化

```
 FSEditLog(Configuration conf, NNStorage storage, List<URI> editsDirs) {
    isSyncRunning = false;
    this.conf = conf;
    this.storage = storage;
    metrics = NameNode.getNameNodeMetrics();
    lastPrintTime = now();
    // If this list is empty, an error will be thrown on first use
    // of the editlog, as no journals will exist
    this.editsDirs = Lists.newArrayList(editsDirs);

    this.sharedEditsDirs = FSNamesystem.getSharedEditsDirs(conf);
  }
```

this.editsDirs就是配置项dfs.namenode.name.dir和dfs.namenode.shared.edits.dir的
和.
this.sharedEditsDirs是配置项dfs.namenode.shared.edits.dir

```
  private synchronized void initJournals(List<URI> dirs) {
    int minimumRedundantJournals = conf.getInt(
        DFSConfigKeys.DFS_NAMENODE_EDITS_DIR_MINIMUM_KEY,
        DFSConfigKeys.DFS_NAMENODE_EDITS_DIR_MINIMUM_DEFAULT);

    journalSet = new JournalSet(minimumRedundantJournals);

    for (URI u : dirs) {
      boolean required =FSNamesystem.getRequiredNamespaceEditsDirs(conf)
          .contains(u);
      if (u.getScheme().equals(NNStorage.LOCAL_URI_SCHEME)) {
        StorageDirectory sd = storage.getStorageDirectory(u);
        if (sd != null) {
          journalSet.add(new FileJournalManager(conf, sd, storage),
              required, sharedEditsDirs.contains(u));
        }
      } else {
        journalSet.add(createJournal(u), required,
            sharedEditsDirs.contains(u));
      }
    }
 
    if (journalSet.isEmpty()) {
      LOG.error("No edits directories configured!");
    } 
  }
```

传进来的是this.editsDirs,一个是本地edit log目录，另外一个是qjournal,JournalSet用来管理多个edit log directory，包括本地的和共享的,那么在我的集群配置下,journalSet里面有两个JournalAndStream对象。JournalAndStream对象包装了具体的edit log输出流和具体的管理流的manager。对于qjournal来说，manager是QuorumJournalManager，对于本地目录来说，manager是FileJournalManager.不同的manager 使用不同的edit log输出流，每一种具体的输出流都继承自EditLogOutputStream这个基类.每次切换edit log segment时，会调用manager的startLogSegment方法来生成一个新的输出流。对于QuorumJournalManager来说，输出流是QuorumOutputStream，对于FileJournalManager来说，输出流是EditLogFileOutputStream.用户可以实现自己的manager，通过配置参数dfs.namenode.edits.journal-plugin.qjournal。上层FSEditLog调用startLogSegment切换一个edit log segment时，调用的是JournalSet的startLogSegment，它会调用它所包含的manager的startLogSegment，这样就产生出了两个输出流。

下面看看写edit log

一般来说，namenode写edit log的函数调用顺序是先调void logEdit(final FSEditLogOp op)然后调用public void logSync(),这种方式主要是为了做batch，提高吞吐.logEdit往buffer里写，logSync在真正flush.

先看FSEditLog的logEdit:

```
 void logEdit(final FSEditLogOp op) {
    synchronized (this) {
      assert isOpenForWrite() :
        "bad state: " + state;
      // wait if an automatic sync is scheduled
      waitIfAutoSyncScheduled();
      long start = beginTransaction();
      op.setTransactionId(txid);

      try {
        editLogStream.write(op);
      } catch (IOException ex) {
        // All journals failed, it is handled in logSync.
      }

      endTransaction(start);
      // check if it is time to schedule an automatic sync
      if (!shouldForceSync()) {
        return;
      }
      isAutoSyncScheduled = true;
    }
    // sync buffered edit log entries to persistent store
    logSync();
  }

```

首先，会检查是否sync操作已经被别人调度了(检查isAutoSyncScheduled变量)，如果是，说明别的线程即将进行sync操作，则该线程wait，别的线程将buffer切换好后，调用doneWithAutoSyncScheduling将isAutoSyncScheduled置为false，然后将其他等待的线程唤醒. 接着，为edit log分配一个transaction id，id从全局分配器txid分配，以1递增，获得的transaction id保存在线程私有变量中，然后将op写入QuorumOutputStream和EditLogFileOutputStream的buffer中.接着调用shouldForceSync()这个方法会检查每个流的shouldForceSync()，只要有一个返回true,就返回true，意思是buffer够大了，攒的差不多了，该sync一次了，接着就调度一次sync将isAutoSyncScheduled置为true.然后调logSync().
QuorumOutputStream这个流永远返回false,EditLogFileOutputStream发现buffer中数据超过512KB(不可配置)，则返回true.如果buffer不满512KB，logEdit()会直接返回，不进行logSync,可以看到这里对log进行了batch。
下面看logSync()


```
public void logSync() {
    long syncStart = 0;

    // Fetch the transactionId of this thread.
    long mytxid = myTransactionId.get().txid;
    boolean sync = false;
    try {
      EditLogOutputStream logStream = null;
      synchronized (this) {
        try {
          printStatistics(false);

          // if somebody is already syncing, then wait
          while (mytxid > synctxid && isSyncRunning) {
            try {
              wait(1000);
            } catch (InterruptedException ie) {
            }
          }
          //
          // If this transaction was already flushed, then nothing to do
          //
          if (mytxid <= synctxid) {
            numTransactionsBatchedInSync++;
            if (metrics != null) {
              // Metrics is non-null only when used inside name node
              metrics.incrTransactionsBatchedInSync();
            }
            return;
          }
          // now, this thread will do the sync
          syncStart = txid;
          isSyncRunning = true;
          sync = true;
          // swap buffers
          try {
            if (journalSet.isEmpty()) {
              throw new IOException("No journals available to flush");
            }
            editLogStream.setReadyToFlush();
          } catch (IOException e) {
            final String msg =
                "Could not sync enough journals to persistent storage " +
                "due to " + e.getMessage() + ". " +
                "Unsynced transactions: " + (txid - synctxid);
            LOG.fatal(msg, new Exception());
            IOUtils.cleanup(LOG, journalSet);
            terminate(1, msg);
          }
        } finally {
          // Prevent RuntimeException from blocking other log edit write 
          doneWithAutoSyncScheduling();
        }
        //editLogStream may become null,
        //so store a local variable for flush.
        logStream = editLogStream;
      }
      // do the sync
      long start = now();
      try {
        if (logStream != null) {
          logStream.flush();
        }
      } catch (IOException ex) {
        synchronized (this) {
          final String msg =
              "Could not sync enough journals to persistent storage. "
              + "Unsynced transactions: " + (txid - synctxid);
          LOG.fatal(msg, new Exception());
          IOUtils.cleanup(LOG, journalSet);
          terminate(1, msg);
        }
      }
      long elapsed = now() - start;
      if (metrics != null) { // Metrics non-null only when used inside name node
        metrics.addSync(elapsed);
      }
    } finally {
      // Prevent RuntimeException from blocking other log edit sync 
      synchronized (this) {
        if (sync) {
          synctxid = syncStart;
          isSyncRunning = false;
        }
        this.notifyAll();
     }
    }
  }
```

首先，检查是不是别的线程正在做sync(isSyncRunning)，如果别的线程正在做并且当前edit log
的mytxid大于到目前位置已经sync的最大的synctxid，那么等待。别的线程sync完成后会更新
synctxid，并且isSyncRunning置为false，然后唤醒这个线程。线程醒来后，检查是否自己的mytxid
对应的edit log已经被sync了，如果是，返回。否则，开始做sync，将isSyncRunning置为true告诉别的线程。然后调用setReadyToFlush切换buffer，调用doneWithAutoSyncScheduling允许别的线程往buffer中写数据。然后进行实际的flush。最后更新synctxid并置isSyncRunning置为false，然后唤醒其他线程.

结束.

####参考资料

hadoop-hdfs-2.4.1.jar

[qjournal-design](https://issues.apache.org/jira/secure/attachment/12547598/qjournal-design.pdf)




