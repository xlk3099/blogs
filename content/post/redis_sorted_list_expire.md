---
title: "Reid 给list添加超时属性"
date: 2018-10-25T17:28:39+08:00
draft: false
---

Redis里简单的K/V Pair默认是有expire属性的，当设定超时时间，一旦超时，设置的k/v对会自动被清除，但对于在list的item而言，并没有这个超时设置选项。

最近业务里有一个小需求，Redis需要缓存一个未确认交易list：

1. 对于list里的任意一个item，需要支持超时功能，自动删除12小时以前的记录。
2. 能通过key（很长的hash）快速删除列表里指定item。
3. 能通过key快速返回列表里item的内容。
4. list能按时间排序返回列表里指定数量的交易（offset,limit) 

单纯使用K/V pair的话已经能满足1，2，3， 主要是需要按时间倒序返回交易列。
我一开始想到的是通过使用一个辅助list，给K/V Pairs增加按时排序功能。具体实现，是将K/V pair set的所有keys按时间排序插入到list里。

**结构**：
* list (ptxList)： 记录每个交易的key，按时间插入（LPUSH）
* K/V pairs： 每笔交易的key跟其对应的详细交易内容。(共享前缀ptx）

**需求实现**：
* 超时设置: 每个K/V pair都自带expire属性，设置12小时即可
* 获取指定未确认交易: GET ptx:[key]
* 删除未确认交易
    ```
    LREM mylist ptx:[key]
    DEL key
    ```
* 获取按时间排序的未确认交易列: 因为list里的items是按时间插入的可以通过 (注意这里我们一定要sort by`not-exist-key`,因为ptxList本身就是按时间插入排序好，这里只是起辅助作用）
     ``` 
     SORT ptxList by not-exist-key GET ptx*
     ```
咋看可行，跑了一段时间貌似也行，但很快就发现了一个问题，mylist 的keys跟k/v pairs里的keys不匹配。

> 原因： 对应K/V pairs，当某个key超时时，该key会被删除，mylist里面keys却不会。

确实，每笔交易确认之后，keys都有从k/v pair组跟list删除。也就是说这个方法只能保证前12小时的数据准确性。之后，mylist的keys会比 k/v pair组多很多。
问题又回到了起点...

### 使用redis sorted list ###

> 万能的Google，每当你遇到一个坑跨不过的时候，总有前辈给你搭好了桥。

redis 里自带了sorted list。 往sorted list里添加item时，可以添加一个score attribute

1. ZADD SCORE MEMBER
2. ZREMRANGEBYSCORE

方法跟上面一致，只不过是将normal的list换成了sorted list，然后每次插入一个item的时候，添加一个score，score值位当前的timestamp（unix epoch格式)。 这样可以通过`ZREMRANGEBYSCORE ptxList min max`来删除过期的members.

例子:
```
建立一个sorted list
127.0.0.1:6379> zadd ptxlist 1540458292 key1
(integer) 1
127.0.0.1:6379> zadd ptxlist 1540458293 key2
(integer) 1
127.0.0.1:6379> zadd ptxlist 1540458294 key3
(integer) 1
127.0.0.1:6379> zadd ptxlist 1540458294 key5
(integer) 1
127.0.0.1:6379> zadd ptxlist 1540458293 key4
(integer) 1
```
这里要注意，虽然key4 比key5 晚插入，但是因为是sorted list， 由于key4的score比key3跟key5低，插入4之后，ptxlist的实际顺序是 `key1, key2, key4, key3, key5`

创建对应的k/v 键值对。
```
127.0.0.1:6379> set ptx:key1 data1
OK
127.0.0.1:6379> set ptx:key2 data2
OK
127.0.0.1:6379> set ptx:key3 data3
OK
127.0.0.1:6379> set ptx:key4 data4
OK
127.0.0.1:6379> set ptx:key5 data5
OK
```

取得最新的两个交易记录, 应该返回data5， data3.
```
127.0.0.1:6379> sort ptxlist limit 0 2 desc by not-exist-key get ptx:*
1) "data5"
2) "data3"
```

删除超时的key, 删除key1，key2，key4
```
127.0.0.1:6379> zremrangebyscore ptxlist 0 1540458293
(integer) 3
127.0.0.1:6379> zrange ptxlist 0 -1
1) "key3"
2) "key5"
```
可以看到通过采取sorted list之后，是可以满足需求的。

当然清理超时的记录是要在应用程序里实现,比如每5秒检查并清除12小时之前的记录，
用`golang`的话可以这么写：

```go
	ticker := time.NewTicker(5*time.Seocnd)
	defer func() {
		ticker.Stop()
	}()
	for {
		select {
		case <-ticker.C:
			redis.ZRemRangeScore(mylist, 0, time.Now().Unix() - 12*3600)
		}
	}
```

**后记**
最后我还是没有采用辅助sorted list的方式，因为有了新的需求，需要按未确认交易里面某些值进行排序，这样的话再纯依靠redis就比较复杂，redis就只保存了k/v pairs键值对。毕竟redis的访问速度很快，未确认交易数量也就100k左右，索性每隔10秒全部取到程序里，在程序里对未确认交易进行排序。