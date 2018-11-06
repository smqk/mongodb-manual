# 查询计划（Query Plans）

在给定可用索引的情况下，MongoDB查询优化器处理查询并且选择出针对某查询而言最高效的查询计划。每次查询执行的时候，查询系统都会使用该查询计划。

查询优化器仅缓存那些有多个可行计划的 [query shape](https://docs.mongodb.com/v3.4/reference/glossary/#term-query-shape)。

对于每个查询，查询计划程序在查询计划缓存中搜索适合 query shape 的查询计划。如果没有匹配到合适的查询计划，则查询计划程序会生成候选计划，以便在试用期内进行评估。查询计划程序选择获胜计划，创建包含获胜计划的缓存条目，并使用它来生成结果文档。

如果存在匹配条目，则查询计划程序将根据该条目生成计划，并通过重新计划机制评估其性能。 此机制根据计划性能进行通过/失败决策，并保留或逐出缓存条目。 在逐出时，查询计划程序使用常规计划过程选择新计划并对其进行缓存。 查询计划程序执行计划并返回查询的结果文档。

下图阐述查询计划逻辑：

![](images/query-planner-diagram.bakedsvg.svg)



有关触发计划缓存更改的其他方案，请参阅[计划缓存刷新](https://docs.mongodb.com/v3.4/core/query-plans/#query-plans-plan-cache-flushes)。

你可以使用 db.collection.explain() 或 cursor.explain() 方法展示给定查询的查询计划相关统计信息。在制定[索引策略](https://docs.mongodb.com/v3.4/applications/indexes/)时，此信息可以提供帮助。

在2.6版中更改：explain() 操作不再读取或写入查询计划程序缓存。



## 计划缓存刷新（Plan Cache Flushes）

索引或集合等目录操作会刷新计划缓存。

mongod 重启或关闭计划缓存（会丢失）将不会被留存。

新版本2.6中：MongoDB 提供 [查询计划缓存方法](https://docs.mongodb.com/v3.4/reference/method/js-plan-cache/)显示及修改已缓存的计划。 [PlanCache.clear()](https://docs.mongodb.com/v3.4/reference/method/PlanCache.clear/#PlanCache.clear) 方法刷新所有的计划缓存。用户也可以使用 [PlanCache.clearPlansByQuery()](https://docs.mongodb.com/v3.4/reference/method/PlanCache.clearPlansByQuery/#PlanCache.clearPlansByQuery) 清除指定的计划缓存



## 索引过滤器（Index Filters）

索引过滤器确定优化程序为 query shape 评估的索引。query shape 由 query，sort 和 projection 的组合组成。 对于给定 query shape 如果存在索引过滤器，则优化程序仅考虑过滤器中指定的那些索引。

当 query shape 存在索引过滤器时，MongoDB 会忽略 hint()。查询MongoDB为 query shape 是否使用了索引过滤器，检查 db.collection.explain() 或 cursor.explain() 方法的 indexFilterSet (boolean 类型)字段。

索引过滤器仅影响优化程序评估的索引; 优化器仍然可以选择集合扫描作为给定 query shape 的获胜计划。

索引过滤器在服务器进程的持续时间内存在，并且在关闭后不会保留。 MongoDB还提供了手动删除过滤器的命令。

由于索引过滤器会覆盖优化程序的预期行为以及hint() 方法，因此请谨慎使用索引过滤器。



## planCacheListFilters

列出与集合 query shape 有关联的索引过滤器。语法格式如下:

```javascript
db.runCommand( { planCacheListFilters: <collection_name> } )
```

运行这个命令用户必须要有 planCacheIndexFilter 操作的权限。命令的输出信息如下：

```javascript
{
   "filters" : [
      {
         "query" : <query>
         "sort" : <sort>,
         "projection" : <projection>,
         "indexes" : [
            <index1>,
            ...
         ]
      },
      ...
   ],
   "ok" : 1
}
```

- filters 包含索引过滤器信息的文档数组。

- filters.query  与此过滤器关联的查询谓词。 虽然查询显示了用于创建索引过滤器的特定值，但谓词中的值无关紧要; 即，查询谓词包括仅在值不同的类似查询。

  例如，对于 { "type": "electronics", "status" : "A" } 的查询谓词等同如下查询谓词：

  ```javascript
  { type: "food", status: "A" }
  { type: "utensil", status: "D" }
  ```

  sort 、projection 和 find 一起构成了索引过滤器的 query shape。

- filters.sort 与此过滤器关联的 sort，可以是一个空文档。

- filters.projection 与此过滤器关联的 projection，可以是一个空文档。

- filters.indexes 此 query shape 的索引数组。查询优化程序仅评估列出的索引和集合扫描，从中选择最佳查询计划。

- ok 命令执行状态。



## planCacheClearFilters

删除集合上的索引过滤器。尽管索引过滤器仅存在于服务器进程中重启进程后不保留，仍可以使用 planCacheClearFilters 命令删除索引过滤器。

指定 query shape 删除指定的索引过滤器。或者忽略 query shape 删除集合上所有的索引过滤器。命令语法格式如下：

```javascript
db.runCommand(
   {
      planCacheClearFilters: <collection_name>,
      query: <query pattern>,
      sort: <sort specification>,
      projection: <projection specification>
   }
)
```

planCacheClearFilters string 类型，其他的 query、sort、projection 都是 document 类型。

运行这个命令用户必须要有 planCacheIndexFilter 操作的权限。

### 在集合上删除指定的索引过滤器

orders 集合包含如下两个索引过滤器：

```javascript
{
  "query" : { "status" : "A" },
  "sort" : { "ord_date" : -1 },
  "projection" : { },
  "indexes" : [ { "status" : 1, "cust_id" : 1 } ]
}

{
  "query" : { "status" : "A" },
  "sort" : { },
  "projection" : { },
  "indexes" : [ { "status" : 1, "cust_id" : 1 } ]
}
```

如下命令仅删除第二个索引过滤器：

```javascript
db.runCommand(
   {
      planCacheClearFilters: "orders",
      query: { "status" : "A" }
   }
)
```

由于查询谓词中的值在确定query shape 时无关紧要，因此以下命令（等同于上）也会删除第二个索引过滤器：

```javascript
db.runCommand(
   {
      planCacheClearFilters: "orders",
      query: { "status" : "P" }
   }
)
```

### 删除集合上所有的索引过滤器

```javascript
db.runCommand(
   {
      planCacheClearFilters: "orders"
   }
)
```



## planCacheSetFilter

为集合设置索引过滤器。对于 query shape 已存在索引过滤器则此命令会重写之前的索引过滤器。命令语法如下：

```javascript
db.runCommand(
   {
      planCacheSetFilter: <collection>,
      query: <query>,
      sort: <sort>,
      projection: <projection>,
      indexes: [ <index1>, <index2>, ...]
   }
)
```

索引过滤器仅存在于服务器进行运行时，进程关闭以后不予以保留。运行这个命令用户必须要有 planCacheIndexFilter 操作的权限。

### 在仅包含谓词的 query shape 上设置过滤器

以下示例在orders集合仅包含status字段上的相等匹配但没有任何 projection 和 sort 的查询上创建索引过滤器，以便对于，查询优化器仅评估两个指定的索引以及获胜计划的集合扫描：

```javascript
db.runCommand(
   {
      planCacheSetFilter: "orders",
      query: { status: "A" },
      indexes: [
         { cust_id: 1, status: 1 },
         { status: 1, order_date: -1 }
      ]
   }
)
```

在查询谓词中，只有谓词的结构（包括字段名称）是重要的; 值是无关紧要的。 因此，创建的过滤器适用于以下操作：

```javascript
db.orders.find( { status: "D" } )
db.orders.find( { status: "P" } )
```

### 包括谓词，投影和排序的 query shape 上设置过滤器

以下示例为orders集合上谓词在item字段上相等匹配的查询，其中仅投影 quantity字段，并指定按 order_date 升序排序的查询创建索引过滤器。

```javascript
db.runCommand(
   {
      planCacheSetFilter: "orders",
      query: { item: "ABC" },
      projection: { quantity: 1, _id: 0 },
      sort: { order_date: 1 },
      indexes: [
         { item: 1, order_date: 1 , quantity: 1 }
      ]
   }
)
```



译文地址： https://docs.mongodb.com/v3.4/core/query-plans/