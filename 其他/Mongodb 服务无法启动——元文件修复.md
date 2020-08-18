# Mongodb 服务无法启动——元文件修复.md



连接服务器 查看 mongodb 日志错误信息：

```
2020-04-21T10:12:44.839+0800 E STORAGE  [WTCheckpointThread] WiredTiger error (28) [1587435164:839015][11912:0x7f69645bd700], file:collection-10--749927511012856183.wt, WT_SESSION.checkpoint: __posix_file_write, 579: /var/lib/mongo/collection-10--749927511012856183.wt: handle-write: pwrite: failed to write 61440 bytes at offset 1440296960: No space left on device Raw: [1587435164:839015][11912:0x7f69645bd700], file:collection-10--749927511012856183.wt, WT_SESSION.checkpoint: __posix_file_write, 579: /var/lib/mongo/collection-10--749927511012856183.wt: handle-write: pwrite: failed to write 61440 bytes at offset 1440296960: No space left on device
2020-04-21T10:12:44.839+0800 E STORAGE  [WTCheckpointThread] WiredTiger error (22) [1587435164:839096][11912:0x7f69645bd700], file:index-7--749927511012856183.wt, WT_SESSION.checkpoint: __wt_block_checkpoint_resolve, 859: index-7--749927511012856183.wt: the checkpoint failed, the system must restart: Invalid argument Raw: [1587435164:839096][11912:0x7f69645bd700], file:index-7--749927511012856183.wt, WT_SESSION.checkpoint: __wt_block_checkpoint_resolve, 859: index-7--749927511012856183.wt: the checkpoint failed, the system must restart: Invalid argument
2020-04-21T10:12:44.839+0800 E STORAGE  [WTCheckpointThread] 、 error (-31804) [1587435164:839123][11912:0x7f69645bd700], file:index-7--749927511012856183.wt, WT_SESSION.checkpoint: __wt_panic, 520: the process must exit and restart: WT_PANIC: WiredTiger library panic Raw: [1587435164:839123][11912:0x7f69645bd700], file:index-7--749927511012856183.wt, WT_SESSION.checkpoint: __wt_panic, 520: the process must exit and restart: WT_PANIC: WiredTiger library panic
2020-04-21T10:12:44.839+0800 F -        [WTCheckpointThread] Fatal Assertion 50853 at src/mongo/db/storage/wiredtiger/wiredtiger_util.cpp 409
2020-04-21T10:12:44.839+0800 F -        [WTCheckpointThread] 

***aborting after fassert() failure
```



尝试使用 mongod 自带的恢复模式修复数据：

```bash
mongod -f /etc/mongod.conf --repair
```

未修复成功，  因此可判定是进程异常终止导致数据的元文件损坏，以至于服务无法启动。

解决方案，**重建损坏的元数据**。



备份db数据, 然后然后在备份的数据库上修复

```bash
cp /var/lib/mongo /var/lib/mongo-2020-04-21
```

通过 WiredTiger 的 `wt`工具可以将其提取出来。默认 MongoDB 的源码里不会编译出 wt 工具，因此需要[自行下载 WiredTiger 源码编译](http://source.wiredtiger.com/)

```bash

# 下载
wget http://source.wiredtiger.com/releases/wiredtiger-3.2.1.tar.bz2

# 解压缩 & 安装
tar xvf wiredtiger-3.2.1.tar.bz2
cd wiredtiger-3.2.1

# 安装依赖的组件
yum install snappy snappy-devel

# 编译安装
./configure --enable-snappy
make
```



> snappy是必须要安装的插件，不然无法正常解码二进制数据问题，Ubuntu 系统对应的命令：
>
> ```bash
> sudo apt-get install libsnappy-dev build-essential
> ```



开始使用 wt 工具修复数据：

```bash
# 打捞数据 （此操作会重写wt文件里面的内容）
./wt -v -h /var/lib/mongo-2020-04-21 -C "extensions=[./ext/compressors/snappy/.libs/libwiredtiger_snappy.so]" -R salvage collection-10--749927511012856183.wt


# 使用wt工具里面的dump命令把里面的数据进行导出
./wt -v -h /var/lib/mongo-2020-04-21 -C "extensions=[./ext/compressors/snappy/.libs/libwiredtiger_snappy.so]" -R dump -f ../collection.dump collection-10--749927511012856183


# 重建数据
./wt -v -h /var/lib/mongo-2020-04-21 -C "extensions=[./ext/compressors/snappy/.libs/libwiredtiger_snappy.so]" -R load -f ../collection.dump -r collection-10--749927511012856183
```

> 其中 mongo-2020-04-21 是 db 文件的目录，`collection-10--749927511012856183.wt`就是需要恢复的wt文件

修改 mongod.conf 配置文件，将db 路径指定到 mongo-2020-04-21，使用 mongod 恢复模式修复一把：

```bash
mongod -f /etc/mongod.conf --repair
```

修复成功后即可正常方式启动

```bash
mongod -f /etc/mongod.conf
```

