---
title: Performance Tunning How To
date: 2016-05-27 14:11:14
tags:
  - MongoDB
  - Nginx
  - Cache
---

## 几个衡量指标

- TPS(Transaction Per second): 即每秒钟系统能够处理事务或交易的数量
- 响应时间： 一般取平均响应时间
- 并发数： 系统同时处理的事务数
- 资源使用率： CPU占用率， 内存使用率，磁盘I/O， 网络I/O

其中 TPS = 并发数 / 响应时间

## 使用工具

- [阿里云性能测试](https://www.aliyun.com/product/pts/?spm=5176.7960203.237031.117.A5pzqQ)
- [AB](https://httpd.apache.org/docs/2.4/programs/ab.html)
- [Jmeter](http://jmeter.apache.org/)
- ...

我本次测试使用的是阿里云提供的云性能测试，原因是使用云测试可以使我们无需安装软件即可轻松地模拟出分布式压测场景（支持几十万用户及百万级TPS性能压测），同时还可以自己编写测试脚本（python 语言）。对于测试结果还有比较全面的图表可用于分析性能瓶颈。
<!-- more -->

## 服务端监控

- 在测试过程中我使用的是[透视宝](https://www.toushibao.com/)的服务来监控请求状态
- 由于服务器 （Web, DB）都是使用阿里云的 ECS, 在构建测试任务时可以将这两台机器加入到 ECS 监控中

## 测试场景

- 现有一个餐桌预定系统，系统中存在 50 家餐厅，每家餐厅在某一天存在 1000 个订单，在此基础上
- 我要测试的是模拟 50 个用户持续向这 50 家餐厅发送创建某一天的预约订单请求，持续 10 分钟

## 测试结果分析

- TPS 及 响应时间

| 优化 | 平均 TPS(次/s) | 平均响应时间(ms) | 最大响应时间(ms) |
|------|--------------|----------------|-----------------|
| 无 | 19.14 | 2196.11 | 2729.30 |

- 资源利用率

![improve with nothing](/images/performance-tunning/perf_nothing.png)

其中黑色表示 MongoDB(本测试系统所使用的数据库) CPU 利用率，蓝色表示 Web Server 的 CPU 利用率(Web Server 与 MongoDB 部署在不同机器上)

可以看到 MongoDB 的 CPU 已经跑到 100%，可以推断出 **性能瓶颈出现在数据库这边**

内存使用率，磁盘I/O， 网络I/O 使用的都不是很高，所以这里我就不再贴图了

## 优化

### 优化数据库

#### 发现问题

- 使用 mongostat 实时监控 MongoDB
- 开启 MongoDB profilling

```
# 表示 query 时间大于 100ms 的请求都会被记录下来
db.setProfilingLevel(1, 100)
```

- 使用[Dex](https://github.com/mongolab/dex)工具来根据请求分析给出创建数据库索引的建议 （经过试验，Dex 只能分析出一些简单的查询语句，对于复杂的查询条件，例如包含 `$or`, `$and` 之类的是推荐不出来的）

通过 `db.system.profile.find({"ns": "perf.*","op": "query"}).pretty()` 查看 profiling 结果, 下面是 profiling 结果的部分截图

```
{
    "op" : "query",
    "ns" : "perf.order",
    "query" : {
        "$query" : {
            "_id" : {
                "$ne" : null
            },
            "restaurantId" : ObjectId("571d6caf987b41366b8b4a82"),
            "$or" : [
                {
                    "type" : NumberLong(1),
                    "status" : {
                        "$in" : [
                            NumberLong(1),
                            NumberLong(2),
                            NumberLong(6)
                        ]
                    },
                    "reserveAt" : {
                        "$gte" : ISODate("2016-05-27T21:00:00Z"),
                        "$lte" : ISODate("2016-05-28T07:30:00Z")
                    }
                },
                {
                    "type" : NumberLong(2),
                    "status" : {
                        "$in" : [
                            NumberLong(6)
                        ]
                    },
                    "seatedAt" : {
                        "$gte" : ISODate("2016-05-27T21:00:00Z"),
                        "$lte" : ISODate("2016-05-28T07:30:00Z")
                    }
                }
            ],
            "tableIds" : ObjectId("573eb8c2987b4136608b4583"),
            "isExpired" : false,
            "isDeleted" : {
                "$exists" : true,
                "$eq" : false
            }
        },
        "$orderby" : {
            "createdAt" : NumberLong(-1)
        }
    },
    "nscanned" : 52219,
    "millis" : 160,
    ...
}
```

- `nscanned`: 表示扫描的文档数目 52219 即数据库中的全部订单
- `millis`: 表示 query 所用的时间 160ms

由此发现在检测订单冲突，获取相关订单的这个查询与排序上耗费了大量时间和 CPU 计算

#### 解决问题

为 Order 表创建索引， 根据 profile 结果，在 order 表中创建了 keys 为 `restaurantId`, `isDeleted`, `isExpired` 的索引

```
db.order.createIndex({restaurantId: 1, isDeleted: 1, isExpired: 1}, {background: true});
```

使用 MongoDB 的 explain 工具对 query 请求进行分析

```
{
        "executionStats" : {
        "executionSuccess" : true,
        "nReturned" : 6,
        "executionTimeMillis" : 9,
        "totalKeysExamined" : 1045,
        "totalDocsExamined" : 1045,
        "executionStages" : {
            ...
        }
    }
}
```

- `totalDocsExamined` 表示的是此次查询所遍历的文档数目，1045 比之前的 52219 减少了很多
- `executionTimeMillis` 表示 query 所用的时间 9ms 相比之前的 160ms 也减少了很多

#### 遗留问题

现在订单以 **餐厅**、 **是否删除**、 **是否过期** 作为索引粒度还是很粗的，就按每个餐厅每天 1000 张订单算 1个月的订单量就是 30000，按照目前的查询条件查找订单，每次都要遍历 30000 条文档，而且这个是随时间增加的。所以这个解决方案并不有真正的解决问题。Take it easy, I will deal with it later.

重新执行性能测试任务，观察结果

- TPS 及 响应时间

| 优化 | 平均 TPS(次/s) | 平均响应时间(ms) | 最大响应时间(ms) |
|------|--------------|----------------|-----------------|
| 无 | 19.14 | 2196.11 | 2729.30 |
| 创建 Order 索引 | 32.79 | 1316.77 | 2163.03 |

通过对比可以看到 TPS 和 响应时间都得到了显著改善

- 资源利用率

![improve with mongo index](/images/performance-tunning/perf_mongo_index.png)

其中黑色表示 MongoDB CPU 利用率，蓝色表示 Web Server 的 CPU 利用率

可以看到 Web Server 的 CPU 已经跑到 100%，可以推断出 **性能瓶颈出现在 Web Server 这边**

### 优化 code

下面是使用透视宝查看慢请求的调用堆栈信息，由于篇幅原因，这里我只截取了部分信息展示

![Call Stack](/images/performance-tunning/perf_call_stack.png)

通过分析耗时最长的几个方法大概是下面几个：

- RestaurantOfflineValidator::validate() 验证餐厅是否下线
- LanguageDetector::detect() 检测客户端语言
- Authenticator::check() 授权
- Lock::accquireLock() 获取订单共享锁，用于在检测冲突时锁住订单不被其它请求更改
- Order::compareTime() 比较时间，计算订单可用翻台时间
- Order::updateInElastic() 将桌子信息更新到 Elasticsearch 上

发现问题：

- 其中 *验证餐厅是否下线* 、 *检测客户端语言* 、 *授权* 都调用了 `getToken` 方法去查询数据库获取 token
- Lock::accquireLock() 需要用到餐厅的 setting, 而 seting 是从 restaurant 中获取的，然后方法的参数接收的是 `restaurantId` 就需要查询一次数据库根据 `restaurantId` 获取 restaurant
- updateInElastic() 中发现有一句数据库调用代码只有在 if 语句中才用到的却写到了 if 语句外面

优化思路：

- 在一次请求中，对于 token, restaurant, setting, user 这些常用的数据使用 in-process cache 进行缓存从而减少一次请求中重复与数据库交互
- 在方法的参数中使用 restaurant 代替 restaurantId, 因为 restaurant 是已经经过缓存处理过的
- 明确代码调用顺序与变量作用域减少没必要的调用与引用
- 改进查询数据库条件从而解决上面提出的 '遗留问题'

通过 profiling 结果，可以看到我们的查询条件是这样的

```
{
    "_id" : {
        "$ne" : null
    },
    "restaurantId" : ObjectId("571d6caf987b41366b8b4a82"),
    "$or" : [
        {
            "type" : NumberLong(1),
            "status" : {
                "$in" : [
                    NumberLong(1),
                    NumberLong(2),
                    NumberLong(6)
                ]
            },
            "reserveAt" : {
                "$gte" : ISODate("2016-05-27T21:00:00Z"),
                "$lte" : ISODate("2016-05-28T07:30:00Z")
            }
        },
        {
            "type" : NumberLong(2),
            "status" : {
                "$in" : [
                    NumberLong(6)
                ]
            },
            "seatedAt" : {
                "$gte" : ISODate("2016-05-27T21:00:00Z"),
                "$lte" : ISODate("2016-05-28T07:30:00Z")
            }
        }
    ],
    "tableIds" : ObjectId("573eb8c2987b4136608b4583"),
    "isExpired" : false,
    "isDeleted" : {
        "$exists" : true,
        "$eq" : false
    }
}
```

使用目前的索引 `{restaurantId: 1, isDeleted: 1, isExpired: 1}` 就会导致随着时间的推移，查询订单时要遍历的文档数目变得越来越多， 有没有一种方法可以把订单按天索引，这样每次查询时数据库要遍历的文档数量都稳定在一家餐厅一天的订单数量？
按照这个想法，在订单数据结构中引入一个额外的字段 `dineAt` 用于表示订单的就餐时间(如果是预约类型的订单就等于 `reserveAt` 的时间，如果是散客(直接到店的)订单就等于 `seatedAt` 的时间)，这样一来，查询条件就变成了下面这样：

```
{
    "_id" : {
        "$ne" : null
    },
    "restaurantId" : ObjectId("571d6caf987b41366b8b4a82"),
    "dineAt" : {
        "$gte" : ISODate("2016-05-27T21:00:00Z"),
        "$lte" : ISODate("2016-05-28T07:30:00Z")
    },
    "$or" : [
        {
            "type" : NumberLong(1),
            "status" : {
                "$in" : [
                    NumberLong(1),
                    NumberLong(2),
                    NumberLong(6)
                ]
            }
        },
        {
            "type" : NumberLong(2),
            "status" : {
                "$in" : [
                    NumberLong(6)
                ]
            }
        }
    ],
    "tableIds" : ObjectId("573eb8c2987b4136608b4583"),
    "isExpired" : false,
    "isDeleted" : {
        "$exists" : true,
        "$eq" : false
    }
}
```

更新之前的索引为 `{restaurantId: 1, dineAt: 1, isDeleted: 1, isExpired: 1}`，现在再查询订单时要遍历的文档数目就固定在了一天一家餐厅的量上了 ：）

当然这样的改动也不是没有代价的，比如：

- 数据库中多了一个字段，每次创建订单和更新订单时都需要维护这个字段
- 需要占用更多的数据库服务器资源去维护这个索引

相比它所带来的好处，我想这些是可以忍受的

### 优化 Web 服务器 (Nginx)

#### TCP 优化

```
http {
    sendfile           on;
    tcp_nopush         on;
    tcp_nodelay        on;
}

```

关于这部分内容的更多介绍可以看这篇文章：[NGINX OPTIMIZATION: UNDERSTANDING SENDFILE, TCP_NODELAY AND TCP_NOPUSH](https://t37.net/nginx-optimization-understanding-sendfile-tcp_nodelay-and-tcp_nopush.html)

#### 开启 gzip

在服务端发送响应之前进行 GZip 压缩很重要，通常压缩后的文本大小会减小到原来的 1/4 - 1/3

```
http {
    gzip               on;
    gzip_vary          on;

    gzip_comp_level    6;
    gzip_buffers       16 8k;

    gzip_min_length    1000;
    gzip_proxied       any;
    gzip_disable       "msie6";

    gzip_http_version  1.0;

    gzip_types         text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript application/javascript;
    ... ...
}
```

#### TCP Fast Open（TFO）

```
server {
    listen             443 ssl spdy fastopen=3 reuseport;
    spdy_headers_comp  6;
    ... ...
}
```

- `fastopen=3` 用来开启 TCP Fast Open 功能。3 代表最多只能有 3 个未经三次握手的 TCP 链接在排队。超过这个限制，服务端会退化到采用普通的 TCP 握手流程。这是为了减少资源耗尽攻击：TFO 可以在第一次 SYN 的时候发送 HTTP 请求，而服务端会校验 Fast Open Cookie（FOC），如果通过就开始处理请求。如果不加限制，恶意客户端可以利用合法的 FOC 发送大量请求耗光服务端资源。
- `reuseport` 就是用来启用 TCP SO_REUSEPORT 选项的配置

TFO 的作用是用来优化 TCP 握手过程。客户端第一次建立连接还是要走三次握手，所不同的是客户端在第一个 SYN 会设置一个 Fast Open 标识，服务端会生成 Fast Open Cookie 并放在 SYN-ACK 里，然后客户端就可以把这个 Cookie 存起来供之后的 SYN 用

关于 TCP Fast Open 的更多信息，可以查看这篇文章：[Shaving your RTT with TCP Fast Open](https://bradleyf.id.au/nix/shaving-your-rtt-wth-tfo/)

#### 开启缓存 （并不适合我们本次测试的场景）

- 服务端
使用 Nginx Microcaching 即使是动态资源也可以缓存。从而提高系统的 TPS. 详细内容请查看 [Nginx microcaching](https://www.nginx.com/blog/benefits-of-microcaching-nginx/)
- 客户端 （HTTP/1.1 的缓存机制）
    - Server 返回 `Last-Modified`（最后修改时间），Client 下一次请求带上 `If-Modified-Since: 上次 Last-Modified 的内容`
    - Server 返回 `ETag`（内容特征），Client 下一次请求带上 `If-None-Match: 上次 ETag 的内容`

更多关于 HTTP 缓存优化可以参考[这里](https://merrychris.com/2017/03/18/http-cache/)

#### HTTPS 优化

由于我们的服务是部署在 https 的，所以对 https 的优化还是十分有必要的，主要包括以下几点：

- Session resumption
- OCSP stapling
- TLS false start

有关这三个配置的优化细节可能参考这篇文章：[TLS 握手优化详解](https://imququ.com/post/optimize-tls-handshake.html)

再次执行性能测试任务，观察结果

- TPS 及 响应时间

| 优化 | 平均 TPS(次/s) | 平均响应时间(ms) | 最大响应时间(ms) |
|------|--------------|----------------|-----------------|
| 无 | 19.14 | 2196.11 | 2729.30 |
| 创建 Order 索引 | 32.79 | 1316.77 | 2163.03 |
| 创建 Order 索引, 优化 code, nginx 服务器 | 42.79 | 1171.58 | 1966.48 |

- 通过对比可以看到 TPS 和 响应时间相比之前 2 次又一次得到了显著改善

使用 vmstat 查看 MongoDB 与 Web Server 资源利用率情况

- Web Server

![vmstat_web_server](/images/performance-tunning/perf_web_server.png)

- MongoDB

![vmstat_mongo](/images/performance-tunning/perf_mongo.png)

通过监控结果看到，Web Server CPU 持续跑满

### 增加 Web Server 数量 (水平扩展)

现在系统的架构如下图所示

![perf_architecture](/images/performance-tunning/perf_architecture.png)

再次进行测试

- TPS 及 响应时间

| 优化 | 平均 TPS(次/s) | 平均响应时间(ms) | 最大响应时间(ms) |
|------|--------------|----------------|-----------------|
| 无 | 19.14 | 2196.11 | 2729.30 |
| 创建 Order 索引 | 32.79 | 1316.77 | 2163.03 |
| 创建 Order 索引, 优化 code, nginx 服务器 | 42.79 | 1171.58 | 1966.48 |
| 创建 Order 索引, 优化 code, nginx 服务器， 增加一台 Web Server | 85.47 | 891.83 | 1511.35 |

TPS 相比一台 Web Server 的情况增长了一倍，响应时间也得到了改善

- 通过阿里云的 ECS 监控可以看到两台 Web Server CPU 已经跑满， MongoDB CPU 跑了 20% 左右。

## 结语

到此我的性能测试也暂时告一段落了，还有很多地方需要学习，有什么问题或写的不对的地方欢迎大家留言。
