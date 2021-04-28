---
layout: post
title: cassandra笔记
date: 2021-03-29
tags: cassandra  
---
bin/cassandra  
bin/nodetool status

配置  
Durable_writes: 是否写commit_log  
cassandra.yaml  
jvm配置:cassandra-env.sh  

commit_sync : batch、periodic（batch和periodic对比性能差距在好几个数量级以上）  

replication ：  
SimpleStrategy  
NetworkTopologyStrategy  
Snitches 机架感知 路由  

读写一致性  
W + R > RF  

key cache   
row cache  write和update的时候会invalidate cache,read的时候再重新写入  

insert、update、delete 都是一个插入操作,update=upsert,delete 是增加了一个墓碑标志  

map、set、list类型都是sorted  

Frozen Types  
对于collections、udf类型，不允许局部更新，只能整个字段更新,没有用Frozen修饰的，可以局部更新  

cassandra排序方式:  
Within the Data.db file, rows are organized by partition. These partitions are sorted in token order (i.e. by a hash of the partition key when the default partitioner, Murmur3Partition, is used). Within a partition, rows are stored in the order of their clustering keys.

select writetime(name) from test;  
select ttl(name) from test;  


cassandra 设计表的时候，要考虑查询的要求，考虑下面几个平衡点  
1、数据尽量分散到各个节点 但是节点不能太多  
2、单个分区里面的数据不能太多  
3、读、更新的比例  
4、排序  

update ttl 是针对修改的数据  

static column:  
The use of static columns as the following restrictions:
tables with the COMPACT STORAGE option (see below) cannot use them.
a table without clustering columns cannot have static columns (in a table without clustering columns, every partition has only one row, and so every column is inherently static).
only non PRIMARY KEY columns can be static.  

cassandra 很多查询的限制可以用ALLOW FILTERING执行，那些限制主要是为了性能考虑，  
个人理解cassandra 是一个SortedMap<RowKey, SortedMap<ColumnKey, ColumnValue>> 
类型，所以只能一层一层按照层次去查询  

BATCH操作建议是单分区，不建议多分区  
单分区batch能保证原子性和隔离性(另外的查询不会见到部分操作的数据)  
多分区只能保证原子性  
batch操作是先写入一个Batchlog的table，然后从这个table分发到不同的分区节点，操作成功后删除这个batchlog，如果中间有失败会有个定时任务
batch会对协调节点造成压力  
https://inoio.de/blog/2016/01/13/cassandra-to-batch-or-not-to-batch/    

truncate和drop keyspaces、drop tables 都不会生成墓碑,貌似都会生成snapshot  

truncate does not write tombstones at all (instead it will delete all on all nodes for your truncated table sstables immediately)

auto_snapshot (Default: true) Enable or disable whether a snapshot is taken of the data before keyspace truncation or dropping of tables. To prevent data loss, using the default setting is strongly advised. If you set to false, you will lose data on truncation or drop.

When creating or modifying tables, you enable or disable the key cache (partition key cache) or row cache for that table by setting the caching parameter. Other row and key cache tuning and configuration options are set at the global (node) level. Cassandra uses these settings to automatically distribute memory for each table on the node based on the overall workload and specific table usage. You can also configure the save periods for these caches globally.  

二级索引的原理  
https://blog.csdn.net/at99ak77/article/details/46754743  
Cassandra之中的索引的实现相对MySQL的索引来说就要简单粗暴很多了。他实际上是自动偷偷新创建了一张表格，同时将原始表格之中的索引字段作为新索引表的Primary Key！并且存储的值为原始数据的Primary Key  


物化视图其实就是根据原数据 建立一张不同的表,数据量大的情况下会消耗磁盘  

cdc log:  
https://developer.aliyun.com/article/725386  
CDC(Change data capture)是Cassandra提供的一种用于捕获和归档数据写入操作的机制，这个功能在3.8以上版本支持。当对一个表设置了“cdc=true”属性之后，包含有这个表的数据的CommitLog在丢弃时会被移动到指定的目录中，用户可以自己编写程序消费（解析并删除）这些日志，实现诸如增量数据导出、备份等功能。本文介绍CDC功能的使用并分析其特点。  

变更数据捕获（CDC）提供了一种机制来标记特定的表用于存档，一旦数据量达到配置的刷新和未刷新的CDC日志的大小之和，就拒绝对这些表的写入。然后将任何包含启用表（启用CDC）数据的CommitLogSegments移动到cassandra.yaml中指定的目录丢弃。在yaml中指定允许的总磁盘空间的阈值，此时新分配的CommitLogSegments将不允许写入CDC数据，直到消费者解析并从目标存档目录中删除数据  

