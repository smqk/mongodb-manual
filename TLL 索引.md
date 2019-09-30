### MongoDB 自动删除集合中过期的数据——TTL索引



### 简介

​		**TTL** (Time To Live, 有生命周期的) 索引是特殊单字段索引，MongoDB可以用来在一定时间后自动从集合中删除文档的特殊索引。 这对于某些类型的数据非常好，例如机器生成的事件数据，日志和会话信息，这些信息只需要在数据库中保留一段时间。

​		创建 TTL 索引，只需要在使用   [`db.collection.createIndex()`](https://docs.mongodb.com/manual/reference/method/db.collection.createIndex/#db.collection.createIndex) 方法，对字段值为日期或者包含日期的数组设置 expireAfterSeconds 选项即可。

> 1、如果字段是一个数组,并有多个日期值时,MongoDB使用最低(即最早)日期值来计算失效阈值。
>
> 2、如果字段不是日期类型也不是一个包含日期的数组类型那么文档就永远不会过期。
>
> 3、如果一个文档不包含索引字段,该文档也不会到期。



### 示例

```js
// 创建一个 TTL 索引
db.eventlog.createIndex( { "lastModifiedDate": 1 }, { expireAfterSeconds: 600 } )

// 修改TTL索引的过期时间
db.runCommand( { collMod: "eventlog",
                 index: { keyPattern: { lastModifiedDate: 1 },
                          expireAfterSeconds: 3600
                        }
})
```



### 删除操作

过期数据的删除工作是由 mongod 后台线程来执行，每60秒进行一次。

对于复制集情况， TTL 后台线程只会删除主节点上过期的数据，从节点过期文档删除则依赖主节点（从节点的 TTL 后台线程是停止状态）。





### 索引区别

对于查询而言，TTL 索引和其他索引没有区别。





### TTL索引有哪些限制

- 单字段索引，复合索引时 expireAfterSeconds 会别忽略掉。
- _id 字段不支持减  TTL 索引。
-  不能在固定集合上创建 TTL 索引，因为固定集合不支持删除操作。
- 不能使用 createIndex() 去修改一个已存在索引的 expireAfterSeconds 。相反，可以使用collMod 命令来修改。否则,改变现有索引的选项的值,你必须删除索引,重新创建。
- 如果一个非TTL单字段索引字段已经存在,您不能再创建一个同字段的TTL索引,因为不能创建相同键不同选项的多个索引。改变非TTL单字段索引成TTL索引,首先你必须先删除索引再现expireAfterSeconds选项创建索引。



官网文档：

https://docs.mongodb.com/manual/core/index-ttl/index.html



> 个人博客：学习园
> 原文链接： [https://xuexiyuan.cn/article/detail/225.html](https://xuexiyuan.cn/article/detail/225.html)

