---
title: "Eth Explorer 开发回顾"
date: 2018-09-27T10:14:13+08:00
draft: false
---

从Evernote工作笔记上来看，开始着手调研以太坊浏览器是3月底，正式开始设计实现浏览器是4月份，到现在浏览器开发，parser 跟ethx 两个服务，也差不多正好半年。

写个小结记录下这段有意义的时光。。。

## **初版 原型**

初版的主要设计需求：

1. 支持获取blocks，transactions, uncles, erc20 transactions 等链上数据。
2. 能准确同步历史数据。
3. 能监听最新block。
4. 能监听最新的tx pool。

关于获取链上历史数据部分，一开始看了[BlockSci](https://github.com/citp/BlockSci)受了启发，了解到获取历史数据主要有两种方式

1. 通过rpc直接从节点获取
2. 直接从节点数据库拉取

方法二不用说肯定是最高效的，比方法一省了一个跟节点1交互的过程。
也是我一开始主攻方向。花了点时间总算能从节点数据库里面获得自己感兴趣的数据，但刚出炉的方法没捂热，就很悲剧的发现，节点数据库采用的是`levelDB`，
只支持单进程访问。意味着用方法2拉数据的时候，eth节点得停止运行，可是有部分数据是必须通过rpc获取，不能停啊

方法2, 卒...

于是转攻方法一: `数据都是直接通过rpc访问eth获取`

方法一表现的还不错，速度还行，通过测试网发现速度可以啊，单线程大概每秒得有30个block的import，再算算600万/30/3600 差不多得55个小时，问问时下行情，貌似可行啊...
更不用说多线程了，当然，后来我知道自己天真了...

抛开后续困扰不谈，初版实现了下列功能：

1. 能够从eth 主链上获取`blocks`,`transactions`,`uncles`并解析这三类信息。
2. eth 大部分json rpc的封装，用golang实现。
3. gRPC+restfulAPI 结合，说白了就是一个服务同时接受gRPC跟https访问。

初版功能不是很多，但真的是一个一个坑过来的，但解决后也确实都成了自己的知识点。

* eth各种数据类型跟parser里自己定义的类型之间的转换，这一部分来回坑了我很多次，还是怪自己定义数据类型是随意跟不熟练。
* eth各种 json rpc 实现跟调用注意事项
* json marshal 跟 unmarshal 高阶玩法
* golang里面数据类型跟mysql交互的orm实现，这里尝试过gorm，但gorm对big.Int支持不佳，遂放弃，最后自己对mysql各种语句进行了封装。
* gRPC 跟 restful apis的结合
* 多线程实现方面的坑跟经验总结
    1. MySQL connection 有连接数量上限,会返回 `too many connections` 错误
    2. 虽然golang的goroutines很轻量级，但是随手写goroutines，尤其是跟eth或者mysql交互部分，很快就将系统可用端口占光。worker pool真的好用。

我看了下git的历史，从设计到实现到运行，差不多一个月吧。

实现完之后，试着导入测试网络，一开始看到每秒大概有上百个blocks导入，心里还是非常开心的，测试网大概近400万个blocks，照这个时间，也就10小时左右能搞定。
但实际情况是，在导入到100万之后，速度几乎停滞不前。。。

![default](https://user-images.githubusercontent.com/1768412/46524268-d488af80-c8ba-11e8-8763-e73c83812457.jpeg)

初版的问题：

1. 所有数据都是从eth rpc获取，速度非常慢，如果后续有新的需求或者说parser部分出错，重新导入绝对是个大工程。
2. 数据类型偏少，需要查询账户相关信息。

## **版本 v0.0.1**

原型主要问题就是在于从eth拿历史数据非常慢，必须有更好的方式获取历史数据.
想到的思路：

1. 起多台eth节点，作负载均衡, 按理论，获取速度可以有线性提升
2. 将eth数据储存到本地缓存，放入到读写更快的系统中,便于今后的开发（痛一次，今后就不痛了...)

方案1是最先想实行的，跟同事提过一两次，没啥反应，加上自己对多台eth到底能提升多少性能，也没啥逼数...算了（**新员工，脸皮薄**）

那就大刀阔斧的搞方案2吧. **（将单机处理能力性能提升到巅峰，想想也是酷，增加对各个模块的了解，没毛病）**

方案2 主要问题就是挑选性能更好的数据服务。
需要考虑的几个点

1. 最好是go语言支持，内嵌数据库，不需要额外的service，多一个service需要多考虑很多因素...
2. 读写速度都要快，多线程读写支持。读的速度要求大于写的速度。
3. SSD 读写优化

这一阶段看了些经典的KV数据库以及缓存性数据库,优先看了下 levelDB golang的实现版：https://github.com/golang/leveldb，后来，网上稍微搜了搜，大家说rocksDB更好，在写跟更并发方面优化做了更多。但rocksDB 是c++写的，golang需要采用的话，需要使用cgo，说实话，不是很喜欢cgo。

后来恰巧在geth看到有人提交了一个pr 用badgerDB 代替levelDB， 这引起了我很大兴趣。链接：https://github.com/ethereum/go-ethereum/issues/15717

稍稍，做了些badger的研究，发现badgerDB 读写速度灰常牛逼，https://blog.dgraph.io/post/badger/ badgerDB 这篇官博里做的benchMark测试算是吊打rocksDB。遂入坑。。。嗯，我咋这么容易就入坑的...

其实中间还看了下[TiDB](https://github.com/pingcap/tidb)，[boltDB](https://github.com/boltdb/bolt), 但从已有的文章来看，单机性能都比不上badgerDB。

选定数据库后, 第二阶段的主要任务可以归纳成

1. 将eth历史数据导入到badgerDB
2. 修改原型架构，分为两部分
    1. 历史数据部分从badgerDB 导入
    2. 最新数据从eth节点导入
3. 支持 erc20 识别
4. 支持 internal transactions
5. 支持accounts 余额查询，按余额排序

其中支持erc20识别是老板来的，通过eventLogs签名识别，老板老中医威武霸气，对以太坊的熟练度确实高。

同步eth主网全节点，采用parity， achive+fatdb+trace模式耗时1周。

从eth导入历史数据到badger，600万个block209小时，心里再度郁闷下，明明知道eth处理query速度不快，为什么不起多个eth节点！！！
![image](https://user-images.githubusercontent.com/1768412/46520762-b9fd0900-c8af-11e8-9e23-1925d8abdb5d.png)

**后悔没有搭建多个eth节点+1**

然后就是新功能开发，采坑...

1. accounts balance计算
2. erc20 transactions 的查询支持，以及用户的token balance追踪
3. internal transaction的查询支持

其中account balance计算，表示手搓党真的伤不起，这里我赌5毛，etherscan的account balance肯定是通过json rpc获取的，这个我来回真算了好久，一直有问题...

到后来所有功能支持，到第一版主网成功同步完毕，差不多是7月底了。。。

v0.0.1，抛开tokens的一些功能限制，etherscan有的功能基本都齐备了

1. 数据类型增加，支持`blocks`, `uncles`, `transactions`, `accounts`, `erc20_transactions`, `internal_transactions`, `contracts`
2. ethx 实现完成，支持上述数据类型的多种api查询
3. 可根据机器配置，自定义block porter的数量，理论上block porter越多，数据导入越快.
4. 支持回滚

写入历史数据部分：
![image](https://user-images.githubusercontent.com/1768412/46521180-48be5580-c8b1-11e8-8985-e1dc07e7bd80.png)

整体来看，当时写入速度还是很均衡的，基本维持在38M 左右，耗时54小时左右能将全部历史数据写到mysql数据库。
再加上索引创立时间13小时，因此全部耗时在70小时左右，可以接受。
全部数据导入到MySQL, 算索引，大小略超出1T。

v0.0.1 问题：

1. 容灾性不强，一旦依赖的三方服务down，比如RDS, Parity，或者Redis，那么数据大概率出现污染
2. Token相关查询支持缺失

## **版本v0.0.2** ##

> 等到产品经理上线了...不容易

要支持token balance查询，按token balance还要排序。。。我了个天，以太坊智能合约是图灵完备的，account的eth balance还能通过对交易的理解，手搓上去，
但token balance，对于好多不按套路写的货币，真的算不准，臣真的无能为力啊。。。

只能堕落，采用eth json rpc来获取account balance了，但是一想到eth rpc的速度，想到近亿的query，心里凉飕飕的。。。

**后悔没有搭建多个eth节点+2**

v0.0.2 提供了更好的容灾性，说白了就是加了更多的retry，回滚重试，异常捕获等

v0.0.2 parser有进一步优化，为了将parser对历史数据解析方面速度最大化，对MySQL, Parity， Badger的瓶颈跟性能有了长足的认识跟了解，

同时，比起第一版数据库层面也做了很多优化跟改良。。。

* 优化数据库表格分表
* 优化数据库统一命名规则
* 优化api查询规则

第二版的数据库写入速度是这样的，基本上逼近网络传输速度上限，所有的历史数据基本能在12小时倒完。
![image](https://user-images.githubusercontent.com/1768412/46522661-f9c6ef00-c8b5-11e8-9177-10120389995c.png)

但12小时只是个开始，后续50个小时是漫长的等待eth 返回所有accounts的eth balance跟token balance...
**后悔没有搭建多个eth节点+3**

## **总结**

parser 解析历史数据基本上现在达到了单机性能极致，进一步提升速度只能让运维大哥们搭建更多的以太坊节点了。

整个开发过程真的踩了挺多坑的也学习了很多，每次有新的idea也是毫不犹豫的回去立马实施，并且能不断的反思进步，基本上整个开发流程中想到的idea跟优化自己都
试了一遍，好几次都是半夜有个想法就爬起来，直接干到凌晨4，5点。

同时也是因为改的频繁，中间出了几次regression...**这里又不得不谈测试的重要性**

1. 加强了对不同数据库的理解，优劣，MySQL 算是入门了吧，会写点索引了，知道部分优化，知道一些坑了，知道一些瓶颈了，今年必须把《高性能MySQL》看完
2. golang 编程能力进一步加强
3. eth理解入门
4. 下次系统设计的时候，一定要 **尽早** **准确** 地了解各个模块中的瓶颈，性能优劣。
5. 事实证明grpc+restful api不好用，限制了api语句查询灵活性，外加 **grpc 组内伙伴根本没人用啊** （暂时）
