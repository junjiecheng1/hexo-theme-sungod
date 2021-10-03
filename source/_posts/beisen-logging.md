---
title: beisen logging
date: 2021-10-03 00:43:45
tags:
- beisen
---
# 项目简介

1.Beisen.Logging是北森的基础服务组件。以SDK的方式提供给业务线做日志打印和调用。

2.Beisen.Logging的日志打印集成了文件落地，分布式应用的日志集成，提供将日志存储至ES，并通过ET查看日志信息

# 日志存储的关键节点分析

##### 1.日志的存储

日志存储分为两部分。

###### 1.1本地文件落盘

```
Beisen.Logging的日志文件落盘存储，通过Log4Net进行日志的落地写入
```

[Log4Net配置文件内容](./logging,production.config)

[Apollo远程配置内容](./LoggingConfig.xml)

日志文件存储位置：

```c#
	C:\beisen.logfiles\$ApplicationName
```

###### 1.2分布式日志存储(ET)

```
日志输出通过调用 LogCollector.XXX方法，将日志异步输出到ET平台
```

```
ET日志级别包括：Fatal、Error、Warn、Info、Debug、ErrorTrace
```

###### 1.2.1日志信息包装

```
日志信息将message、线程id、sessionId、applicantionName、租户id、用户id、机器信息、时间信息、traceId（携带http trace则记录入口traceId）
```

```
http请求携带入参和基本请求头信息
```

###### 1.2.2将日志写入线程安全队列 `ConcurrentQueue<LoggingEntity>`

###### 1.2.3开启多个线程，消费队列内日志信息

```java
Enumerable.Range(0, handlers).ToList().ForEach(_ =>
            {
                var work = CreateThread(name + _.ToString(), () =>
                  {
                      while (true)
                      {
                          if (_queue.TryDequeue(out LoggingEntity loggingEntity))
                          {
                              Do(loggingEntity);
                          }
                          else
                          {
                              Thread.Sleep(1);
                          }
                      }
                  });
                work.Start();
            });
```

```
消费队列内日志信息时，将日志信息推送到kafka，不同类型的日志，发送不同Topic
```

###### 1.2.4Topic消费

```
不同Topic启动不同的windows服务，进行日志消息消费
```

```
每个windows service 中处理分为三部分
```

```
1、consumer消费kafka，并将对应日志加入BlockingCollection
```

```java
BlockingCollection<DebugPackageInfo>.AddToAny(ChannelCollections, package);
```

```
2、channel， 从BlockingCollection取出日志信息，添加到package消息包中，当消息达到一定数量，写入处理消息组
```

```java
 _collectTask = new Task(() =>
            {
                while (!cancellationToken.Token.IsCancellationRequested)
                {
                    var index = BlockingCollection<T>.TryTakeFromAny(_dataChannels, out T obj, 5);
                    if (obj != null)
                    {
                        package.Add(obj);
                        Interlocked.Increment(ref _count);
                        if (_count >= MaxPackageSize || _overtime)
                        {
                            handler(package);
                            package.Clear();
                            _count = 0;
                            _overtime = false;
                        }
                    }
                }
            }, cancellationToken.Token);
```

```
3、sinkAction，批量将日志内容写入到ES中
```

```
   var debugEntities = packages.Select(s => s.DebugEntity).ToList();
                    var lastDate = Convert.ToDateTime(debugEntities[0].LogDate);
                    debugEntities.ForEach(entity =>
                    {
                        entity.LogDate = Convert.ToDateTime(entity.LogDate).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ss.fff");
                        var logTag = LogTagFetcher.Fetch(entity);
                        entity.LogLevel = (int)LogLevelFetcher.Fetch(entity);
                        entity.LogLevelTag = (int)logTag.Category;
                    });
                    bulkAction.Bulk(indices.Indices(prefix, lastDate, lastDate)[0], _mapping, debugEntities, (e) =>
                    {
                        var tmp = e as DebugEntity;
                        return new ValueTuple<string, string, object, string, string, int, int>(tmp.LogGuid, tmp.ApplicationName, tmp, string.Empty, string.Empty, 0, 0);
                    });
```

# 设计优点

##### 2.1并发设计采用线程安全的队列

##### 2.2基础服务SDK减少依赖包引用

```
作为基础服务，北森全应用会引入此SDK，为了后续升级方便，且避免依赖包版本不兼容问题，减少依赖，能不依赖的就不依赖
```

```
kakfa的生产端和消费端自己内置了一套，避免依赖Kafka 的BeisenSDK
```

##### 2.3.链接池化，kafka的链接使用连接池，避免频繁创建socket链接

##### 2.4.日志服务写入存储对象时，避免频繁写入造成的IO调用的耗时，影响应用的吞吐量，采用异步队列的方式，进行日志的kafka消息发送

##### 2.5开关等变量信息，采用配置文件存储。针对Beisen的配置中心，采用Beisen.Configuration

# 拓展

#### 2.1 `ConcurrentQueue<T>队列详解`

[c#高效的线程安全队列](https://blog.csdn.net/liunianqingshi/article/details/79025818)

#### 2.2 `BlockingCollection<T>详解`

[MSDN BlockingCollection 概述](https://docs.microsoft.com/zh-cn/dotnet/standard/collections/thread-safe/blockingcollection-overview)

#### 2.3 ES的Bulk写入

[ES写入优化](https://jishuin.proginn.com/p/763bfbd54dfd)

[ES最佳性能优化](https://cloud.tencent.com/developer/article/1507035)
