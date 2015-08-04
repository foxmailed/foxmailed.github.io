###基于KV Data Model实现Table Data Model

HBase对外暴露出来的是一个表格数据模型，如下图所示

![](http://pic002.cnblogs.com/images/2012/176446/2012010621260722.jpg)

rowkey应用程序可以自己设计。每一个Cell可以保存多个版本的数据，由timestamp标示版本。应用程序可以自己指定timestamp，如果不指定HBase会设置为当前时间。

RegionServer是HBase对外提供数据存取的server，核心的数据结构是java.util.concurrent ConcurrentSkipListMap, 实际上是一个用SkipList实现有序Map，对外是一个KV的接口。那上图中的表格数据模型是如何映射到KV接口上的?

实际上，从表格模型可以看出，<rowkey,column family, column qualifier, timestamp>这个4元组唯一的决定一个value，从这里可以看出，KV接口的Key基本就是由上述这几项组成。具体的在HBase中， 
rowkey, column family, column qualifier, timestamp,value 都存储在结构KeyValue中，结构如下图所示：


![](http://www.searchtb.com/wp-content/uploads/2011/01/image0090.jpg)


这个结构直接存储在ConcurrentSkipListMap中，作为Key，ConcurrentSkipListMap的Value不重要，在实现中也是KeyValue。

ConcurrentSkipListMap在构造时，会传入一个java.util.Comparator<T>，实现中对普通的数据表格来说，传入的是KVComparator，它比较两个KeyValue，比较准则是先比较按照字典序比较rowkey，如果相等则比较column family,接着是column qualifier, timestamp, type, mvcc。其中type有如下几种：

```
public static enum Type {
    Minimum((byte)0),
    Put((byte)4),
    Delete((byte)8),
    DeleteFamilyVersion((byte)10),
    DeleteColumn((byte)12),
    DeleteFamily((byte)14),

    // Maximum is used when searching; you look from maximum on down.
    Maximum((byte)255);
}
```
所以实际上，删除某行，或者某个column family，或者某个qualifier的过程实际上都是往ConcurrentSkipListMap插入了一行，并且，一次put多个qualifier最后体现在ConcurrentSkipListMap中也是多行。那么，显然，在读一行数据的时候，需要scan这个ConcurrentSkipListMap，将rowkey相同的行都给扫描出来，将相关的qualifier组装起来，并且需要处理各种Delete相关的type，TTL以及KeyValue是否可读的问题(mvcc字段，关于MVCC，参见这篇[博客](http://www.cnblogs.com/foxmailed/p/3897884.html)),还有用户设置的一些filter等等。由于用户一般读数据都是读最新的版本，为了scan更少的数据，KeyValue的排序中有一个规则是，timestamp越大的KeyValue排在更前面。


LevelDB/RocksDB实际上提供的就是KV的接口，Key和Value都是byte[]。它实际上是一个LSM的系统。普遍认为它是BigTable的tablet server中的一个模块，将和NameNode打交道，上层的表格封装给去掉了。

实际上，国内Baidu的网页搜索部已经开源了一个叫做Tera的表格系统，其中tablet server就是基于LevelDB做的。

####参考资料

[HBase 0.98](https://github.com/apache/hbase/tree/0.98)

[Tera](https://github.com/baidu/tera)





