# MongoDB update 更新命令汇总

## Update 更新命令

```javascript
//写法一
db.collection.update( <query>, <update>, upsert, multi );

//写法二
db.collection.update(
   <query>,
   <update>,
   {
     upsert: <boolean>,
     multi: <boolean>,
     writeConcern: <document>,
     collation: <document>,
     arrayFilters: [ <filterdocument1>, ... ]
   }
)
```

- query：update的查询条件，类似sql update查询内where后面的;

- update：更新的操作，也可以理解为sql update查询内set后面的;

- upsert：不存在是否新增，默认false，不存在update的记录不插入;

- multi：是否批量更新，默认false，不更批量更新只更新找到的第一条记录;
- collation：排序规则，详细参看[Collation Locales](https://docs.mongodb.com/manual/reference/collation-locales-defaults/#collation-languages-locales);
- writeConcern：写关注;
- arrayFilters：数组过滤条件，用于确定数组中那些元素需要更新。

> query、update 是必选条件，其他的都是选填。

```javascript
//只更新一条记录
db.students.update( { "grade" : { $gt : 100 } } , { $set : { "mark" : "优"} } ); 

//更新全部
db.students.update( { "grade" : { $gt : 80 } } , { $set : { "mark" : "良"} },false,true ); 
```



arrayFilters 实例，准备如下数据：

```javascript
db.students.insert([
   { "_id" : 1, "grades" : [ 95, 92, 90 ] },
   { "_id" : 2, "grades" : [ 98, 100, 102 ] },
   { "_id" : 3, "grades" : [ 95, 110, 100 ] }
])
```

更新操作

```javascript
//在<update>中，使用$[<identifier>]运算符来定义标识符，然后在数组过滤器中引用该标识符。 
db.students.update(
   { grades: { $gte: 100 } },
   { $set: { "grades.$[element]" : 100 } },
   {
     multi: true,
     arrayFilters: [ { "element": { $gte: 100 } } ]
   }
)
```

更新后结果

```javascript
{ "_id" : 1, "grades" : [ 95, 92, 90 ] }
{ "_id" : 2, "grades" : [ 98, 100, 100 ] }
{ "_id" : 3, "grades" : [ 95, 100, 100 ] }
```

参见：https://docs.mongodb.com/manual/reference/method/db.collection.update/#update-arrayfilters



## Update 字段更新操作

- **$set**

用法：{ $set : { field : value } }

就是相当于 sql 的set field = value，全部数据类型都支持$set，字段不存在则新增，字段存在则修改。

```javascript
> db.students.update( { "_id" : 15 } , { $set : { age : 18} } );
> db.students.update( { "_id" : 15 } , { $set : { hobby : ["basketball"], name: 'zs'} } );
```



- $unset

用法：{ $unset : { field : 1} }

删除字段。 字段不存在删除不会报错。

```javascript
> db.students.update( { "_id" : 15 } , { $unset : { test:1, hobby:1} } );
```



- $inc

用法：{ $inc : { field : value } }

对一个数字字段field增加value。

```javascript
> db.students.update( { "_id" : 15 } , { $inc : { "age" : 1 } } );
```



- $push

用法：{ $push : { field : value } } 把value追加到field里面去(值可以重复)，field 一定要是数组类型才行(对非数组类型 $push 会报错)，如果 field不存在，会新增一个数组类型加进去。

```javascript
db.students.update( { "_id" : 15 } , { $push : { "hobby": "football" } } );
```





## Update 数组字段更新操作

- $pushAll

用法：{ $pushAll : { field : value_array } }

同$push,只是一次可以追加多个值到一个数组字段内。

```javascript
> db.students.update( { "_id" : 15 } , { $pushAll : { "hobby": ["basketball","football"] } } );
```



- $addToSet

用法：{ $addToSet : { field : value } } 当值不在数组时增加值到数组。

```javascript
//这里的$each 相当于是 for，对$each 后的组件中的值做 $addToSet  操作
db.students.update( { "_id" : 15 } , { $addToSet : { "hobby": {$each : ["basketball","football"] } } } );

//如果没有 $each，则数组就相当于是一个 value 存在
```



- $pop 删除数组内的一个值

删除数组最后一个值：{ $pop : { field : 1 } }

删除数组第一个值：{ $pop : { field : -1  } }

注意，只能删除一个值，这里的1或-1代表的仅仅是顺序，小于 0 删除第一个，大于等于0删除最后一个。

```javascript
db.students.update( { "_id" : 15 } , { $pop : { "hobby": 1 } } );
```



- $pull用法：$pull : { field : value } } 从数组field内删除所有的value值。

```javascript
db.students.update( { "_id" : 15 } , { $pull : { "hobby": "basketball" } } );
```



- $pullAll用法：{ $pullAll : { field : value_array } }同$pull,可以一次删除数组内的多个值。

```javascript
db.students.update( { "_id" : 15 } , { $pullAll : { "hobby": [ "basketball" , "basketball" ] } } );
```



- $  操作符$是他自己的意思，代表按条件找出的数组里面某项他自己。

```javascript
> db.article.find({ "_id" : 1 })
{ "_id" : 1, "title" : "xuexiyuan.cn",  "comments" : [ { "by" : "joe", "votes" : 3 }, { "by" : "jane", "votes" : 7 } ] }

> db.article.update( {"_id" : 1, 'comments.by':'joe'}, {$inc:{'comments.$.votes':1}}, false, true )
> db.article.find({ "_id" : 1 })
{ "_id" : 1, "title" : "xuexiyuan.cn",  "comments" : [ { "by" : "joe", "votes" : 4 }, { "by" : "jane", "votes" : 7 } ] }
```

需要注意的是，**$只会应用找到的第一条数组项**，后面的就不管了。还是看例子：

```javascript
> db.t.insert({x: [1, 2, 3, 2]})
> db.t.update({x: 2}, {$inc: {"x.$": 1}}, false, true);
> db.t.find();
{"_id" : ObjectId("5bf67ef393df7c4446775ec1"),"x" : [1,3,3,2]}
```

还有注意$配合$unset 使用的时候，会留下一个null 的数组项，不过可以用{$pull:{x:null}}删除全部是null的数组项。例：

```javascript
> db.t.insert({x: [1,2,3,4,3,2,3,4]})
> db.t.find()
{ "_id" : ObjectId("5bf67fdb93df7c4446775ec2"), "x" : [ 1, 2, 3, 4, 3, 2, 3, 4 ] }

> db.t.update({x:3}, {$unset:{"x.$":1}})
> db.t.find()
{ "_id" : ObjectId("5bf67fdb93df7c4446775ec2"), "x" : [ 1, 2, null, 4, 3, 2, 3, 4 ] }
> db.t.update({},{$pull:{x:null}})
{ "_id" : ObjectId("5bf67fdb93df7c4446775ec2"), "x" : [ 1, 2, 4, 3, 2, 3, 4 ] }
```



> 官网参考： https://docs.mongodb.com/manual/tutorial/update-documents/
>

