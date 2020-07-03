# 删除复制集oplog数据后启动不起来

28001.conf

```
port=28001
bind_ip= 127.0.0.1
logpath=/usr/local/mongodb/log/28001.log
dbpath=/usr/local/mongodb/data/28001/
logappend=true
pidfilepath=/usr/local/mongodb/data/28001/28001.pid
oplogSize=1024
replSet=SMQK
```



28002.conf

```
port=28002
bind_ip= 127.0.0.1
logpath=/usr/local/mongodb/log/28002.log
dbpath=/usr/local/mongodb/data/28002/
logappend=true
pidfilepath=/usr/local/mongodb/data/28002/28002.pid
oplogSize=1024
replSet=SMQK
```



28003.conf

```
port=28003
bind_ip= 127.0.0.1
logpath=/usr/local/mongodb/log/28003.log
dbpath=/usr/local/mongodb/data/28003/
logappend=true
pidfilepath=/usr/local/mongodb/data/28003/28003.pid
oplogSize=1024
replSet=SMQK
```



> 复制集名称必须相同
>



mongo 127.0.0.1:28001
不指定数据库默认进入 test 数据库

```
// 注意  _id 必须时 配置文件中的  replSet 值
mongo 127.0.0.1:28001/admin
config = {
	_id:"SMQK",
	members:[
	{_id:0, host:"127.0.0.1:28001"},
	{_id:1, host:"127.0.0.1:28002"},
	{_id:2, host:"127.0.0.1:28003", arbiterOnly: true} 
	]
}
rs.initiate(config)
```





Arbiter 仲裁

```
// 不开启的话无法读取从节点的数据
rs.slaveOK(true)
```



1,  3个节点服务都停掉;
2,  清空从节点及仲裁节点的 data 数据(没有数据即空服务);
3,  主要节点去掉复制集配置单节点启动然后删除 local 数据库然后停掉;
4,  主节点添加复制集配置启动;
5,  从节点, 仲裁节点响应启动;
6,  mongo 链主节点进行复制集配置 & 初始化;