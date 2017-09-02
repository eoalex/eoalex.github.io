---
title: '以太坊搭建本地私有链'
date: 2017-09-02 21:58:34
tags:
  - 区块链
  - 以太坊
categories:
  - 云计算及虚拟化
---

## 1. 介绍
本文我们演示如何使用以太坊客户端搭建以太坊的私有链。
环境：Ubuntu 16.04

## 2. 步骤
### 2.1 安装客户端
从以下地址下载以太坊Linux包

	https://github.com/ethereum/mist/releases

安装
	
	sudo apt-get -f install libappindicator1 libindicator7
	sudo dpkg -i Mist-linux64-0-9-0.deb

设置geth执行路径，编辑.bashrc

	export PATH=$PATH:～／.config/Mist/binaries/Geth/unpacked
### 2.2 建立目录和genesis.json
创建目录

	sudo mkdir /opt/mychains

cd /opt/mychains，编辑genesis.json

	{
    "nonce": "0x0000000000000042",     
    "timestamp": "0x00",
    "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "extraData": "0x00",     
    "gasLimit": "0x8000000",     
    "difficulty": "0x400",
    "mixhash": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "coinbase": "0x3333333333333333333333333333333333333333",     
    "alloc": {
     },
    "config": {
        "chainId": 15,
        "homesteadBlock": 0,
        "eip155Block": 0,
        "eip158Block": 0
   	 }
	}
	
Mixhash
一个256位的哈希值，和nonce配合，一起用来证明在区块链上已经做了足够的计算量（工作证明）。这个nonce 和 mixhash 的组成，必须满足一个在黄皮书中所描述的数学上的条件，黄皮书 4.3.4。
Nonce
一个64位的哈希值，和mixhash配合，一起用来证明在区块链上已经做了足够的计算量（工作证明）
Difficulty
定义挖矿的目标，可以用上一个区块的难度值和时间戳计算出来，值越高，矿工越难挖到区块
Alloc 预先填入一些钱包和余额
Coinbase
160位的钱包地址。在创世区块中可以被定义成任何的地址，因为当每挖到一个区块的时候，这个值会变成矿工的etherbase地址
Timestamp  一个unix的time()函数的输出值，时间戳
extraData  32字节长度，可以为私有链留下一些信息，如你的姓名等，用以证明这个私有链是你创建的
gasLimit   当前链，一个区块所能消耗的gas上限

### 2.3 初始化创世块

	geth --identity "mydev1" --rpc --rpccorsdomain "*" --datadir "./mydev1" --rpcapi "db,eth,net,web3" --networkid 100 init "./genesis.json"
	
	WARN [09-02|21:15:15] No etherbase set and no accounts found as default 
	INFO [09-02|21:15:15] Allocated cache and file handles database=/opt/mychains/mydev1/geth/chaindata cache=16 handles=16
	INFO [09-02|21:15:15] Writing custom genesis block 
	INFO [09-02|21:15:15] Successfully wrote genesis state database=chaindata  hash=6650a0…b5c158
	INFO [09-02|21:15:15] Allocated cache and file handles database=/opt/mychains/mydev1/geth/lightchaindata cache=16 handles=16
	INFO [09-02|21:15:15] Writing custom genesis block 
	INFO [09-02|21:15:15] Successfully wrote genesis state atabase=lightchaindata hash=6650a0…b5c158
 
