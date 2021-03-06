---
title: "ETH包含多个节点私链搭建"
date: 2018-03-28T10:24:55+08:00
draft: false
author: "xlk3099"
categories: ["ETH"]
tags: ["eth","blockchain"]
---
网上搜了下ETH私链搭建教程，大多都是单个节点，关于多个节点之间的交互搭建，网上信息不是很多。虽说ETH官方文档比起绝大多数开源社区好多了，但有些信息也有点过时。

当然用docker搭建集群包含多个节点,但测试的话，两个node，本地运行更为方便。

_前提_：安装官方geth

1. 创世模块搭建

    每一个区块链都有自己的创世区块，我们需要一个创世模块配置文件（genesis.json) 来设定我们的私链信息，挖矿难度，初始分配等等。
    通常一个genesis.json包含以下信息：
    1. `config`
        * `chainID`: 指定私链ID， 这里我们定义成10086
        * `homesteadBlock`: homestead 是ETH 第二个major verision 也是第一个production released version。因为目前主链已经是在homestead上，所以这里设置为  0
        * `eip155Block` 跟`eip158Block`: Homestead 发布带来了了一些不兼容的protocol改变，为了兼容之前版本，需要通过EIP满足协议的hard fork。这里我们私链不包  含任何的hard fork，所以设置为0
    2. `diffuculty`: 用来控制区块生成难度或者时间，值越高，需求的计算越困难。
    3. `gasLimit`: 指定每一个block最大能够消耗的gas上限，顾名思义，影响一个block能包含的交易数量。
    4. `alloc`: 用来创建wallet并提前分配。
    打开命令行，新建一个genesis.json: ```vim genesis.json```.
    本文采用下面genesis.json文件:

    ```json
    {
        "config": {
            "chainId": 10086,
            "homesteadBlock": 0,
            "eip155Block": 0,
            "eip158Block": 0
        },
        "difficulty": "0x4000",
        "gasLimit": "0x8000000", //为了测试，这里设置的很高。
        "alloc": {} // 因为挖矿难度射的很低，此处不需要提前分配。
    }
    ```

2. 启动私链第一个节点

    利用第一步的生成的genesis.json，指定你想储存chaindata的位置，打开命令行依次输入：
    ```shell
    geth --datadir node1 init genesis.json
    ```
    指定节点连接端口为30301，设置networkid 为100861（别在意为啥是100861， 随便一个数都行 😏)
    ```shell
    geth --datadir node1 --networkid 100861 --port 30301 console
    ```
    启动正常的话，你应该会看到以下输出：
    ```shell
    INFO [03-28|14:05:39] Maximum peer count                       ETH=25 LES=0 total=25
    INFO [03-28|14:05:39] Starting peer-to-peer node               instance=Geth/v1.8.2-stable/darwin-amd64/go1.10
    INFO [03-28|14:05:39] Allocated cache and file handles           database=/Users/lekai/go/src/github.com/ethereum/go-ethereum/lekai_test/node1/geth/chaindata cache=768 handles=1024
    INFO [03-28|14:05:39] Initialised chain configuration          config="{ChainID: 10086 Homestead: 0 DAO: <nil> DAOSupport: false EIP150:   <nil> EIP155: 0 EIP158: 0 Byzantium: <nil> Constantinople: <nil> Engine: unknown}"
    INFO [03-28|14:05:39] Disk storage enabled for ethash caches     dir=/Users/lekai/go/src/github.com/ethereum/go-ethereum/lekai_test/node1/geth/ethash count=3
    INFO [03-28|14:05:39] Disk storage enabled for ethash DAGs     dir=/Users/lekai/.ethash                                                               count=2
    INFO [03-28|14:05:39] Initialising Ethereum protocol           versions="[63 62]" network=100861
    INFO [03-28|14:05:39] Loaded most recent local header          number=0 hash=cbfef6…afcbee td=16384
    INFO [03-28|14:05:39] Loaded most recent local full block      number=0 hash=cbfef6…afcbee td=16384
    INFO [03-28|14:05:39] Loaded most recent local fast block      number=0 hash=cbfef6…afcbee td=16384
    INFO [03-28|14:05:39] Loaded local transaction journal         transactions=0 dropped=0
    INFO [03-28|14:05:39] Regenerated local transaction journal    transactions=0 accounts=0
    INFO [03-28|14:05:39] Starting P2P networking
    INFO [03-28|14:05:42] Mapped network port                      proto=udp extport=30301 intport=30301 interface="UPNP IGDv1-IP1"
    INFO [03-28|14:05:42] UDP listener up                            self=enode://aba37dd4d832aebf36a0d4d279c269db8517f54b8ff15e5dd5660e6cc184a1e0c795ef39fc73ae027c5afaa6e35d494887df5a8ff580ad8905bd70ddf0a91  afc@10.34.97.15:30301
    INFO [03-28|14:05:42] RLPx listener up                           self=enode://aba37dd4d832aebf36a0d4d279c269db8517f54b8ff15e5dd5660e6cc184a1e0c795ef39fc73ae027c5afaa6e35d494887df5a8ff580ad8905bd70ddf0a91  afc@10.34.97.15:30301
    INFO [03-28|14:05:42] IPC endpoint opened                        url=/Users/lekai/go/src/github.com/ethereum/go-ethereum/lekai_test/node1/geth.ipc
    INFO [03-28|14:05:42] Mapped network port                      proto=tcp extport=30301 intport=30301 interface="UPNP IGDv1-IP1"
    ```
    上面有很多有用信息，这里留意两条：

    enode：
    ```shell
    INFO [03-28|14:05:42] UDP listener up self=enode://aba37dd4d832aebf36a0d4d279c269db8517f54b8ff15e5dd5660e6cc184a1e0c795ef39fc73ae027c5afaa6e35d494887df5a8ff580ad8905bd70ddf0a91  afc@127.0.0.1:30301
    ```

    IPC：
    ```shell
    INFO [03-28|14:05:42] IPC endpoint opened url=/Users/lekai/go/src/github.com/ethereum/go-ethereum/lekai_test/node1/geth.ipc
    ```
    **enode** 用来告知其它node跟他连接，**IPC** 是为了方便我们进行一些本地测试。

