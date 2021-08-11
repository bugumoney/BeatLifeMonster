# MongoDB日常操作

### 复制集操作

```javascript
//移除复制集节点，在主库操作
rs.remove("10.10.2.214:27017");

//加入复制集节点，在主库操作
rs.add({"host":"10.10.2.216:27017"});

//加入复制集隐藏节点和延迟节点，在主库操作
rs.add({"host":"10.10.2.214:27017","priority":0,"hidden":true});
rs.add({"host":"10.10.2.215:27017","priority":0,"hidden":true, slaveDelay: 1800});

//复制集配置
rs.conf();

//查看复制集信息
rs.printReplicationInfo();
```

### db操作 

```javascript
//当前数据库服务器状态
db.serverStatus();

//使用mongostat监控MongoDB全局情况
mongostat 10 --host localhost:27017 -u root -p xxxx --authenticationDatabase admin

//集合占用时间
mongotop 10 --host localhost:27017 -u root -p xxxx --authenticationDatabase admin

//慢查询性能监控，查看查询记录
//查看当前是否打开慢查询
db.getProfilingLevel({});

//查询时间超过2000ms则记录慢查询
db.setProfilingLevel(1, 2000);

//慢查询记录
db.getCollection('system.profile').find({});

//杀死mongodb连接，opid为慢查询的唯一标识
db.killOp("opid");

//查看客户端连接数，按客户端ip分组汇总
netstat -anp | grep 27017 | awk '{print $5}' |awk -F: '{print $1}' | sort | uniq -c | sort - rn | head -30
```

