---
title: "EOS 主网节点搭建"
date: 2018-08-02T13:40:58+08:00
draft: false
tags: ["EOS"]
---

一旦eos代码下载完成，编译完成，可使用下列代码来搭建一个主网同步节点。

**注意**：这里搭建的是一个non-producer节点。

EOS 主网setup script 参考自 https://medium.com/@cc32d9/running-a-full-eos-mainnet-node-d7da2d3b154f

```
mkdir $HOME/node
cd $HOME/node
cat >config.ini <<'EOF'
chain-state-db-size-mb = 12288
get-transactions-time-limit = 3
block-log-dir = "blocks"
http-server-address = 0.0.0.0:8888
p2p-listen-endpoint = 0.0.0.0:9856
p2p-server-address = :9856
allowed-connection = any
log-level-net-plugin = info
max-clients = 100
connection-cleanup-period = 30
network-version-match = 0
sync-fetch-span = 2000
enable-stale-production = false
required-participation = 33
plugin = eosio::chain_plugin
plugin = eosio::chain_api_plugin
plugin = eosio::history_plugin
plugin = eosio::history_api_plugin
filter-on = *
plugin = eosio::db_size_api_plugin
p2p-peer-address = 106.10.42.238:9876
p2p-peer-address = 130.211.59.178:9876
p2p-peer-address = 159.65.214.150:9876
p2p-peer-address = 18.191.33.148:59876
p2p-peer-address = 185.253.188.1:19876
p2p-peer-address = 185.253.188.1:19877
p2p-peer-address = 34.252.209.121:5556
p2p-peer-address = 807534da.eosnodeone.io:19872
p2p-peer-address = api-full1.eoseoul.io:9876
p2p-peer-address = api-full2.eoseoul.io:9876
p2p-peer-address = api.eosuk.io:12000
p2p-peer-address = boot.eostitan.com:9876
p2p-peer-address = bp.cryptolions.io:9876
p2p-peer-address = bp.eosbeijing.one:8080
p2p-peer-address = bp.eosmedi.com:9877
p2p-peer-address = bp.libertyblock.io:9800
p2p-peer-address = br.eosrio.io:9876
p2p-peer-address = dc1.eosemerge.io:9876
p2p-peer-address = eos-seed-de.privex.io:9876
p2p-peer-address = eos.nodepacific.com:9876
p2p-peer-address = eos.staked.us:9870
p2p-peer-address = eosapi.blockmatrix.network:13546
p2p-peer-address = eosboot.chainrift.com:9876
p2p-peer-address = eosbp.buildteam.io:8532
p2p-peer-address = eu-west-nl.eosamsterdam.net:9876
p2p-peer-address = eu1.eosdac.io:49876
p2p-peer-address = fn001.eossv.org:443
p2p-peer-address = fullnode.eoslaomao.com:443
p2p-peer-address = m.eosvibes.io:9876
p2p-peer-address = mainnet-eos.wancloud.cloud:55576
p2p-peer-address = mainnet.eoscalgary.io:5222
p2p-peer-address = mainnet.eoseco.com:10010
p2p-peer-address = mainnet.eosoasis.io:9876
p2p-peer-address = mainnet.eospay.host:19876
p2p-peer-address = mars.fnp2p.eosbixin.com:443
p2p-peer-address = node.eosflare.io:1883
p2p-peer-address = node.eosio.lt:9878
p2p-peer-address = node.eosmeso.io:9876
p2p-peer-address = node1.eoscannon.io:59876
p2p-peer-address = node1.eosnewyork.io:6987
p2p-peer-address = node2.eosnewyork.io:6987
p2p-peer-address = p.jeda.one:3322
p2p-peer-address = p2p.eos.bitspace.no:9876
p2p-peer-address = p2p.eosdetroit.io:3018
p2p-peer-address = p2p.eosholding.ca:9876
p2p-peer-address = p2p.eosio.cr:1976
p2p-peer-address = p2p.eosio.cr:5418
p2p-peer-address = p2p.genereos.io:9876
p2p-peer-address = p2p.mainnet.eosgermany.online:9876
p2p-peer-address = p2p.mainnet.eospace.io:88
p2p-peer-address = p2p.saltblock.io:19876
p2p-peer-address = p2p.unlimitedeos.com:15555
p2p-peer-address = peer.eosjrr.io:9876
p2p-peer-address = peer.eosn.io:9876
p2p-peer-address = peer.main.alohaeos.com:9876
p2p-peer-address = peer1.mainnet.helloeos.com.cn:80
p2p-peer-address = peer2.eosthu.com:8080
p2p-peer-address = peer2.mainnet.helloeos.com.cn:80
p2p-peer-address = peering.mainnet.eoscanada.com:9876
p2p-peer-address = peering1.mainnet.eosasia.one:80
p2p-peer-address = peering2.mainnet.eosasia.one:80
p2p-peer-address = pub0.eosys.io:6637
p2p-peer-address = pub1.eostheworld.io:9876
p2p-peer-address = pub1.eosys.io:6637
p2p-peer-address = pub2.eostheworld.io:9876
p2p-peer-address = publicnode.cypherglass.com:9876
p2p-peer-address = seed1.greymass.com:9876
p2p-peer-address = seed2.greymass.com:9876
EOF
wget https://raw.githubusercontent.com/CryptoLions/EOS-MainNet/master/genesis.json
screen -L 
nodeos --data-dir . --config-dir . --genesis-json genesis.json
## Ctrl-a Ctrl-d to detach from screen
## screen -R to re-attach to the screen
```

> 吐槽下，速度真的奇慢无比....

## EOS 主网搭建的几个坑
1. EOS 官方github 无效
官方source code下载的EOS https://github.com/EOSIO/eos
用来搭建主网，反正我试了好几个版本，浪费了好几天，基本都是同步到不同的高度crash，然后根据社区的replay什么的都不好使。。。
可参考https://github.com/EOS-Mainnet 搭建教程

2. node的内存一定要超过7个G， 当然也可以修改build_eosio.sh来跳过检验。
3. node的硬盘要超过20个G， 可修改build_eosio.sh跳过，但考虑到作为主网同步机器，还是至少20G的好。
4. 主网搭建完毕后
要使用`get transaction`, `get actions`， 记得`config.ini`里面 设置`filter-on=*`