Read Repair  
分为两种  
前台Read Repair：  
根据一致性级别决定,第一个节点读取数据，其他节点读取digest,比较digest，如果不同则对其他节点全量读取数据，根据最新的数据修复
然后再把结果返回给客户端，会阻塞  

后台Read Repair：  
由read_repair_chance 和 dclocal_read_repair_chance 决定，后台执行

写入流程  
https://cwiki.apache.org/confluence/display/CASSANDRA2/WritePathForUsers  

读取流程  
记得先读取sstable再读取memtable然后合并  
https://cwiki.apache.org/confluence/display/CASSANDRA2/ReadPathForUsers  

写入的时候,没有rollback过程，一个节点写入后，后续通过read repari修复  
写入的时候，满足写入一致性才会有hint，不满足不会有Hint  
max_hint_window_in_ms   
hint数据的回写有两个时机  
1、检测到失败节点恢复  
2、对于瞬间的失败没有检测到节点失败的情况  
The coordinator also checks every ten minutes for hints corresponding to writes that timed out during an outage too brief for the failure detector to notice through gossip

The database does not replay hints older than gc_grace_seconds after creation.  

read的时候 失败了 会有speculative_retry的操作  

speculative_retry   
Xpercentile: Cassandra constantly tracks each table's typical read latency (in milliseconds). If you set speculative retry to Xpercentile, Cassandra sends redundant read requests if the coordinator has not received a response after X percent of the table's typical latency time.
retry triggers if the read operation takes more time than what the fastest X percent of reads took in the past.
In rapid read protection, extra read requests are sent to other replicas, even after the consistency level has been met  

anti-entropy repair是采用Merkle trees实现的  
Merkle trees (默克尔树)  
https://blog.csdn.net/wo541075754/article/details/54632929  


Log Structured Merge Tree (LSM)   
就是包括WAL（write ahead log）也称预写log、MemTable、SSTable,磁盘顺序写、Log合并等  

min_index_interval 和 max_index_interval 和 -Index.db  这个文件的索引抽取有关系  

Size Tiered Compaction Strategy（STCS）  
优点：写占比高的情况下压缩效果好。  
缺点：  
（1）将读操作变慢了，因为根据大小来进行合并的过程并不会对数据进行分组，这样使某个特定行的多个版本很有可能分散在多个SSTable中。  
（2）浪费存储空间。由于SSTable是经过一段时间后合并的，一个被删除的记录，它的老版本可能一直存储在旧的SSTable中，直到新的合并出现才会将这些记录删除。因此对于一些经常进行删除操作的系统，其浪费的的空间是很大的。  
（3）压缩时占用大量的空间。随着时间的推移，系统中会出现一些很大的SSTable。这时如果需要合并四个很大的SSTable，压缩占用的存储空间就是这四个SSTable大小的总和。  

Leveled Compaction Strategy（LCS）  
（1）每一层的各个SSTable中的数据都有序不相交，可以保证90%的读操作都在一个SSTable中完成，最坏情况是一个记录存在每一层，这样最坏情况下10TB数据也只需要读7层，也就是7个SSTable。  
（2）最多只有１０％的空间会被浪费。因为最坏的情况是该层的记录和完全存在在下一层中，而且每一层都是这种情况。也就是会所每一层都有１０％（下一层数据是上一层的１０倍）的数据时冗余的。  
（3）在压缩合并操作的开销上，每次只会使用10倍于要压缩的sstable大小的空间。  
适用条件：  
对于一个更新操作和删除操作比较多的系统，或者读操作占比高的系统，使用分层压缩是比较合适的。因为这种系统会产生同一份数据的多个版本。但是由于这种压缩会在压缩中进行更多的IO操作，所以如果是一个主要是insert操作的系统，建议不要使用分层压缩方法。  

Time Window Compaction Strategy（TWCS）  
优势:  
用作时间序列数据，为表中所有数据使用默认的TTL。比DTCS配置更简单。
劣势:不适用于时间乱序的数据，因为SSTables不会继续做压缩，存储会没有边界的增长，所以也不适用于没有设置TTL（Time To Live）的数据。相比较DTCS，需要更少的调优配置。  


cassandra 压缩  
1、压缩的主要好处是它减少了写入磁盘的数据量。  
2、不仅减小的大小节省了存储需求，它经常增加读取和写入吞吐量，因为压缩数据的CPU开销比从磁盘读取或写入更大量的未压缩数据所需的时间更快。  


