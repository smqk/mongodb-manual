MongoDB 提供 db.collection.explain(),  cursort.explain() 及 explain 命令返回查询计划及查询计划执行统计信息。

expalin 结果将查询计划呈现为阶段树。 每个阶段将其结果（文档或索引键）传递给父节点。 叶节点访问集合或索引。 内部节点操纵由子节点产生的文档或索引键。 根节点是MongoDB从中派生结果集的最后阶段。

阶段操作描述,例：

- **COLLSCAN**  集合扫描
- **IXSCAN**  索引扫描
- **FETCH**  检出文档
- **SHARD_MERGE**  合并分片中结果
- **SHARDING_FILTER**  分片中过滤掉孤立文档
- **LIMIT** 使用limit 限制返回数
- **PROJECTION** 使用 skip 进行跳过
- **IDHACK** 针对_id进行查询
- **COUNT** 利用db.coll.explain().count()之类进行count运算
- **COUNTSCAN**  count不使用Index进行count时的stage返回
- **COUNT_SCAN**  count使用了Index进行count时的stage返回
- **SUBPLA**  未使用到索引的$or查询的stage返回
- **TEXT**  使用全文索引进行查询时候的stage返回
- **PROJECTION**  限定返回字段时候stage的返回
- ...



##  explain 操作返回结果

### queryPlanner

queryPlanner 信息显示的是被查询优化器选择出来的查询计划。

以下未分片集合 explain 操作结果如下：

```javascript
{
   "queryPlanner" : {
      "plannerVersion" : <int>,
      "namespace" : <string>,
      "indexFilterSet" : <boolean>,
      "parsedQuery" : {
         ...
      },
      "winningPlan" : {
         "stage" : <STAGE1>,
         ...
         "inputStage" : {
            "stage" : <STAGE2>,
            ...
            "inputStage" : {
               ...
            }
         }
      },
      "rejectedPlans" : [
         <candidate plan 1>,
         ...
      ]
   }
```

- queryPlanner.namespace  一个字符串，指定运行查询的命名空间（即<database>.<collection>）。
- queryPlanner.indexFilterSet  boolan值，表示MongoDB 对于此query shape 是否使用了索引过滤器。
- queryPlanner.winningPlan  文档类型，详细显示查询优化程序选择的查询计划。
  - winningPlan.stage 阶段名称。每个阶段都有每个阶段特有的信息。 例如，IXSCAN 阶段将包括索引边界以及特定于索引扫描的其他数据。 如果阶段具有子阶段或多个子阶段，则阶段将具有inputStage 或 inputStages。
  - winningPlan.inputStage 描述子阶段的文档。它为其父级提供文档或索引键。 如果父级只有一个子级，则该字段存在。
  - winningPlan.inputStages  描述子阶段的数组。子阶段为父阶段提供文档或索引键。 如果父级具有多个子节点，则该字段存在。 例如，$or 表达式或索引交集的阶段消耗来自多个源的输入。
- queryPlanner.rejectedPlans 查询优化器考虑和拒绝的候选计划数组。 如果没有其他候选计划，则该数组可以为空。

对于分片集合，获胜计划包括分每个访问的分片的计划信息数组。



### executionStats

executionStats 返回的是获胜计划执行相关信息。必须在 executionStats 或 allPlansExecution 详细模式下运行explain才显示executionStats 相关信息。

要捕获选择执行计划期间执行相关信息，必须使用 allPlansExecution 模式运行。

executionStats 模式 explain 实例：

```javascript
db.products.explain("executionStats").find(
   { quantity: { $gt: 50 }, category: "apparel" }
)
```

allPlansExecution 模式 explain 实例：

```javascript
db.products.explain("allPlansExecution").update(
   { quantity: { $lt: 1000}, category: "apparel" },
   { $set: { reorder: true } }
)
```

以下未分片集合 explain 操作结果如下：

