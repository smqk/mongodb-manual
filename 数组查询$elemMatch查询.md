# $elemMatch

### 描述

$elemMatch 数组查询操作用于查询数组值中至少有一个能完全匹配所有的查询条件的文档。语法格式如下：

```js
{ <field>: { $elemMatch: { <query1>, <query2>, ... } } }
```

> 如果只有一个查询条件就没必要使用 $elemMatch。





### 限制

1. 不能指定 $where 查询条件在 $elemMatch 内；
2. 不能指定 $text 查询条件在 $elemMatch 内；





### 实例

有如下测试数据：

```json
{ _id: 1, results: [ 82, 85, 88 ] }
{ _id: 2, results: [ 75, 88, 89 ] }
```

如下语句用于匹配 results 数组中含有值同时满足大于等于 80 且小于 85 的文档。

```js
db.scores.find(
   { results: { $elemMatch: { $gte: 80, $lt: 85 } } }
)
```

如下查询结果，文档中 results 中 82 满足大于等于 80 且小于85。

```json
{ "_id" : 1, "results" : [ 82, 85, 88 ] }
```





### 数组嵌套文档

例如有如下测试数据：

```json
{ _id: 1, results: [ { product: "abc", score: 10 }, { product: "xyz", score: 5 } ] }
{ _id: 2, results: [ { product: "abc", score: 8 }, { product: "xyz", score: 7 } ] }
{ _id: 3, results: [ { product: "abc", score: 7 }, { product: "xyz", score: 8 } ] }
```

如下语句用于匹配 results 数组中含有值同时满足product 为 “xyz” 且 score 大于等于8 的文档。

```js
db.survey.find(
   { results: { $elemMatch: { product: "xyz", score: { $gte: 8 } } } }
)
```

查询结果如下：

```json
{ _id: 3, results: [ { product: "abc", score: 7 }, { product: "xyz", score: 8 } ] }
```



### 单查询条件

如果只有单个查询条件时没有必要使用 $elemMatch 操作符。例如如下查询：

```js
// 没有必要使用 $elemMatch 操作符
db.survey.find(
   { results: { $elemMatch: { product: "xyz" } } }
)
```

$elemMatch 操作符只有单个查询条件，完全可以使用如下写法代替：

```js
db.survey.find(
   { "results.product": "xyz" }
)
```



官网文档： https://docs.mongodb.com/manual/reference/operator/query/elemMatch/index.html



> 作者：学习园
>
> 来自个人博客： https://xuexiyuan.cn/article/detail/227.html