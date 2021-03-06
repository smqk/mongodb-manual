

# 批量操作（bulkWrite）

## 定义

**db.collection.bulkWrite()**

提供可控执行顺序的批量写操作。

语法格式如下：

```javascript
db.collection.bulkWrite(
   [ <operation 1>, <operation 2>, ... ],
   {
      writeConcern : <document>,
      ordered : <boolean>
   }
)
```

| 参数           | 类型     | 描述                                                         |
| -------------- | -------- | ------------------------------------------------------------ |
| **operations** | array    | bulkWrite() 写操作的数组。支持操作：insertOne、updateOne、updateMany、deleteOne、deleteMany、replaceOne |
| **writeConcern**   | document | 可选， [write concern](https://docs.mongodb.com/v3.4/reference/write-concern/) 文档，省略则使用默认的 write concern。 |
| **ordered**    | boolean  | 可选，表示mongod实例有序还是无序执行操作。默认值true。       |

方法返回值：

- 操作基于 write concern 运行则 acknowledged 值为true，如果禁用 write concern 运行则 acknowledged 值为false。
- 每一个写操作数。
- 成功 inserted 或 upserted文档的 _id 的组数。



## 行为

bulkWrite() 接收一个写操作的数组然后执行它们中的每一个。默认是有序的执行。



## 写操作

### insertOne

插入单个文档到集合中。

```javascript
db.collection.bulkWrite( [
   { insertOne : { "document" : <document> } }
] )
```

### updateOne 及 updateMany

updateOne 更新集合中 filter 匹配的单个文档。如果匹配到多个文档 updateOne 仅更新第一个匹配到的文档。

```javascript
db.collection.bulkWrite( [
   { updateOne :
      {
         "filter" : <document>,
         "update" : <document>,
         "upsert" : <boolean>
      }
   }
] )
```

updateMany 更新集合中所有匹配到的文档。

```javascript
db.collection.bulkWrite( [
   { updateMany :
      {
         "filter" : <document>,
         "update" : <document>,
         "upsert" : <boolean>
      }
   }
] )
```

对字段的更新操作例如 $set 、$unset 、$rename等。

默认情况 upsert 为 false。

### replaceOne

replaceOne 替换集合中 filter 匹配到的单个文档。如果匹配到多个文档 replaceOne 只会替换一个匹配到的文档。

```javascript
db.collection.bulkWrite([
   { replaceOne :
      {
         "filter" : <document>,
         "replacement" : <document>,
         "upsert" : <boolean>
      }
   }
] )
```

replacement 字段中不能包含 update 操作。

默认情况 upsert 为 false。

### deleteOne 及 deleteMany

deleteOne 删除集合中 filter 匹配到的单个文档。如果匹配到多个文档 deleteOne 只会删除一个匹配到的文档。

```javascript
db.collection.bulkWrite([
   { deleteOne :  { "filter" : <document> } }
] )
```

deleteMany 删除集合中 filter 匹配到的所有文档。

```javascript
db.collection.bulkWrite([
   { deleteMany :  { "filter" : <document> } }
] )
```



## _id 字段

如果文档未指定 _id 字段，则mongod会在 insert 或 upsert 文档之前添加 _id 字段并指定唯一的ObjectId。 大多数驱动程序会创建一个ObjectId并插入到 _id 字段，但如果驱动程序或应用程序没有，mongod将创建并填充 _id。

如果文档包含 _id 字段，则 _id 值在集合中必须是唯一的，以避免重复键错误。

更新或替换操作不能指定与原始文档不同的 _id 值。



## 执行操作

ordered 参数指定  bulkWrite() 是否有序执行，默认情况下是有序执行。

含有6个操作的 bulkWrite()  代码如下：

```javascript
db.collection.bulkWrite(
   [
      { insertOne : <document> },
      { updateOne : <document> },
      { updateMany : <document> },
      { replaceOne : <document> },
      { deleteOne : <document> },
      { deleteMany : <document> }
   ]
)
```

默认情况下 ordered : true ,每个操作将会有序的执行，从第一个insertOne 到最后一个deleteMany 顺序执行。

应用程序不依赖操作执行顺序是，可以设置 ordered 为 false ，此时mongod 会重新排序操作来提高性能。

含有6个操作无序的 bulkWrite()  代码如下：

```javascript
db.collection.bulkWrite(
   [
      { insertOne : <document> },
      { updateOne : <document> },
      { updateMany : <document> },
      { replaceOne : <document> },
      { deleteOne : <document> },
      { deleteMany : <document> }
   ],
   { ordered : false }
)
```

对于ordered：false，操作结果可能会有所不同。 例如，deleteOne或deleteMany 删除的文档可能会变多或变少，具体取决于deleteOne或deleteMany 是在insertOne，updateOne，updateMany或replaceOne操作之前或之后的运行。

每组操作最多可以有1000次操作。 如果一个组超过此[限制](https://docs.mongodb.com/v3.4/reference/limits/#Write-Command-Operation-Limit-Size)，MongoDB会将该组划分为1000或更小的组。 例如，如果队列包含2000个操作，MongoDB将创建2个组，每个组具有1000个操作。

大小和分组机制是内部的执行细节，在将来的版本中可能会有所变化。

在分片集合上执行有序操作通常比执行无序操作慢，因为对于有序，每个操作必须等待上一个操作完成。



## 固定集合（Capped Collections）

bulkWrite() 写操作在固定集合上使用有所限制。

- updateOne 和 updateMany 更新时增加了被修改文档的大小将会抛出 WriteError
- replaceOne 操作替换的文档比之前的文档大会抛出 WriteError
- deleteOne 和 deleteMany 操作在固定集合上会抛出 WriteError



## 操作处理（Error Handling）

bulkWrite() 在错误发生时会抛出 BulkWriteError 异常。

排除Write Concern错误，有序操作在发生错误后停止，及无序操作继续处理队列中的剩余写入操作。

Write Concern 错误显示在 writeConcernErrors字段中，而所有其他错误都显示在writeErrors字段中。 如果遇到错误，则显示成功写入操作的数量而不是插入的_id值。 有序操作显示遇到的单个错误，而无序操作显示数组中的每个错误。



## 实例

### 批量写

characters 集合包含以下文档：

```javascript
{ "_id" : 1, "char" : "Brisbane", "class" : "monk", "lvl" : 4 },
{ "_id" : 2, "char" : "Eldon", "class" : "alchemist", "lvl" : 3 },
{ "_id" : 3, "char" : "Meldane", "class" : "ranger", "lvl" : 3 }
```

bulkWrite() 在集合上执行批量操作：

```javascript
try {
   db.characters.bulkWrite(
      [
         { insertOne :
            {
               "document" :
               {
                  "_id" : 4, "char" : "Dithras", "class" : "barbarian", "lvl" : 4
               }
            }
         },
         { insertOne :
            {
               "document" :
               {
                  "_id" : 5, "char" : "Taeln", "class" : "fighter", "lvl" : 3
               }
            }
         },
         { updateOne :
            {
               "filter" : { "char" : "Eldon" },
               "update" : { $set : { "status" : "Critical Injury" } }
            }
         },
         { deleteOne :
            { "filter" : { "char" : "Brisbane"} }
         },
         { replaceOne :
            {
               "filter" : { "char" : "Meldane" },
               "replacement" : { "char" : "Tanys", "class" : "oracle", "lvl" : 4 }
            }
         }
      ]
   );
}
catch (e) {
   print(e);
}
```

操作结果如下：

```javascript
{
   "acknowledged" : true,
   "deletedCount" : 1,
   "insertedCount" : 2,
   "matchedCount" : 2,
   "upsertedCount" : 0,
   "insertedIds" : {
      "0" : 4,
      "1" : 5
   },
   "upsertedIds" : {

   }
}
```

如果 第二个 insertOne 操作的 _id 是集合中已经存在的，则会抛出以下错误：

```javascript
BulkWriteError({
   "writeErrors" : [
      {
         "index" : 0,
         "code" : 11000,
         "errmsg" : "E11000 duplicate key error collection: guidebook.characters index: _id_ dup key: { : 4 }",
         "op" : {
            "_id" : 5,
            "char" : "Taeln"
         }
      }
   ],
   "writeConcernErrors" : [ ],
   "nInserted" : 1,
   "nUpserted" : 0,
   "nMatched" : 0,
   "nModified" : 0,
   "nRemoved" : 0,
   "upserted" : [ ]
})
```

默认情况下 ordered 为 true， 顺序执行时遇到错误就停止执行（后续的操作不会被执行）。

### 无序批量写

characters 集合包含以下文档：

```javascript
{ "_id" : 1, "char" : "Brisbane", "class" : "monk", "lvl" : 4 },
{ "_id" : 2, "char" : "Eldon", "class" : "alchemist", "lvl" : 3 },
{ "_id" : 3, "char" : "Meldane", "class" : "ranger", "lvl" : 3 }
```

bulkWrite() 在集合上执行批量操作：

```javascript
try {
   db.characters.bulkWrite(
         [
            { insertOne :
               {
                  "document" :
                  {
                     "_id" : 4, "char" : "Dithras", "class" : "barbarian", "lvl" : 4
                  }
               }
            },
            { insertOne :
               {
                  "document" :
                     {
                        "_id" : 4, "char" : "Taeln", "class" : "fighter", "lvl" : 3
                     }
               }
            },
            { updateOne :
               {
                  "filter" : { "char" : "Eldon" },
                  "update" : { $set : { "status" : "Critical Injury" } }
               }
            },
            { deleteOne :
               { "filter" : { "char" : "Brisbane"} }
            },
            { replaceOne :
               {
                  "filter" : { "char" : "Meldane" },
                  "replacement" : { "char" : "Tanys", "class" : "oracle", "lvl" : 4 }
               }
            }
         ],
            { ordered : false }
      );
   }
   catch (e) {
   print(e);
}
```

操作结果如下：

```javascript
BulkWriteError({
   "writeErrors" : [
      {
         "index" : 0,
         "code" : 11000,
         "errmsg" : "E11000 duplicate key error collection: guidebook.characters index: _id_ dup key: { : 4 }",
         "op" : {
            "_id" : 4,
            "char" : "Taeln"
         }
      }
   ],
   "writeConcernErrors" : [ ],
   "nInserted" : 1,
   "nUpserted" : 0,
   "nMatched" : 2,
   "nModified" : 2,
   "nRemoved" : 1,
   "upserted" : [ ]
})
```

无序操作，尽管操作过程中出现错误，剩余的操作也不会就此终止执行。

### 基于 Write Concern 的批量写

enemies 集合包含以下文档：

```javascript
{ "_id" : 1, "char" : "goblin", "rating" : 1, "encounter" : 0.24 },
{ "_id" : 2, "char" : "hobgoblin", "rating" : 1.5, "encounter" : 0.30 },
{ "_id" : 3, "char" : "ogre", "rating" : 3, "encounter" : 0.2 },
{ "_id" : 4, "char" : "ogre berserker" , "rating" : 3.5, "encounter" : 0.12}
```

以下使用 write concern 值为 "majority" 及 timeout 为 100 毫秒来执行批量写操作：

```javascript
try {
   db.enemies.bulkWrite(
      [
         { updateMany :
            {
               "filter" : { "rating" : { $gte : 3} },
               "update" : { $inc : { "encounter" : 0.1 } }
            },

         },
         { updateMany :
            {
               "filter" : { "rating" : { $lt : 2} },
               "update" : { $inc : { "encounter" : -0.25 } }
            },
         },
         { deleteMany : { "filter" : { "encounter" { $lt : 0 } } } },
         { insertOne :
            {
               "document" :
                  {
                     "_id" :5, "char" : "ogrekin" , "rating" : 2, "encounter" : 0.31
                  }
            }
         }
      ],
      { writeConcern : { w : "majority", wtimeout : 100 } }
   );
}
catch (e) {
   print(e);
}
```

如果副本集中所有必需节点确认写入操作所需的总时间大于wtimeout，则在wtimeout 时间过去时将显示以下writeConcernError。

```javascript
BulkWriteError({
   "writeErrors" : [ ],
   "writeConcernErrors" : [
      {
         "code" : 64,
         "errInfo" : {
            "wtimeout" : true
         },
         "errmsg" : "waiting for replication timed out"
      }
   ],
   "nInserted" : 1,
   "nUpserted" : 0,
   "nMatched" : 4,
   "nModified" : 4,
   "nRemoved" : 1,
   "upserted" : [ ]
   })
```

结果集显示执行的操作，因为writeConcernErrors错误不是任何写入操作失败的标志。



https://docs.mongodb.com/v3.4/reference/method/db.collection.bulkWrite

