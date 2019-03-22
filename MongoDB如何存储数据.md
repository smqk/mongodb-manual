# MongoDB如何存储数据



想要深入了解MongoDB如何存储数据之前，有一个概念必须清楚，那就是Memeory-Mapped Files。

# Memeory-Mapped Files

下图展示了数据库是如何跟底层系统打交道的。

- 内存映射文件是OS通过mmap在内存中创建一个数据文件，这样就把文件映射到一个虚拟内存的区域。
- 虚拟内存对于进程来说，是一个物理内存的抽象，寻址空间大小为2^64
- 操作系统通过mmap来把进程所需的所有数据映射到这个地址空间(红线)，然后再把当前需要处理的数据映射到物理内存(灰线)
- 当进程访问某个数据时，如果数据不在虚拟内存里，触发page fault，然后OS从硬盘里把数据加载进虚拟内存和物理内存
- 如果物理内存满了，触发swap-out操作，这时有些数据就需要写回磁盘，如果是纯粹的内存数据，写回swap分区，如果不是就写回磁盘。

![img](images/13172853-be5f9db5554f4dbcba2b5a5e6ab538ec.png)

 

# MongoDB的存储模型

![img](images/14112916-de9fe4d822ee403a94a11c87605e3799.png)

 

- 有了内存映射文件，要访问的数据就好像都在内存里面，简单化了MongoDB访问和修改数据的逻辑
- MongoDB读写都只是和虚拟内存打交道，剩下都交给OS打理
- 虚拟内存大小=所有文件大小+其他一些开销(连接，堆栈)
- 如果journal开启，虚拟内存大小差不多翻番
- 使用MMF的好处1：不用自己管理内存和磁盘调度2：LRU策略3：重启过程中，Cache依然在。
- 使用MMF的坏处1：RAM使用会受磁盘碎片的影响，高预读也会影响2：无法自己优化调度算法，只能使用LRU

 

![img](images/14112957-9df227b538f6491f83f62de3f667b618.png)

![img](images/14113402-dcc86495473145948be7214910080257.png)

![img](images/14113219-3aa96ad975f8457b9673b30c7fdad3ae.png)

 

- 磁盘上的文件是有extent构成，分配集合空间的时候也是以extent为单位进行分配的
- 一个集合有一个或者多个etent
- ns文件里面命名空间记录指向那个集合的第一个extent

  

# 数据文件与空间分配

当创建数据库时(其实MongoDB没有显式创建数据库的方法，在向数据库中的集合写入数据时会自动创建该数据库)，MongoDB会在磁盘上分配一组数据文件，所有集合，索引和数据库的其他元数据都保存在这些文件里。数据文件被放在启动时指定的dbpath里，默认放入/data/db下面。典型的一个文件组织结构如下：

```bash
$ cat /data/db
$ ls -al
-rw------- 1 root root   16777216 09-18 00:54 local.ns
-rw------- 1 root root   67108864 09-18 00:54 local.0
-rw------- 1 root root 2146435072 09-18 00:55 local.1
-rw------- 1 root root 2146435072 09-18 00:56 local.2
-rw------- 1 root root 2146435072 09-18 00:57 local.3
-rw------- 1 root root 2146435072 09-18 00:58 local.4
-rw------- 1 root root 2146435072 09-18 00:59 local.5
-rw------- 1 root root 2146435072 09-18 01:01 local.6
-rw------- 1 root root 2146435072 09-18 01:02 local.7
-rw------- 1 root root 2146435072 09-18 01:03 local.8
-rw------- 1 root root 2146435072 09-18 01:04 local.9
-rw------- 1 root root 2146435072 09-18 01:05 local.10
-rw------- 1 root root   16777216 09-18 01:06 test.ns
-rw------- 1 root root   67108864 09-18 01:06 test.0
-rw------- 1 root root  134217728 09-18 01:06 test.1
-rw------- 1 root root  268435456 09-18 01:06 test.2
-rw------- 1 root root  536870912 09-18 01:06 test.3
-rw------- 1 root root 1073741824 09-18 01:07 test.4
-rw------- 1 root root 2146435072 09-18 01:07 test.5
-rw------- 1 root root 2146435072 09-18 01:09 test.6
-rw------- 1 root root 2146435072 09-18 01:11 test.7
-rw------- 1 root root 2146435072 09-18 01:13 test.8
...
-rwxr-xr-x 1 root root          6 09-18 13:54 mongod.lock
drwxr-xr-x 2 root root       4096 11-13 18:39 journal
drwxr-xr-x 2 root root       4096 11-13 19:02 _tmp
```