### 2.4 启动私有链

	geth --datadir "./mydev1" --identity "mydev1" --rpccorsdomain "*" --networkid 100 console
	
	WARN [09-02|21:21:00] No etherbase set and no accounts found as default 
	INFO [09-02|21:21:00] Starting peer-to-peer node               instance=Geth/mydev1/v1.6.6-stable-10a45cb5/linux-amd64/go1.8.3
	INFO [09-02|21:21:00] Allocated cache and file handles         database=/opt/mychains/mydev1/geth/chaindata cache=128 handles=1024
	WARN [09-02|21:21:00] Upgrading chain database to use sequential keys 
	INFO [09-02|21:21:00] Initialised chain configuration          config="{ChainID: 15 Homestead: 0 DAO: <nil> DAOSupport: false EIP150: <nil> EIP155: 0 EIP158: 0 Metropolis: <nil> Engine: unknown}"
	INFO [09-02|21:21:00] Disk storage enabled for ethash caches   dir=/opt/mychains/mydev1/geth/ethash count=3
	INFO [09-02|21:21:00] Disk storage enabled for ethash DAGs     dir=/home/alex/.ethash               count=2
	WARN [09-02|21:21:00] Upgrading db log bloom bins 
	INFO [09-02|21:21:00] Bloom-bin upgrade completed              elapsed=248.19µs
	INFO [09-02|21:21:00] Initialising Ethereum protocol           versions="[63 62]" network=100
	INFO [09-02|21:21:00] Loaded most recent local header          number=0 hash=6650a0…b5c158 td=1024
	INFO [09-02|21:21:00] Loaded most recent local full block      number=0 hash=6650a0…b5c158 td=1024
	INFO [09-02|21:21:00] Loaded most recent local fast block      number=0 hash=6650a0…b5c158 td=1024
	INFO [09-02|21:21:00] Starting P2P networking 
	INFO [09-02|21:21:00] Database conversion successful 
	INFO [09-02|21:21:00] UDP listener up                          self=enode://4203871e783238ac002006ded5de0627d3bd05e76403833ba4c2b3ecec0e98cecae5d367123b1c1aadb6ce3cc59491de3cb78d4ff469a452beb49ce2de9f0c01@192.168.1.5:30303
	INFO [09-02|21:21:00] RLPx listener up                         self=enode://4203871e783238ac002006ded5de0627d3bd05e76403833ba4c2b3ecec0e98cecae5d367123b1c1aadb6ce3cc59491de3cb78d4ff469a452beb49ce2de9f0c01@192.168.1.5:30303
	INFO [09-02|21:21:00] IPC endpoint opened: /opt/mychains/mydev1/geth.ipc 
	Welcome to the Geth JavaScript console!

	instance: Geth/mydev1/v1.6.6-stable-10a45cb5/linux-amd64/go1.8.3
 	modules: admin:1.0 debug:1.0 eth:1.0 miner:1.0 net:1.0 personal:1.0 rpc:1.0 txpool:1.0 web3:1.0

	INFO [09-02|21:21:01] Mapped network port                      proto=udp extport=30303 intport=30303 interface=NAT-PMP(192.168.199.1)
	INFO [09-02|21:21:01] Mapped network port                      proto=tcp extport=30303 intport=30303 interface=NAT-PMP(192.168.199.1)

### 2.5 创建账号并启动挖矿

创建账号personal.newAccount("123456")
	
	"0x852ffa22f80d5ce869b94bcf60fbe60bb867415a"
	INFO [09-02|21:31:48] New wallet appeared url=keystore:///opt/mychains/mydev1… status=Locked

启动挖矿miner.start()
	
	INFO [09-02|21:33:09] Updated mining threads  threads=0
	INFO [09-02|21:33:09] Transaction pool price threshold updated price=18000000000	null
	INFO [09-02|21:33:09] Starting mining operation 
	INFO [09-02|21:33:09] Commit new mining work number=1 txs=0 uncles=0 elapsed=131.912µs
	INFO [09-02|21:33:25] Generating DAG in progress  epoch=1 percentage=0 elapsed=13.305s
	INFO [09-02|21:33:37] Generating DAG in progress  epoch=1 percentage=1 elapsed=24.781s
	INFO [09-02|21:33:48] Generating DAG in progress  epoch=1 percentage=2 elapsed=36.059s
	INFO [09-02|21:33:59] Generating DAG in progress  epoch=1 percentage=3 elapsed=47.375s
	INFO [09-02|21:34:10] Generating DAG in progress  epoch=1 percentage=4 elapsed=58.127s
	INFO [09-02|21:34:15] Successfully sealed new block  number=1 hash=1a9b10…5d488d
	INFO [09-02|21:34:15] 🔨 mined potential block  number=1 hash=1a9b10…5d488d
	INFO [09-02|21:34:15] Commit new mining work  number=2 txs=0 uncles=0 elapsed=5.490ms
	INFO [09-02|21:34:17] Successfully sealed new block  number=2 hash=8fc2dc…501949
	INFO [09-02|21:34:17] 🔨 mined potential block  number=2 hash=8fc2dc…501949
	INFO [09-02|21:34:17] Commit new mining work  number=3 txs=0 uncles=0 elapsed=176.082µs
	INFO [09-02|21:34:21] Generating DAG in progress  epoch=1 percentage=5 elapsed=1m9.380s


### 2.6 检验挖矿是否成功
查看主账户的以太币数量

	acc0=eth.accounts[0]
	eth.getBalance(acc0)

结果只要不为0，那就说明挖矿成功！
	
	40000000000000000000