3. 启动第二个节点

    保持第一个node运行，本文指定第二个节点端口为30302,networkID 跟第一个保持一致 100861
    **_Note_**: 在私链环境下，每次启动一个新的节点，都需要以下两步：
    * 用同一个genesis.json初始创世模块
    * 启动时需要指定同样networkID

    打开新的命令行命令行，依次输入：

    ```shell
    geth --datadir node2 init genesis.json
    ```
    ```shell
    geth --datadir node2 --networkid 100861 --port 30302
    ```
    命令行输出
    ```shell
    INFO [03-28|14:10:51] Maximum peer count                       ETH=25 LES=0 total=25
    INFO [03-28|14:10:51] Starting peer-to-peer node               instance=Geth/v1.8.2-stable/darwin-amd64/go1.10
    INFO [03-28|14:10:51] Allocated cache and file handles         database=/Users/lekai/go/src/github.com/ethereum/go-ethereum/lekai_test/node2/geth/chaindata cache=768 handles=1024
    WARN [03-28|14:10:51] Upgrading database to use lookup entries
    INFO [03-28|14:10:51] Database deduplication successful        deduped=0
    INFO [03-28|14:10:51] Initialised chain configuration          config="{ChainID: 10086 Homestead: 0 DAO: <nil> DAOSupport: false EIP150: <nil> EIP155: 0 EIP158: 0 Byzantium: <nil> Constantinople: <nil> Engine: unknown}"
    INFO [03-28|14:10:51] Disk storage enabled for ethash caches   dir=/Users/lekai/go/src/github.com/ethereum/go-ethereum/lekai_test/node2/geth/ethash count=3
    INFO [03-28|14:10:51] Disk storage enabled for ethash DAGs     dir=/Users/lekai/.ethash                                                             count=2
    INFO [03-28|14:10:51] Initialising Ethereum protocol           versions="[63 62]" network=100861
    INFO [03-28|14:10:51] Loaded most recent local header          number=0 hash=cbfef6…afcbee td=16384
    INFO [03-28|14:10:51] Loaded most recent local full block      number=0 hash=cbfef6…afcbee td=16384
    INFO [03-28|14:10:51] Loaded most recent local fast block      number=0 hash=cbfef6…afcbee td=16384
    INFO [03-28|14:10:51] Regenerated local transaction journal    transactions=0 accounts=0
    INFO [03-28|14:10:51] Starting P2P networking
    INFO [03-28|14:10:53] Mapped network port                      proto=udp extport=30302 intport=30302 interface="UPNP IGDv1-IP1"
    INFO [03-28|14:10:53] UDP listener up                          self=enode://40fd51f9857071bee5c0115831aaaf7ebe2574af7cf91c014f207a83dfc73e8d4f1abe6e029552c7004ea495a6d6421901a1040510fb98592e96f7e369af2dc2@10.34.97.15:30302
    INFO [03-28|14:10:53] RLPx listener up                         self=enode://40fd51f9857071bee5c0115831aaaf7ebe2574af7cf91c014f207a83dfc73e8d4f1abe6e029552c7004ea495a6d6421901a1040510fb98592e96f7e369af2dc2@10.34.97.15:30302
    INFO [03-28|14:10:53] IPC endpoint opened                      url=/Users/lekai/go/src/github.com/ethereum/go-ethereum/lekai_test/node2/geth.ipc
    INFO [03-28|14:10:54] Mapped network port                      proto=tcp extport=30302 intport=30302 interface="UPNP IGDv1-IP1"
    ```

4. 利用IPC访问节点控制台

    在geth里，我们可以使用IPC来访问节点控制台进行一系列查询以及挖矿操作。在启动第一第二个节点的时候，输出信息里都有IPC位置信息。

    保持上述两个节点运行同时打开新的命令窗口：

    访问第节点1控制台：
    ```shell
    geth attach /Users/lekai/go/src/github.com/ethereum/go-ethereum/lekai_test/node1/geth.ipc
    ```
    让节点1连接节点2:
    ```shell
    > admin.addPeer("enode://40fd51f9857071bee5c0115831aaaf7ebe2574af7cf91c014f207a83dfc73e8d4f1abe6e029552c7004ea495a6d6421901a1040510fb98592e96f7e369af2dc2@127.0.0.1:30302")
    true
    ```
    返回值为true的话，意味着连接成功。

