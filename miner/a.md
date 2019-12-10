# miner
# 主要流程
# 涉及到的数据结构
# 主要函数

## newWorker
进行初始化。
- 订阅NewTxsEvent
```
	// Subscribe NewTxsEvent for tx pool
	worker.txsSub = eth.TxPool().SubscribeNewTxsEvent(worker.txsCh)
```
- 订阅 chainHeadSub、chainSideSub
```
// Subscribe events for blockchain
	worker.chainHeadSub = eth.BlockChain().SubscribeChainHeadEvent(worker.chainHeadCh)
	worker.chainSideSub = eth.BlockChain().SubscribeChainSideEvent(worker.chainSideCh)
```
- 开启4个协程
```
	go worker.mainLoop()
	go worker.newWorkLoop(recommit)
	go worker.resultLoop()
	go worker.taskLoop()
```
- 初始化。给startCh传入一个消息。
```
 // Submit first work to initialize pending state.
	if init {
		worker.startCh <- struct{}{}
	}
	return worker
```

# 协程一号 mainLoop
- for构成死循环，始终等待通道中的消息到来。
具体信号如下：
newWorkCh-->运行commitNewWork函数

## commitNewWork 函数
- 防止挖矿挖太快
```
// this will ensure we're not going off too far in the future
	if now := time.Now().Unix(); timestamp > now+1 {
		wait := time.Duration(timestamp-now) * time.Second
		log.Info("Mining too far in the future", "wait", common.PrettyDuration(wait))
		time.Sleep(wait)
	}
```
- 初始化header
```
num := parent.Number()
	header := &types.Header{
		ParentHash: parent.Hash(),
		Number:     num.Add(num, common.Big1),
		GasLimit:   core.CalcGasLimit(parent, w.config.GasFloor, w.config.GasCeil),
		Extra:      w.extra,
		Time:       uint64(timestamp),
	}
```
- 处理叔块
```
	// Accumulate the uncles for the current block
	uncles := make([]*types.Header, 0, 2)
	commitUncles := func(blocks map[common.Hash]*types.Block) {
		// Clean up stale uncle blocks first
		for hash, uncle := range blocks {
			if uncle.NumberU64()+staleThreshold <= header.Number.Uint64() {
				delete(blocks, hash)
			}
		}
		for hash, uncle := range blocks {
			if len(uncles) == 2 {
				break
			}
			if err := w.commitUncle(env, uncle.Header()); err != nil {
				log.Trace("Possible uncle rejected", "hash", hash, "reason", err)
			} else {
				log.Debug("Committing new uncle to block", "hash", hash)
				uncles = append(uncles, uncle.Header())
			}
		}
	}
	// Prefer to locally generated uncle
	commitUncles(w.localUncles)
	commitUncles(w.remoteUncles)
```
- 填充交易
将pending中的交易分为locals与remotes
分别填充到localTxs、remoteTxs中

- 退出协程以后需要取消订阅
```
	defer w.txsSub.Unsubscribe()
	defer w.chainHeadSub.Unsubscribe()
	defer w.chainSideSub.Unsubscribe()
```

# commit
commit函数用于提交状态修改

- taskCh通道
处理接收到的新task

- exitCh通道
worker退出。

- 计算矿工费
此处计算了所有交易的gas。
fees存着总矿工费。feesEth与feesWei区别在于单位。

```
			feesWei := new(big.Int)
			for i, tx := range block.Transactions() {
				feesWei.Add(feesWei, new(big.Int).Mul(new(big.Int).SetUint64(receipts[i].GasUsed), tx.GasPrice()))
			}
			feesEth := new(big.Float).Quo(new(big.Float).SetInt(feesWei), new(big.Float).SetInt(big.NewInt(params.Ether)))

```
- 检测是否需要更新
```
if update {
		w.updateSnapshot()
	}
```
updateSnapshot函数更新快照。





