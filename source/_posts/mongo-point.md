---
title: 使用MongoDB的位置索引实现“附近的人”的计算
date: 2017-04-28 07:57:41
tags:
    - MongoDB
    - LBS
categories:
    - MongoDB
---


### 使用MongoDB的位置索引实现“附近的人”的计算

最近一直在忙工作和毕设，没什么时间写博客，真的非常不好意思，在写毕设的时候，有一个需求，就是找到地图上附近的点，我使用的是MongoDB存储我的信息，正好Mongo也有位置索引，也可以进行位置搜索，而且使用起来非常方便，这边记录一下。

- 语言：Java8
- 框架：Spring MVC、Spring、Mybatis
- 数据库：MongoDB 3.4.2

#### 数据保存

你需要将坐标数据以特定的格式保存到MongoDB，我的数据源如下所示：
```json
{
    "restaurantId" : "fd433baf3a7e4722bdd2c7a9c1a0f7c0",
    "restaurantName" : "XX餐厅",
    "belong" : 2,
    "createTime" : "2017-04-26T09:24:43.266Z"
}
```

想要给他加上一个位置信息、[官方文档](https://docs.mongodb.com/manual/core/2d/)上是这样写的：

```json
loc : [ <longitude> , <latitude> ]
和
loc : { lng : <longitude> , lat : <latitude> }
```
用这样两种方式都是可以的，记住，一定是lng在前，lat在后，之前写的时候就是因为顺序写反了，导致我一直算不出附近的点，非常气。
于是我的数据源就变成了如下

```json
{
    "restaurantId" : "fd433baf3a7e4722bdd2c7a9c1a0f7c0",
    "restaurantName" : "XX餐厅",
    "position" : [
        120.576861,
        30.000107
    ],
    "belong" : 2,
    "createTime" : "2017-04-26T09:24:43.266Z"
}
```

在MongoDB的客户端选择上，Spring Data的MongoTemplate非常方便，可以使用它对Mongo做一系列操作，
将数据源插入MongoDB以及对位置信息做索引的代码如下：

```java
//插入数据，我使用了Restaurant这个类作为Entity，其实什么数据都可以，关键就是要有正确格式的坐标数据
mongoTemplate.insert(restaurant,collectionName);
//定义某个key为位置索引，索引格式为GEO_2D
GeospatialIndex position = new GeospatialIndex("position");
position.typed(GeoSpatialIndexType.GEO_2D);
//索引操作，给position这个key添加位置索引
mongoTemplate.indexOps(collectionName).ensureIndex(position);
```

这样您保存的数据就已经做好了索引，我们可以多添加一些点。

#### 数据查询

假设你有一个APP，你获取到了你当前的坐标，想要获取左边周围固定范围的点的集合，这时候可以使用MongoDB来做，因为在上面我们已经对position做了索引，所以可以使用mongoTemplate的NearQuery类进行附近的点的查询，查询的代码如下：

```java
//得到当前用户的坐标
double lng = 120.576861;
double lat = 30.000107;
//定义范围，在这个范围内的点都会被返回
double length = 1000.0;

//NearQuery专门为位置查询而生，Metrics类储存了单位的数据，让您可以设置不同的单位，这里我们设置为KILOMETERS(千米)
NearQuery nearQuery = NearQuery.near(lng, lat, Metrics.KILOMETERS).maxDistance(length);

//查询成功后则返回GeoResults<T>，该类专门存储位置信息和Entity信息，包括距离等有用的信息。
GeoResults<Restaurant> geoResults = mongoTemplate.geoNear(nearQuery, Restaurant.class, collectionName);
```

让我们来看一下GeoResults内部的格式是怎么样的：

```json
{
    "averageDistance": {
        "metric": "KILOMETERS",
        "normalizedValue": 0.003964060840443525,
        "unit": "km",
        "value": 25.28332311668394
    },
    "content": [
        {
            "content": {
                "belong": 2,
                "createTime": 1493198674010,
                "position": [
                    120.152171,
                    30.266635
                ],
                "restaurantId": "eea5e08be3dc4341b66ec0fc27b9a085",
                "restaurantName": "A"
            },
            "distance": {
                "metric": "KILOMETERS",
                "normalizedValue": 0.000014026592741484346,
                "unit": "km",
                "value": 0.08946353014839274
            }
        },
        {
            "content": {
                "belong": 2,
                "createTime": 1493198683266,
                "position": [
                    120.576861,
                    30.000107
                ],
                "restaurantId": "fd433baf3a7e4722bdd2c7a9c1a0f7c0",
                "restaurantName": "B"
            },
            "distance": {
                "metric": "KILOMETERS",
                "normalizedValue": 0.007914095088145565,
                "unit": "km",
                "value": 50.47718270321948
            }
        }
    ]
}
```

我们发现MongoTemplate为我们封装了很多有用的信息，包括averageDistance平均距离都有。这样我们就可以把附近的点的信息找到啦，是不是非常方便？


### 反馈与建议

- 微博：[@lzx2005](http://weibo.com/u/2557929062)
- 邮箱：<crow2005@vip.qq.com>
  ​                      
