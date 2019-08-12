# Ethereum Tutorial
本文结合了一些网上资料，加上个人的原创结合而成。若有疑问，还请及时批评指出。

## 目录

- [go-ethereum代码阅读环境搭建](/go-ethereum源码阅读环境搭建.md)
- [以太坊黄皮书 符号索引](a黄皮书里面出现的所有的符号索引.md)
- [account文件解析](/accounts源码分析.md)
- build文件解析： 此文件主要用于编译安装使用
- [cmd文件解析](/cmd.md)
  - [geth](/cmd-geth.md)
- common文件：　此文件是提供系统的一些通用的工具集 (utils)
- [consensus文件解析](/consensus.md)
- console文件解析：　Console is a JavaScript interpreted runtime environment.
- contract文件： Package checkpointoracle is a an on-chain light client checkpoint oracle about contract.
- core文件源码分析
	- [types文件解析](/types.md)
	- [state文件分析](/core-state源码分析.md)
	- [core/genesis.go](/core-genesis创世区块源码分析.md)
	- [core/blockchain.go](/core-blockchain源码分析.md)
	- [core/tx_list.go & tx_journal.go](/core-txlist交易池的一些数据结构源码分析.md)
	- [core/tx_pool.go](/core-txpool交易池源码分析.md)
	- [core/block_processor.go & block_validator.go](/blockvalidator&blockprocessor.md)
	- [chain_indexer.go](/core-chain_indexer源码解析.md)
	- [bloombits源码分析](/core-bloombits源码分析.md)
	- [statetransition.go & stateprocess.go](/core-state-process源码分析.md)
	- vm 虚拟机源码分析
		- [EVM Overview](/EVMOverview.md)
		- [虚拟机堆栈和内存数据结构分析](/core-vm-stack-memory源码分析.md)
		- [虚拟机指令,跳转表,解释器源码分析](/core-vm-jumptable-instruction.md)
		- [虚拟机源码分析](/core-vm源码分析.md)
- crypto文件： 整个system涉及的有关密码学的configuration
- Dashboard: The dashboard is a data visualizer integrated into geth, intended to collect and visualize useful information of an Ethereum node. It consists of two parts: 1) The client visualizes the collected data. 2) The server collects the data, and updates the clients.
- [eth源码分析](/eth源码分析.md)
- [ethdb源码分析](/ethdb源码分析.md)
- [miner文件解析](/miner-module.md)
- [p2p源码分析](/p2p源码分析.md)
- [rlp源码解析](/rlp文件解析.md)
- [rpc源码分析](/rpc源码分析.md)
- [trie源码分析](/trie源码分析.md)
- [pow一致性算法](/pow一致性算法.md)
- [以太坊测试网络Clique_PoA介绍](/以太坊测试网络Clique_PoA介绍.md)


