# worker
worker.go属于package miner
# 主要流程
# 涉及到的数据结构

# 主要函数

## newWorker
```
func newWorker(config *Config, chainConfig *params.ChainConfig, engine consensus.Engine, eth Backend, mux *event.TypeMux, isLocalBlock func(*types.Block) bool, init bool) *worker {
```
创建worker,开启四个协程
```
	go worker.mainLoop()
	go worker.newWorkLoop(recommit)
	go worker.resultLoop()
	go worker.taskLoop()
```

## mainLoop 函数
根据收到的消息生成封装任务
消息如下：
```
w.newWorkCh
w.chainSideCh
w.txsCh
w.exitCh
w.txsSub.Err()
w.chainHeadSub.Err()
w.chainSideSub.Err()
```


- newWorkCh
接受newWorkCh消息提交work.


- chainSideCh
剔除掉重复的块->添加叔块->生成新的主链上的块

- txsCh
添加交易到pending中
调用commitTransactions提交交易

- exitCh:
- txsSub.Err():
- w.txsSub.Err()
- w.chainHeadSub.Err()
- w.chainSideSub.Err()
系统退出。

# commitTransaction
调用ApplyTransaction（core/state_processor.go）
	将交易状态写入state，完成receipt的设置
	```
	// Set the receipt logs and create a bloom for filtering
	receipt.Logs = statedb.GetLogs(tx.Hash())
	receipt.Bloom = types.CreateBloom(types.Receipts{receipt})
	receipt.BlockHash = statedb.BlockHash()
	receipt.BlockNumber = header.Number
	```

将交易加入txs列表
将receipt加入receipts列表