jvm配置  
https://docs.datastax.com/en/archived/cassandra/3.0/cassandra/operations/opsTuneJVM.html  

https://blog.csdn.net/u011250186/article/details/106917077  

If the option only_purge_repaired_tombstones is enabled, tombstones are only removed if the data has also been repaired.
only_purge_repaired_tombstones=true,那么只有在repair后tombstones才会清除，不管gc_grace_seconds  


Cassandra isn’t optimized for large file or BLOB storage and a single blob value is always read and send to the client entirely. As such, storing small blobs (less than single digit MB) should not be a problem, but it is advised to manually split large blobs into smaller chunks.  

What happens if two updates are made with the same timestamp?
Updates must be commutative, since they may arrive in different orders on different replicas. As long as Cassandra has a deterministic way to pick the winner (in a timestamp tie), the one selected is as valid as any other, and the specifics should be treated as an implementation detail. That said, in the case of a timestamp tie, Cassandra follows two rules: first, deletes take precedence over inserts/updates. Second, if there are two updates, the one with the lexically larger value is selected.  


https://cassandra.apache.org/doc/3.11/tools/cqlsh.html?highlight=parsing
TRACING ON  
TRACING OFF  

EXPAND ON  
EXPAND OFF  

cassandra 轻量级事务，basic paxos,会有性能损耗，有点类似cas  
insert的时候 IF NOT EXISTS  
update的时候 IF ( EXISTS | condition）  
delete的时候 IF ( EXISTS | condition）
https://developer.aliyun.com/article/714656  

Cassandra内存占用  
Cassandra performs the following major operations within JVM heap:  
To perform reads, Cassandra maintains the following components in heap memory:  
Bloom filters  
Partition summary  
Partition key cache  
Compression offsets  
SSTable index summary  
This metadata resides in memory and is proportional to total data  

Cassandra gathers replicas for a read or for anti-entropy repair and compares the replicas in heap memory.
Data written to Cassandra is first stored in memtables in heap memory. Memtables are flushed to SSTables on disk.
To improve performance, Cassandra also uses off-heap memory as follows:
Page cache. Cassandra uses additional memory as page cache when reading files on disk.
The Bloom filter and compression offset maps reside off-heap.
Cassandra can store cached rows in native memory, outside the Java heap. This reduces JVM heap requirements, which helps keep the heap size in the sweet spot for JVM garbage collection performance.  

https://docs.datastax.com/en/developer/java-driver/3.4/manual/  

default_time_to_live 是针对column级别的，到期了就生成tombstones  


WITH COMPACT STORAGE  
https://wiki.jikexueyuan.com/project/wiki-journal-201507-1/cassandra-compact-storage.html  


(  
cluster_name  
num_tokens  
hinted_handoff_enabled  
max_hint_window_in_ms  
hints_directory  
hints_flush_period_in_ms  
max_hints_file_size_in_mb  
batchlog_replay_throttle_in_kb  
partitioner  
key_cache_size_in_mb  
seed_provider  
concurrent_reads  
concurrent_writes  
memtable_heap_space_in_mb  
memtable_offheap_space_in_mb  
memtable_allocation_type（https://www.datastax.com/blog/heapmemtables-cassandra-21）  
repair_session_max_tree_depth (https://zhaoyanblog.com/archives/1023.html)  
memtable_flush_writers  
index_summary_capacity_in_mb  
column_index_size_in_kb(https://blog.csdn.net/FireCoder/article/details/5731666,Cassandra假定同一个row key可以有成千上万个column,所以对column也有一个index,每隔column_index_size_in_kb 序列化，rang思想)  
column_index_cache_size_in_kb  
concurrent_compactors  
read_request_timeout_in_ms  
write_request_timeout_in_ms  
slow_query_log_timeout_in_ms  
gc_log_threshold_in_ms  
gc_warn_threshold_in_ms  
max_value_size_in_mb  
cross_node_timeout  
range_request_timeout_in_ms  
request_timeout_in_ms  
buffer_pool_use_heap_if_exhausted  
disk_failure_policy  
commit_failure_policy  
)  

硬件规格和容量规划  
https://cassandra.apache.org/doc/3.11/operating/hardware.html  
https://cassandra.apache.org/doc/latest/data_modeling/data_modeling_refining.html
  
转载请注明：[这不是一只猫的博客](http://1024.notacat.cn) » [点击阅读原文](http://1024.notacat.cn/2021/03/cassandra%E7%AC%94%E8%AE%B0/)