```javascript
"executionStats" : {
   "executionSuccess" : <boolean>,
   "nReturned" : <int>,
   "executionTimeMillis" : <int>,
   "totalKeysExamined" : <int>,
   "totalDocsExamined" : <int>,
   "executionStages" : {
      "stage" : <STAGE1>
      "nReturned" : <int>,
      "executionTimeMillisEstimate" : <int>,
      "works" : <int>,
      "advanced" : <int>,
      "needTime" : <int>,
      "needYield" : <int>,
      "isEOF" : <boolean>,
      ...
      "inputStage" : {
         "stage" : <STAGE2>,
         ...
         "nReturned" : <int>,
         "executionTimeMillisEstimate" : <int>,
         "keysExamined" : <int>,
         "docsExamined" : <int>,
         ...
         "inputStage" : {
            ...
         }
      }
   },
   "allPlansExecution" : [
      { <partial executionStats1> },
      { <partial executionStats2> },
      ...
   ]
}
```

- executionStats  描述获胜计划完整的查询执行统计信息。 对于写入操作，是指将要执行修改的统计信息，但不会真实修改数据库。
  - executionStats.nReturned  查询条件匹配到的文档数量。
  - executionStats.executionTimeMillis  查询计划选择和查询执行所需的总时间（以毫秒为单位）。executionTimeMillis 对应于早期版本的MongoDB中cursor.explain() 返回的millis字段。
  - executionStats.totalKeysExamined  扫描的索引条目数。totalKeysExamined 对应于早期版本的MongoDB中cursor.explain() 返回的 nscanned字段。
  - executionStats.totalDocsExamined  扫描的文档数。totalDocsExamined 对应于早期版本的MongoDB中cursor.explain() 返回的 nscannedObjects字段。
  - executionStats.executionStages  阶段树形式展示获胜计划完整的执行信息。即，一个阶段可以有一个inputStage或多个inputStages。
    - executionStats.executionStages.works  查询执行阶段执行的“工作单元”的数量。 查询执行阶段将其工作分为小单元。 “工作单位”可能包括检查单个索引键，从集合中提取单个文档，将投影应用于单个文档或执行内部记账。

    - executionStats.executionStages.advanced  返回到父阶段的结果数。

    - executionStats.executionStages.needTime  未将中间结果返回给其父级的工作循环数。 例如，索引扫描阶段可以花费一个工作周期来寻找索引中的新位置而不是返回索引关键字; 这个工作周期将计入explain.executionStats.executionStages.needTime而不是explain.executionStats.executionStages.advanced。

    - executionStats.executionStages.needYield  存储层请求查询系统产生锁定的次数。

    - executionStats.executionStages.isEOF  执行阶段是否已到达流的结尾：

      - 如果为true或1，则执行阶段已到达流末尾。
      - 如果为false或0，则阶段可能仍会返回结果。 例如，有限制的查询，其执行阶段包含LIMIT阶段，其中查询的输入阶段为IXSCAN。 如果查询返回超过指定的限制，LIMIT阶段将报告isEOF：1，但其基础IXSCAN阶段将报告isEOF：0。

    - executionStats.executionStages.inputStage.keysExamined 对于扫描索引的查询执行阶段（例如IXSCAN），keysExamined是在索引扫描过程中检查的入站和越界键的总数。 如果索引扫描由单个连续范围的键组成，则只需要检查入站键。 如果索引边界由若干键范围组成，则索引扫描执行过程可以检查越界键，以便从一个范围的末尾跳到下一个范围的开头。

      考虑以下示例，集合包含字段x值为1到100的100个文档，其中索引字段为x：

      ```javascript
      for(var x=1;x<=100;x++){
          db.keys.insert({x:x});
      }
      db.keys.ensureIndex({x:1});
      db.keys.find( { x : { $in : [ 3, 4, 50, 74, 75, 90 ] } } ).explain( "executionStats" )
      ```

      该查询将扫描键3和4.然后它将扫描键5，检测它是否超出界限，并跳到下一个键50。

      继续该过程，查询扫描键3,4,5,50,51,74,75,76,90和91.键5,51,76和91是仍在检查的越界键。 keysExamined的值为10。

    - executionStats.executionStages.inputStage.docsExamined  在查询执行阶段扫描的文档数。

    - executionStats.allPlansExecution  包含在计划选择阶段为获胜和拒绝计划捕获的部分执行信息。 仅当explain在allPlansExecution详细模式下运行时，该字段才会出现。



### serverInfo