- mongod.lock中存储了服务器的进程ID，是一个进程锁定文件。数据文件是依据所属的数据库命名的。

- test.ns是第一个生成的文件(ns扩展名就是namespace的意思)，数据库中的每个集合和索引都有自己的命名空间，每个命名空间的元数据都存放在这个文件里。默认情况下，.ns文件大小固定在16MB，大约可以存储24000个命名空间。也就是说数据库中的索引和集合总数不能超过24000，该值可以通过mongod的--nssize选项进行定制。

- 像test.0这样以0开始的整数结尾的文件就是集合和索引数据文件。刚开始的时候，即使只有一条数据，MongoDB也会预分配几个文件，这种预分配的做法，能让数据尽可能连续存储，减少磁盘碎片。在像数据库添加数据时，MongoDB会分配更多的数据文件。每个新数据文件的大小都是上一个已分配文件的两倍(64M->128M->256M)，直到预分配文件大小的上限2G。此处基于一个假设，如果总数据大小呈恒定速率增长，应该逐渐增加数据文件分配的空间。当然这个预分配策略也是可以通过--noprealloc关掉，但是不建议在production环境下使用。

- 默认的local数据库，该数据库不参与replication。当mongod是一个副本集的成员时，在local数据库中就有一个叫做oplog.rs的预分配的capped集合，预分配的大小为磁盘空间的5%。这个大小可以通过--oplogSize进行调整。oplog主要用于副本集Primary和Secondary成员见的replication，它的大小限制了两个副本集之间，在重新完全同步之前，允许多长时间不同步。

- journal目录，journal功能2.4版本默认是开启的。

- 可以使用db.stats()来确认已使用空间和已分配空间。

  ```javascript
  {
      "db" : "test",
      "collections" : 37,
      "objects" : 317894523,  #文档总个数
      "avgObjSize" : 232.3416429039893,  #单位是字节
      "dataSize" : 73860135744, #集合中所有数据实际大小(包括padding factor为每个文档分配的额外空间以允许文档增长)。该值在文档size变小的时候，这个值不会减少，除非文档被删除，或者执行compact或者repairDatabase操作
      "storageSize" : 97834319392, #分配给集合的空间大小(包括为集合增长预留的额外空间和未分配的已删除空间，即不会因为文档size变小或者删除而减小)，实际上从数据文件中分配给集合的空间是以块为单位，也称之为extents，即分配的extents的大小
      "numExtents" : 385,
      "indexes" : 86,
      "indexSize" : 58687466992,
      "fileSize" : 182380920832, #所有数据文件大小之和，不包括命名空间文件(ns文件)
      "nsSizeMB" : 16,
      "dataFileVersion" : {
          "major" : 4,
          "minor" : 5
      },
      "ok" : 1
  }
  ```

    

- 使用db.accesslog.stats()确认某个集合的使用量

  ```javascript
  {
      "ns" : "test.accesslog",
      "count" : 145352932,
      "size" : 37060264352, ＃实际数据大小，不包括索引
      "avgObjSize" : 254.967435758365,
      "storageSize" : 45794676448, #预分配的数据存储空间
      "numExtents" : 42,
      "nindexes" : 4,
      "lastExtentSize" : 2146426864,
      "paddingFactor" : 1, #当文档因更新size增长时事先padding可以提速，减少碎片的产生
      "systemFlags" : 1,
      "userFlags" : 0,
      "totalIndexSize" : 31897944512,
      "indexSizes" : {
          "_id_" : 6722168208,
          "action_1_time_1" : 8606482752,
          "gz_id_1_action_1_time_1" : 10753778336,
          "time_1" : 5815515216
      },
      "ok" : 1
  }
  ```



> 来自：https://www.cnblogs.com/foxracle/p/3421893.html

