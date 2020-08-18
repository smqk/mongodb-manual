# save 命令

```javascript
db.collection.save(
   <document>,
   {
     writeConcern: <document>
   }
)
```

如果 document 中不包含 _id 字段时，save 方法相当于是调用 insert 方法，运行时系统自动生成一个 ObjectId 值赋给 _id字段。如果包含 _id 字段，save 方法相当于是调用 upsert 为 true 的update 方法。

```javascript
//_id系统会生成
db.students.save({name:"张三",grade:90});

db.students.save({_id:40,name:"李四",grade:95}); 
//等价于如下的update
db.students.update({_id:40},{name:"李四",grade:95},true,false); 
```



