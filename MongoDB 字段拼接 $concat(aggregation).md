# MongoDB 字段拼接 $concat(aggregation)

## $concat

拼接字符串操作，返回拼接后的字符串。语法格式如下：

```java
{ $concat: [ <expression1>, <expression2>, ... ] }
```

> 参数可以是任何有效的表达式，只要它们解析为字符串即可。 有关表达式的更多信息，请参阅[表达式](https://docs.mongodb.com/manual/meta/aggregation-quick-reference/#string-expression-operators)。





## 示例

准备以下测试数据：

```javascript
db.inventory.drop();
var rows = 
[
	{ "_id" : 1, "item" : "ABC1", quarter: "13Q1", "description" : "product 1" },
	{ "_id" : 2, "item" : "ABC2", quarter: "13Q4", "description" : "product 2" },
	{ "_id" : 3, "item" : "XYZ1", quarter: "14Q2", "description" : null },
	{ "_id" : 4, "item" : "CCCC", quarter: "4000"}
];
db.inventory.insert(rows);
```

使用 $conct 连接 item 和 description字段，字段之间以 “-” 分割：

```javascript
db.inventory.aggregate(
   [
      { $project: { itemDescription: { $concat: [ "$item", " - ", "$description" ] } } }
   ]
)
```

运行结果如下：

```javascript
{ "_id" : 1, "itemDescription" : "ABC1 - product 1"},
{ "_id" : 2, "itemDescription" : "ABC2 - product 2"},
{ "_id" : 3, "itemDescription" : null},
{ "_id" : 4, "itemDescription" : null}
```

如果数据库中有不能解析成字符串的异常数据，例如如下数据：

```javascript
{ "_id" : 5, "item" : "CCCC", quarter: "4000", "description" : 4}
```

则查询时会抛错误，错误信息如下：

```javascript
{
	"message" : "$concat only supports strings, not double",
	"ok" : 0,
	"code" : 16702,
	"codeName" : "Location16702",
	"name" : "MongoError"
}
```



## 总结

在使用 $concat  是做字符串拼接操作时，如果参数解析为null值或引用缺少的字段，则 $concat 返回null。



