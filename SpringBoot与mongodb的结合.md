title: "SpringBoot与mongodb的结合"
date: 2019-07-28 10:48:16
categories: java
tags: [spring boot, mongodb]

----



本文系列文章：

​	使用Shell 操作 MongoDB的技巧

​	MongoTemplate的使用技巧及其注意事项

敬请期待。

## 前言

最近公司想要做一个用户行为数据的收集，最开始想用mysql来存储后来发现这种方式对于不固定数据格式的保存存在局限性，也不利于查询统计操作。所以衍生了使用mongodb这种非结构化的数据库来保存。

## mongoDB简介

MongoDB（来自于英文单词“Humongous”，中文含义为“庞大”）是可以应用于各种规模的企业、各个行业以及各类应用程序的开源数据库。作为一个适用于敏捷开发的数据库，MongoDB的数据模式可以随着应用程序的发展而灵活地更新。与此同时，它也为开发人员 提供了传统数据库的功能：二级索引，完整的查询系统以及严格一致性等等。 MongoDB能够使企业更加具有敏捷性和可扩展性，各种规模的企业都可以通过使用MongoDB来创建新的应用，提高与客户之间的工作效率，加快产品上市时间，以及降低企业成本。

MongoDB是专为可扩展性，高性能和高可用性而设计的数据库。它可以从单服务器部署扩展到大型、复杂的多数据中心架构。利用内存计算的优势，MongoDB能够提供高性能的数据读写操作。 MongoDB的本地复制和自动故障转移功能使您的应用程序具有企业级的可靠性和操作灵活性。

<!-- more -->

## 1. mongoDB 安装

本文采用docker才安装mongoDB

访问：<https://hub.docker.com/r/bitnami/mongodb/>  选取需要的版本号,具体操作也可查看

```shell
docker pull bitnami/mongodb
docker run --name mongodb -d -p 27017:27017 bitnami/mongodb
```

## 2. Springboot 添加mongoDB依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

## 3. 修改配置文件

```properties
#mongoDb
spring.data.mongodb.host=127.0.0.1
spring.data.mongodb.port=27017
#数据库名称
spring.data.mongodb.database=behavior 
```

## 4.简单使用

```java
    @Autowired 
    MongoTemplate mongoTemplate;
    @Test
    public void testMongo(){
        Query query = Query.query(Criteria.where("key").is("mongo"));
        // "mongo" 为容器名 mongodb的具体知识请访问mongodb官方
        mongoTemplate.findOne(query,Map.class,"mongo");
    }
```

