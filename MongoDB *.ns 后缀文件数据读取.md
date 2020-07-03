# MongoDB *.ns 后缀文件数据读取.md



.ns 是mmapv1引擎的数据库文件, mongo3.2 之后默认用了wiredtiger引擎 ，就是那些wt后缀的文件。知道了这些信息其实加载读取　.ns　文件数据思路就是使用　mongod　读取数据文件：

```bash
# ns的数据库数据使用 mongod　服务加载读取，然后用过　mongo　即可看查看到数据
mongod  --dbpath=你的db文件路径  --storageEngine=mmapv1
```



需要注意　**Ｍongod 的版本需要和数据文件 (*.ns) 所使用的数据库版本保持一致**，尝试使用　v4.0.17　版本的　mongod　读取　ｖ3.2.22　版本的数据库文件结果读取不到数据库数据。



MongoDB使用 mmapv1 引擎启动数据库服务时会产生以下两类数据库文件：　

一、{dbname}.ns   

二、{dbname}.{序号}



