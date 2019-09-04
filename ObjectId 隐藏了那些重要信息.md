# ObjectId 隐藏了那些重要信息

## 描述

ObjectId(<hexadecimal>)

- hexadecimal 参数选填，16进制的数字。

返回一个新的 ObjectId 值，其中12字节ObjectId 由以下部分组成：

- 4个字节时间戳。
- 5个字节的随机数
- 3个字节的基于随机数的计数器



## 方法及属性

| 属性/方法               | 描述                                                    |
| ----------------------- | ------------------------------------------------------- |
| ObjectId.getTimestamp() | 获取当前对象时间戳信息。                                |
| ObjectId.toString()     | 返回当前对象以字符串 ObjectId(...) 形式的值。           |
| ObjectId.valueOf()      | 返回当前对象16进制对应的值。返回的字符串就是 str 属性。 |





## 实例

```javascript
// 无参形式生成一个 ObjectId
var x = ObjectId()

// 有参形式生成一个 ObjectId
var y = ObjectId("507f191e810c19729de860ea")

// 获取对象的 str 属性
print("x.getTimestamp(): " + x.getTimestamp());
print("y.getTimestamp():" + y.getTimestamp());

// 调用对象的 toString 方法
print("x.toString(): " + x.toString());
print("y.toString(): " + y.toString());

// 调用对象的 valueOf 方法
print("x.valueOf(): " + x.valueOf());
print("y.valueOf(): " + y.valueOf());

// ====== 运行结果 =======
x.getTimestamp(): Tue Jul 23 2019 16:57:34 GMT+0800 (CST)
y.getTimestamp():Thu Oct 18 2012 04:46:22 GMT+0800 (CST)
x.toString(): ObjectId("5d36cbfeb1af39357dc4b867")
y.toString(): ObjectId("507f191e810c19729de860ea")
x.valueOf(): 5d36cbfeb1af39357dc4b867
y.valueOf(): 507f191e810c19729de860ea
```





```json
var id = new ObjectId("507f191e810c19729de860ea");
print(id.getTimestamp()); //1350506782000（十进制）
// 507f191e（十六进制）  -->  1350506782
//总结 objectId 前8个字符就是时间戳

db.test.insert({_id: id, date: id.getTimestamp()})

var id = new ObjectId();
db.test.insert({_id: id, date: id.getTimestamp()})


db.test.find()
db.test.find({_id:{$gt:ObjectId("5d68a10f0000000000000000")}})  // 等价于  2012-10-18T04:46:22.000db.test.find({_id:{$gt:ObjectId("5d68a10fc03cf22aaa8ab761")}})
db.test.find({_id:{$lt:ObjectId("5d68a10fc03cf22aaa8ab761")}})
```





https://blog.csdn.net/qq_20545159/article/details/48657479

