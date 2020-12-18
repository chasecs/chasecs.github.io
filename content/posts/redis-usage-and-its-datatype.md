---
title: redis 的使用场景及其数据类型
date: 2019-08-09 16:02:43

---



redis 的使用场景总结有以下几种：

- 缓存/数据存储。涉及大部分 redis 的数据类型， string，Hash ，Zset ，set 等
- 队列。利用 list 类型的 push pop 操作。
- 事件通知。利用 publish/subscribe 消息系统

<!--more-->

## 1. 缓存/数据存储
<!--缓存主要应对大量读取请求、保存临时数据的场景，把可以复用的数据存到 redis 上，可以提高响应速度，减少服务器压力。

一个临时保存数据的场景就是保存用户的session，把通过验证的用户数据存到 redis，设置好时效，然后返回 ssesionId 给前端。用户再次访问时，可以通过 sessionId 鉴权并获取访问用户数据。
-->
- 主要应对大量读取请求、保存临时数据的等场景 
	- 保存临时数据场景：保存用户的session，把通过验证的用户数据存到 redis，设置好时效，然后返回 ssesionId 给前端。
	- 保存可复用的数据，某些请求频繁的数据，存到 redis 可提高响应速度，减少服务器压力。

- ### string 类型的应用
	- redis 最常见简单的操作就是以 string 类型保存数据。
	 
		```bash
		redis> SET userId:10001 "jack" #写入
		"OK"
		redis> GET  userId:10001 #读取
		"jack"
		redis> EXPIRE  userId:10001 100  # 100秒后过期
		```

	- redis 的 key不建议太短，词和词自己通常用 : - . 来分割，例如：`comment:1234:reply-to`.

- ### hash 类型

	- Hash 类型是可以以键值对（object，map）的格式保存和读取数据：
	
		```bash
		# command key field-1 value-1 field-2 value-2
		> hmset user:1000 username antirez birthyear 1977 verified 1
		OK
		```
- ### hash VS string

	- 键值对数据可以用 hash 保存；也可以转换成 json 格式，以 string 类型保存:
	 	
	 	```bash
	 	> SET user:1000 '{"username":"antirez","birthyear": 1977}'
	 	```
	- 如果数据的大部分操作是整存整取的话，以 string 保存 json 更好。
	- 如果大部分操作是零存零取，hash 会更好，因为 hash 可以指定读取某个 field 的值：

		```
		> hget user:1000 birthyear
		"1977"
		```

	- 大多数时候，尤其 field 不是非常多的情况，**这两种方法区别不大**，因为 json 的解码和编码过程很快。

	
- ### sets & sorted set

	- sets 类型是唯一、无排序的 string 的合集。
	- sorted sets 和 set 相似，唯一区别是里面有元素有个有一个数值，并按分数排序，示例：
		
		```bash
		# command key value-score value
		> zadd hackers 1940 "Alan Kay"
		(integer) 1
		> zadd hackers 1957 "Sophie Wilson"
		> (integer) 1
		> zadd hackers 1914 "Hedy Lamarr"
		(integer) 1
		```

		读取时，可以按顺序取出数据：
		
		```
		> zrange hackers 0 1
		1) "Alan Turing"
		2) "Hedy Lamarr"
		```

	- ZSET 适合需要对数据进行**频繁排序、筛选的场景** 。


## 2. 队列 与 List 类型

- list 类型以 Linked List 实现。优点：从头部或尾部插入数据事件复杂度是 O(N); 缺点是；从中间插入数据不快。 如果需要大量插入数据到中间，应该使用 SET 类型

- 一个典型的应用场景，网站首页展示用户们最新发布的 10 条推特，
	- 每当有用户发推时，`LPUSH` 消息 id 到 list
	- 有人访问网站首页时， `LRANGE 0 9` 获取最新 10条更新


## 3. 事件通知，publish/subscribe 消息系统

- “[publish/subscribe](https://en.wikipedia.org/wiki/Publish%E2%80%93subscribe_pattern
)”消息模式
	- 发布者不直接把消息发给接收方，也不需要知道接收方都是谁，它只要把消息发送到一个类（class，channel）即完成任务。同样的，而相信接收方即订阅者，只需要关心它订阅了哪些 channel，并从这些类获取消息，不需要知道消除是谁发送的。


- redis 的 publish/subscribe 消息模式

	- 在客户端 client_1 订阅 channel ：
	
		```bash
		# client_1
		# 订阅 channel 名为 example
		> SUBSCRIBE example
		Reading messages... (press Ctrl-C to quit)
		1) "subscribe"
		2) "example"
		3) (integer) 1
		```
	- 然后该客户端将一直监听新的通知。此时如果 client_2 发布消息：
	
		```bash
		# client_2
		PUBLISH example 2
		(integer) 1
		```
	- client_1 将弹出新消息：
	
		```bash
		# client_1
		#...
		1) "message"
		2) "example"
		3) "2"
		```

## 4. HyperLogLog 类型，统计合集总数

- HyperLogLog 数据类型用于统计集合总数的数据类型。相比传统的计数方式，它使用独特的算法去计数，只需占用很小的内存。
- 缺点：hyperloglog的数目是估算出来的，而且有 1% 情况会报错。
- 适用于统计某些大量的数据。例如，你建了一个爬虫，需要记录某天爬取的多少网站，可以这样操作:

	
```bash
	# 记录
	> PFADD crawled:20171124 "http://www.google.com/"
	(integer) 1
	> PFADD crawled:20171124 "http://www.redislabs.com/"
	(integer) 1
	> PFADD crawled:20171124 "http://www.redis.io/"
	(integer) 1
	> PFADD crawled:20171125 "http://www.redisearch.io/"
	(integer) 1
	....
	
```
	
```
	# 统计总数
	> PFCOUNT crawled:20171124
	(integer) 3
```

## 5. 待补充：
- [Introduction to Redis Streams](https://redis.io/topics/streams-intro)
- [redis as message borker VS rabbitmq](https://stackoverflow.com/questions/43777807/publish-subscribe-reliable-messaging-redis-vs-rabbitmq)



### 主要参考：

- [An introduction to Redis data types and abstractions](https://redis.io/topics/data-types-intro)
- [HyperLogLog](https://redislabs.com/redis-best-practices/counting/hyperloglog)
- [HyperLogLog in redis](https://thoughtbot.com/blog/hyperloglogs-in-redis)
- [stackoverflow： hash vs JSON string](https://stackoverflow.com/a/18991212/5561328)
- [quora: usage-of-Redis-pub-sub](https://www.quora.com/What-is-the-usage-of-Redis-pub-sub)
