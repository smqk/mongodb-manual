MongoDB 索引简介

在MongoDB中索引支持使得执行查询更高效。 如果没有索引，MongoDB必须执行集合扫描，即扫描集合中的每个文档，以选择与查询语句匹配的文档。如果查询存在适当的索引，MongoDB可以使用索引来限制它必须检查的文档数。

索引是特殊的数据结构[1]，它以易于遍历的形式存储集合数据集的一小部分。 索引存储特定字段或字段集的值，按字段值排序。 索引条目的排序支持有效的等式匹配和基于范围的查询操作。 此外，MongoDB可以使用索引中的顺序返回排序结果。

下图查询实例，使用索引选择和排序匹配文档：

![](https://docs.mongodb.com/manual/_images/index-for-sort.bakedsvg.svg)

从原理上说，MongoDB中的索引与其他数据库系统中的索引类似。 MongoDB在集合级别定义索引，并支持MongoDB集合中文档的任何字段或子字段的索引。

[1]. MongoDB索引使用B-树的数据结构。



## 默认_id索引

MongoDB在创建集合期间在 __id 字段上创建唯一索引。 _id索引可防止客户端为_id字段插入具有相同值的两个文档。 您不能在 _id字段上删除此索引。

> 在分片群集中，如果不使用_id字段作为分片键，则应用程序必须确保_id字段中值的唯一性以防止出错。 这通常通过使用标准的自动生成的ObjectId来完成。



## 创建索引

使用[`db.collection.createIndex()`](https://docs.mongodb.com/manual/reference/method/db.collection.createIndex/#db.collection.createIndex) 在Mongo shell 中创建索引：

```javascript
db.collection.createIndex( <key and index type specification>, <options> )
```

以下示例在name字段上创建单个键降序索引

```javascript
db.collection.createIndex( { name: -1 } )
```

db.collection.createIndex 方法仅在尚不存在相同规范的索引时才创建索引。



## 索引类型

MongoDB提供了许多不同的索引类型来支持特定类型的数据和查询。

### 单字段类型

除MongoDB定义的_id索引外，MongoDB还支持在文档的单个字段上创建用户定义的升序/降序索引。

![](https://docs.mongodb.com/manual/_images/index-ascending.bakedsvg.svg)

对于单字段索引和排序操作，索引键的排序顺序（升序或降序）无关紧要，因为MongoDB可以在任一方向上遍历索引。

有关单字段索引的详细信息，请参阅[单字段索引](https://docs.mongodb.com/manual/core/index-single/) 和[单字段索引排序](https://docs.mongodb.com/manual/tutorial/sort-results-with-indexes/#sort-results-single-field) 。



### 复合索引

MongoDB还支持用户自定义的多个字段索引，即[复合索引](https://docs.mongodb.com/manual/core/index-compound/)。

复合索引中列出的字段顺序具有重要意义。 例如，如果复合索引由{userid：1，score：-1}组成，则索引首先按userid排序，然后在每个userid值内按score排序。

![复合索引](https://docs.mongodb.com/manual/_images/index-compound-key.bakedsvg.svg)

对于复合索引和排序操作，索引键的排序顺序（即升序或降序）可以决定索引是否可以支持排序操作。 有关索引顺序对复合索引中结果的影响的详细信息，请参阅[排序顺序](https://docs.mongodb.com/manual/core/index-compound/#index-ascending-and-descending)。

有关复合索引的详细信息，请参阅[复合索引](https://docs.mongodb.com/manual/core/index-compound/)和[多个字段排序](https://docs.mongodb.com/manual/core/index-compound/)。



### 多键索引

MongoDB使用[多键索引](https://docs.mongodb.com/manual/core/index-multikey/)来索引存储在数组中的内容。 如果索引字段的值是数组，MongoDB会为数组的每个元素创建单独的索引条目。 这些多键索引允许查询通过匹配数组的元素或元素来选择包含数组的文档。 如果索引字段包含数组值，MongoDB会自动确定是否创建多键索引; 您不需要显式指定多键类型。

![多键索引](https://docs.mongodb.com/manual/_images/index-multikey.bakedsvg.svg)

有关多键索引的详细信息，请参阅[多键索引](https://docs.mongodb.com/manual/core/index-multikey/)和[多键索引边界](https://docs.mongodb.com/manual/core/multikey-index-bounds/)。



### 地理空间索引

为了支持对地理空间坐标数据的有效查询，MongoDB提供了两个特殊索引：[2d索引](https://docs.mongodb.com/manual/core/2d/)在返回结果时使用平面几何，而[2dphere索引](https://docs.mongodb.com/manual/core/2dsphere/)使用球形几何返回结果。

有关地理空间索引的高级介绍，请参阅[2d Index Internals](https://docs.mongodb.com/manual/core/geospatial-indexes/)。



### 文本索引

MongoDB提供了一种文本索引类型，支持在集合中搜索字符串内容。 这些文本索引不存储特定于语言的停用词（例如“the”，“a”，“or”），并且阻止集合中的单词仅存储词根。

有关文本索引和搜索的更多信息，请参阅[文本索引](https://docs.mongodb.com/manual/core/index-text/)。



### 散列索引

为了支持[基于散列的分片](https://docs.mongodb.com/manual/core/hashed-sharding/#sharding-hashed-sharding)，MongoDB提供了[散列索引](https://docs.mongodb.com/manual/core/index-hashed/)类型，该类型索引字段值的散列值。 这些索引在其范围内具有更随机的值分布，但仅支持相等匹配，并且不支持基于范围的查询。



## 索引特性

### 唯一索引

索引的[unique](https://docs.mongodb.com/manual/core/index-unique/) 属性会导致MongoDB拒绝索引字段的重复值。 除了唯一约束之外，唯一索引在功能上可与其他MongoDB索引互换。

### 局部索引

[局部索引](https://docs.mongodb.com/manual/core/index-partial/)仅索引符合指定过滤表达式集合中的文档。 通过索引集合中文档的子集，局部索引具有较低的存储要求，并降低了索引创建和维护的性能成本。

### 稀疏索引

索引的[sparse](https://docs.mongodb.com/manual/core/index-sparse/) 属性可确保索引仅包含具有索引字段的文档的条目。 索引会跳过没有索引字段的文档。

您可以将稀疏索引选项与唯一索引选项组合，以拒绝具有字段重复值的文档，但忽略没有索引键的文档。

### TTL索引

TTL(Time To Live, 有生命周期的) 索引是MongoDB可以用来在一定时间后自动从集合中删除文档的特殊索引。 这对于某些类型的信息非常理想，例如机器生成的事件数据，日志和会话信息，这些信息只需要在数据库中保留一段时间。

请参阅：通过[设置TTL从集合中过期数据](https://docs.mongodb.com/manual/tutorial/expire-data/)。



## 索引使用

索引可以提高读取操作的效率。[分析查询性能](https://docs.mongodb.com/manual/tutorial/analyze-query-plan/)教程提供了包含和不包含索引的查询的执行统计信息的示例。

有关MongoDB如何选择要使用的索引的信息，请参阅[查询优化](https://docs.mongodb.com/manual/core/query-plans/#read-operations-query-optimization)。



## 索引和排序规则

要使用索引进行字符串比较，操作还必须指定相同的排序规则。 也就是说，如果操作指定了不同的排序规则，则具有排序规则的索引不支持对索引字段执行字符串比较的操作。

例如，集合 myColl 在字符串字段 category 上具有索引，其中排序规则区域设置为“fr”。

```javascript
db.myColl.createIndex( { category: 1 }, { collation: { locale: "fr" } } )
```

以下查询操作（指定与索引相同的排序规则）可以使用索引：

```javascript
db.myColl.find( { category: "cafe" } ).collation( { locale: "fr" } )
```

但是，以下查询操作（默认情况下使用“简单”二进制比较）无法使用索引：

```javascript
db.myColl.find( { category: "cafe" } )
```

对于索引前缀键不是字符串，数组和嵌入文档的复合索引，指定不同排序规则的操作仍然可以使用索引来支持对索引前缀键的比较。

例如，集合myColl在数字字段 score 和 price 以及字符串字段 category 上有一个复合索引; 使用排序规则区域设置“fr”创建索引以进行字符串比较：

```javascript
db.myColl.createIndex(
   { score: 1, price: 1, category: 1 },
   { collation: { locale: "fr" } } )
```

以下操作使用“简单”二进制排序规则进行字符串比较，可以使用索引：

```javascript
db.myColl.find( { score: 5 } ).sort( { price: 1 } )
db.myColl.find( { score: 5, price: { $gt: NumberDecimal( "10" ) } } ).sort( { price: 1 } )
```

以下操作使用“简单”二进制排序规则对索引 category 字段进行字符串比较，可以使用索引仅满足查询的 score：5部分：

```javascript
db.myColl.find( { score: 5, category: "cafe" } )
```

有关排序规则的详细信息，请参阅[排序规则参考页面](https://docs.mongodb.com/manual/reference/collation/)。

以下索引仅支持简单的二进制比较，不支持排序[规则](https://docs.mongodb.com/manual/reference/bson-type-comparison-order/#collation)：

- [文本](https://docs.mongodb.com/manual/core/index-text/) 索引
- [2d](https://docs.mongodb.com/manual/core/2d/) 索引
- [geoHaystack](https://docs.mongodb.com/manual/core/geohaystack/) 索引



## 覆盖查询

当查询条件和查询的投影都包含在索引字段时，MongoDB直接从索引返回结果，而不扫描任何文档或将文档带入内存。 这些覆盖的查询可以非常有效。

![覆盖查询](https://docs.mongodb.com/manual/_images/index-for-covered-query.bakedsvg.svg)

有关覆盖查询的更多信息，请参阅[覆盖查询](https://docs.mongodb.com/manual/core/query-optimization/#read-operations-covered-query)。

## 索引交集

MongoDB 可以使用索引的交集实现查询。对于复合查询，如果一个索引可以满足查询的一部分，另一个索引可以满足查询条件的另一部分，这时MonoDB会使用这两个索引的交集来完成查询。使用复合索引还是索引交集哪个更高效取决于特性的查询和系统。

有关索引交集的更多信息，请参阅[索引交集](https://docs.mongodb.com/manual/core/index-intersection/)。



## 索引限制

使用索引的某些限制，例如索引键的长度或每个集合的索引数。有关详情信息，请参阅[索引限制](https://docs.mongodb.com/manual/reference/limits/#index-limitations)。

- 一个集合最多只能有64个索引。
- “ 4.0”或更早版本的MongoDB，包括名称空间和点分隔符（即<数据库名称>。<集合名称>。$ <索引名称>）， 不能超过127个字节。
- 复合索引中的字段不能超过32个。
- 查询不能同时使用文本索引和地理空间索引。
- 2dsphere索引的字段只能几何数据。
- 多键索引不能覆盖对数组字段的查询。
- 地理空间索引无法覆盖查询。
- text 、2d、geoHaystack索引类型仅支持简单的二进制比较，不支持排序规则。



## 注意事项

虽然索引可以提高查询性能，但也有一些操作需要注意的事项。有关更多信息，请参阅[索引操作注意事项](https://docs.mongodb.com/manual/core/data-model-operations/#data-model-indexes)。

如果您的集合包含大量数据，并且您的应用程序需要能够在构建索引时访问数据，请考虑在[后台构建索引](https://docs.mongodb.com/manual/core/index-creation/#index-creation-background)。

某些驱动程序可以使用NumberLong(1) 而不是1作为规范来指定索引。 这对结果索引没有任何影响。



## 附录

译自(4.0版)：https://docs.mongodb.com/manual/indexes/

个人博客： [学习园](http://www.xuexiyuan.cn?from=github)

![MongoDB学习园](./images/wechat_mongodb_xuexiyuan.png)











