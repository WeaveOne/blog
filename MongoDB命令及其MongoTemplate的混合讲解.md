# MongoDB命令及其MongoTemplate的混合讲解

## 前言

前面讲解了如何在springboot中集成mongodb，本文将讲解mongodb命令操作及其MongoTemplate的使用。穿插的目的在于不用先去寻找mongodb的命令又去寻找在java中的应用。本人就是从中过来的，所以本文旨在减少刚入门的同学少走一点弯路。

mongoDB所存储的数据以键值对的方式存储

## 注意事项

**在创建数据库和创建集合没有讲解MongoTemplate的使用是因为：在创建数据库时只需要在配置文件中配置database路径即可。而在创建集合是只需要进行插入或者更新操作即可自动创建集合**

注入mongotemplate

```java
@Autowired 
MongoTemplate mongoTemplate;
```



## 1. 创建数据库

```shell
use mydatabase
#插入数据
db.mydatabase({"name","我的数据库"})
show dbs
```

## 2. 创建集合

```shell
use mydatabase
db.createCollection("mycollection")
# 显示所有集合
show collections
```

## 3. 插入文档

命令方式：

```shell
db.mycollection.insert({
	"key":"data",
	"value":"数据"，
	"num": 1
})
```

java代码：

```java
// 自定义对象
Data data = new Data("data","数据",1);
mongoTemplate.insert(data, "mycollection");
```

## 3. 更新文档

命令方式：

```shell
# 更新单行
db.mycollection.update({"key":"data"},{"$set":{"value":"更新数据"}})
# 更新多行
db.mycollection.update({"key":"data"},{"$set":{"value":"更新数据"}},{multi:true})

#如果数据结构是 
{
	"key":"key"
    "data":{
        "value":"数据"
    }
}
db.mycollection.update({"key":"data"},{"$set":{"data.value":"更新数据"}})
```

java代码：

```java
 Update update = new Update();
 update.set("value":"更新数据")；
 update.set("data.value":"更新数据")；
 // num自增加 前提 num 对应的必须为整形或者浮点型
 update.inc("num", 1);
 Query query = new Query(
      Criteria.where("key").is("data")
 );
 mongoTemplate.updateFirst(query, update, "mycollecition");
```

## 4. 查询文档

命令方式

```shell
#普通查询
db.mycollection.find({"key":"data"})
#正则查询
db.mycollection.find({"key":{"$regex":"dat"}})
#大于等于 gte 小于等于 lte 大于 gt 小于 lt
db.mycollection.find({"num":{"$lt":1}})
```

