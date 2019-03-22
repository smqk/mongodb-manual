# NoSQLBooster for MongoDB 中跨库关联查询



使用 MongoDB 是我们常常会遇到一些特殊的需求需要跨库关联查询，比如订单明细缺商品重量需要补商品重量，而商品重量数据又在商品库中，这事就需要跨库关联操作，示例代码如下：

```javascript
// 使用 order 库，注意语句后面不要加分号
use order

var count = 0;
db.order_detail.find({"store_code":"110"}).forEach(function(_order){
    var item = db.getSiblingDB("goods").item.findOne({"barcode":_order.barcode});
    if(item){
        db.order_detail.update({_id:_order._id},{$set:{"weight":item.weight}},false,true);
        count++;
    }else{
        print("商品不存在， 条码：" + _order.barcode);
    }
})；
print("更新条数：" + count);
```

> 注意：跨库查询时必须使用 admin 库来授权连接。

上面示例的代码，数量不多时还能勉强凑合着使用。当数据量达到上万条数据时就显示非常非常慢。因为更新一条数据需要单条 findOne 在单条 update。因此得优化，将单条查询改批量查询（缓存查询结果），示例代码如下：

```javascript
use order

var count = 0;
var items = {};
db.getSiblingDB("item").goods.find({"store_code":"110"}).forEach(function(_item){
    items[_item.barcode] = _item;
});
db.order_detail.find({"store_code":"110"}).forEach(function(_order){
    var item = items[_order.barcode];
    if(item){
        db.order_detail.update({_id:_order._id},{$set:{"weight":item.weight}},false,true);
        count++;
    }else{
        print("商品不存在， 条码：" + _order.barcode);
    }
})；
print("更新条数：" + count);
```

 

进过将单条查询改成批量查询后执行效率确实提升不少，但是还是觉得慢，还得继续优化，将单条更新改成批量更新，示例代码如下：

```javascript
use order

var count = 0;
var items = {};
db.getSiblingDB("item").goods({"store_code":"110"}).forEach(function(_item){
    items[_item.barcode] = _item;
});
var ops = [];
db.order_detail.find({"store_code":"110"}).forEach(function(_order){
    var item = items[_order.barcode];
    if(item){
        var f = {_id:_order._id};
        var upd = {$set:{"weight":item.weight}};
        ops.push({"updateMany":{"filter":f, "update":upd, "upsert":false}});
        count++;
    }else{
        print("商品不存在， 条码：" + _order.barcode);
    }
    
    if(count > 0 && count % 1000 == 0){
        // 批量更新， ordered：false 无序操作
        db.order_detail.bulkWrite(ops, {ordered:false});
        ops = [];
        print("更新条数：" + count);
    } 
})；

if(ops.length > 0){
    db.order_detail.bulkWrite(ops, {ordered:false});
} 
print("更新完成，更新总条数：" + count);
```



批量更新参见：  http://www.xuexiyuan.cn/article/detail/173.html



> 作者：学习园
> 来自个人博客： https://xuexiyuan.cn/article/detail/204.html