对于未分片的集合，MongoDB实例explain返回以下信息：

```javascript
"serverInfo" : {
   "host" : <string>,
   "port" : <int>,
   "version" : <string>,
   "gitVersion" : <string>
}
```

对于分片集合，explain返回每个访问的分片的serverInfo。 



### Sharded Collection

对于分片集合，explain在shards字段中返回每个访问的分片的核心查询规划器和服务器信息：

```javascript
{
   "queryPlanner" : {
      ...
      "winningPlan" : {
         ...
         "shards" : [
            {
              "shardName" : <shard>,
              <queryPlanner information for shard>,
              <serverInfo for shard>
            },
            ...
         ],
      },
   },
   "executionStats" : {
      ...
      "executionStages" : {
         ...
         "shards" : [
            {
               "shardName" : <shard>,
               <executionStats for shard>
            },
            ...
         ]
      },
      "allPlansExecution" : [
         {
            "shardName" : <string>,
            "allPlans" : [ ... ]
         },
         ...
      ]
   }
}
```

- queryPlanner.winningPlan.shards  每个访问分片的queryPlanner和serverInfo的文档数组。
- executionStats.executionStages.shards  包含每个访问分片的executionStats的文档数组。



## 兼容性的修改

> 在 3.0 版本中的修改。
>

explain 结果的格式和字段与以前的版本相比有所变化。 以下列出了一些主要差异。

### 集合扫描 VS 索引使用

如果查询计划程序选择了集合扫描，则说明结果包括COLLSCAN阶段。

如果查询计划程序选择索引，则说明结果包括IXSCAN阶段。 该阶段包括诸如索引key的匹配，遍历方向和索引边界之类的信息。

在早期版本的MongoDB中，cursor.explain() 返回了光标字段，其值为：

用于集合扫描的BasicCursor，以及索引扫描的BtreeCursor <索引名称> [<方向>]。


### 覆盖查询

当索引覆盖查询时，MongoDB可以匹配查询条件并仅使用索引键返回结果; 即MongoDB不需要检查集合中的文档以返回结果。

当索引覆盖查询时，解释结果的IXSCAN阶段不是FETCH阶段的后代，而在executionStats中，totalDocsExamined为0。

在早期版本的MongoDB中，cursor.explain() 返回indexOnly字段以指示索引是否覆盖了查询。



### 索引交叉

对于索引交集计划，结果将包括AND_SORTED阶段或AND_HASH阶段，其中包含详细说明索引的inputStages数组; 例如。：

```javascript
{
   "stage" : "AND_SORTED",
   "inputStages" : [
      {
         "stage" : "IXSCAN",
         ...
      },
      {
         "stage" : "IXSCAN",
         ...
      }
   ]
}
```

在以前版本的MongoDB中，cursor.explain() 返回了游标字段，其中包含索引交叉点的复杂计划。



### $or 表达式

如果MongoDB使用$or 表达式的索引，则结果将包含OR阶段，其中包含详细说明索引的inputStages数组; 例如。：

```javascript
{
   "stage" : "OR",
   "inputStages" : [
      {
         "stage" : "IXSCAN",
         ...
      },
      {
         "stage" : "IXSCAN",
         ...
      },
      ...
   ]
}
```

在早期版本的MongoDB中，cursor.explain() 返回了详细索引的子句数组。



### Sort 阶段

如果MongoDB可以使用索引扫描来获取请求的排序顺序，则结果将不包括SORT阶段。 否则，如果MongoDB无法使用索引进行排序，则解释结果将包括SORT阶段。

在MongoDB 3.0之前，cursor.explain() 返回scanAndOrder字段以指定MongoDB是否可以使用索引顺序返回排序结果。



## 总结

### explain 希望看到的阶段

Fetch+IDHACK

Fetch+ixscan

Limit+（Fetch+ixscan）

PROJECTION+ixscan

SHARDING_FILTER+ixscan

COUNT_SCAN

...



### explain 不希望看到的阶段

COLLSCAN（全表扫描），

SORT（使用sort但是无index），

不合理的SKIP，

SUBPLA（未用到index的$or），

COUNTSCAN



https://docs.mongodb.com/v3.4/reference/explain-results/