5. 创建Account
    
    交易之前，我们需要建立两个account，利用第四步，分别打开节点一跟节点二的控制行，利用下面命令创建新的钱包：
    
    节点1：
    ```shell
    > personal.newAccount()
    Passphrase:
    Repeat passphrase:
    "0x6f60cb356a7455248278f14bff5f436f1cd05549"

    ```
    节点2：
    ```shell
    > personal.newAccount()
    Passphrase:
    Repeat passphrase:
    "0x5486090d62d36f3d2b3781f5b1936f43793a5c2e"
    ```
    当然在一个节点上，你可以创建多个account。

6. 挖矿
    
    利用eth挖矿开始之前，我们需要建立两个ethbase(coinbase),当然每个节点的默认挖矿地址是那个节点的primary account.
   
    通过IPC访问两个节点，并设置ethbase为primary account：
    
    ```shell
    miner.setEtherbase(eth.accounts[0])
    ```
    查询节点1当前account余额：
    
    ```shell
    > web3.eth.getBalance(eth.accounts[0])
    0
    ```
    不出意料，返回值为0. 因为我们还没挖矿。
    
    **开始挖矿**

    在节点1的控制台里输入：
    ```shell
    > miner.start()
    ```
    这时候你会发现，第一个节点，第二个节点都在同步更新.

    因为我们的挖矿难度设置的很低，挖30秒左右，我们停止挖矿。
    在节点1控制台里输入：
    ```shell
    > miner.stop()
    ```
    再次查询节点一account余额,会忽然发现自己超有钱：
    ```shell
    > web3.eth.getBalance(eth.accounts[0])
515000000000000000000
    ```
    一小会大概515个ETH了！！但这个当然不能流通到主链上哈哈。

    因为节点2没有挖矿，这时候我们再次查询节点2的primary account，balance还是为0：
    ```shell
    > web3.eth.getBalance("0x5486090d62d36f3d2b3781f5b1936f43793a5c2e")
    0
    ```

7. 交易

    节点1有钱了，这时候想转点给节点2，小康时代，共同富裕，这时候我们可以在节点1上发起一次Tx。

    同样，还是在节点1的控制台进行操作。

    首先定义一笔交易,因为是共同富裕嘛，所以就给节点2转250枚eth，注意TX只接受Wei为单位。
    ```shell
    > var tx = {from: "0x6f60cb356a7455248278f14bff5f436f1cd05549", to: "0x5486090d62d36f3d2b3781f5b1936f43793a5c2e", value: web3.toWei(250, "ether")}
    ```
    发起TX：
    ```shell
    > personal.sendTransaction(tx, "[YOUR_PASSPHRASE]")
    ```
    submit成功后，命令台会返回一个TX hash值，我们可以通过`txpool` 管理API来查询当前交易池状态。
    举个🌰, `txpool.content`可以用来查询当前所有pending的TX。
    ```json
    > txpool.content
    {
    pending: {
        0x6F60cb356a7455248278F14Bff5f436f1cd05549: {
        0: {
            blockHash: "0x0000000000000000000000000000000000000000000000000000000000000000",
            blockNumber: null,
            from: "0x6f60cb356a7455248278f14bff5f436f1cd05549",
            gas: "0x15f90",
            gasPrice: "0x430e23400",
            hash: "0xd656cfcb80b60df99e529f862373c3b8ae9048280c26ef1148fef60cdad1b531",
            input: "0x",
            nonce: "0x0",
            r: "0x5f67dde9479ac43f4b4eb70a4ac352258e0388a427a82effecac7eb2388a06b",
            s: "0x566e866a6bd3b49c867978bfb9d6588961ffb40e992aa479e4ac608ca458581d",
            to: "0x5486090d62d36f3d2b3781f5b1936f43793a5c2e",
            transactionIndex: "0x0",
            v: "0x4eef",
            value: "0xd8d726b7177a80000"
        }
        }
    },
    queued: {}
    }
    ```

    每一笔交易都需要挖矿来进行验证。我们这个时候可以在节点2来挖矿完成这笔验证，在节点2控制台开始挖矿：
    ```shell
    > miner.start()
    ```
    交易基本瞬间被确认，而且写入到区块中去。这时候我们再查询节点2的primary account余额，就可以发现，已经有400多枚eth了（挖矿用力过猛）
    而且节点1的余额被更新，扣除了交易款项。

_参考_:

* [官方文档](https://github.com/ethereum/go-ethereum/wiki/Private-network)
* [Setting-up-private-network-or-local-cluster](https://github.com/ethereum/go-ethereum/wiki/Setting-up-private-network-or-local-cluster)
* [ETH Management API](https://github.com/ethereum/go-ethereum/wiki/Management-APIs)