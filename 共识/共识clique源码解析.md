# 以太坊共识
# clique
实现POA(权威证明，Proof of Authority)共识算法

# 委员会成员管理
委员会成员加入、退出
通过投票实现。

# 出块规则

# 源码解析
## 初始化部分
```
func New(config *params.CliqueConfig, db ethdb.Database) *Clique
```

# 校验header
```
func (c *Clique) verifyHeader(chain consensus.ChainReader, header *types.Header, parents []*types.Header) error
```
# VerifySeal
VerifySeal检查Head是否包含签名，接受叔块并生成快照。

func (c *Clique) VerifySeal(chain consensus.ChainReader, header *types.Header)

包含：
snapshot函数恢复快照、
snapshot解析authorization key
检查signer

## 检查出块列表
检查最近一段时间出块的列表。列表内出现的signer中需要是前半段中的signer。
```
for seen, recent := range snap.Recents {
		if recent == signer {
			// Signer is among recents, only fail if the current block doesn't shift it out
			if limit := uint64(len(snap.Signers)/2 + 1); seen > number-limit {
				return errRecentlySigned
			}
		}
	}
```
## 校验出块难度。
是否轮到自己出块时的难度不同
```
if !c.fakeDiff {
		inturn := snap.inturn(header.Number.Uint64(), signer)
		if inturn && header.Difficulty.Cmp(diffInTurn) != 0 {
			return errWrongDifficulty
		}
		if !inturn && header.Difficulty.Cmp(diffNoTurn) != 0 {
			return errWrongDifficulty
		}
	}
```

## prepare
```
func (c *Clique) Prepare(chain consensus.ChainReader, header *types.Header) error
```
包括功能：
初始化header
epoch为一轮委员会的任期时间

收集所有选票信息
```
// Gather all the proposals that make sense voting on
		addresses := make([]common.Address, 0, len(c.proposals))
		for address, authorize := range c.proposals {
			if snap.validVote(address, authorize) {
				addresses = append(addresses, address)
			}
		}
```
## 统计所有选票信息
```
if len(addresses) > 0 {
			header.Coinbase = addresses[rand.Intn(len(addresses))]
			if c.proposals[header.Coinbase] {
				copy(header.Nonce[:], nonceAuthVote)
			} else {
				copy(header.Nonce[:], nonceDropVote)
			}
		}
```
## 输出
当前块号为epoch整除的时候，输出委员会所有成员的名单
```
if number%c.config.Epoch == 0 {
		for _, signer := range snap.signers() {
			header.Extra = append(header.Extra, signer[:]...)
		}
	}
```

# Finalize
```
func (c *Clique) Finalize(chain consensus.ChainReader, header *types.Header, state *state.StateDB, txs []*types.Transaction, uncles []*types.Header) 
```
Finalize函数用于执行收到的交易导致的修改。
在clique中由于没有block rewards，该函数确保叔块被丢掉，挖到叔块没有奖励。仅计算了root数值。

# Authorize
```
func (c *Clique) Authorize(signer common.Address, signFn SignerFn) 
```
换新的私钥出块

# Seal
```
func (c *Clique) Seal(chain consensus.ChainReader, block *types.Block, results chan<- *types.Block, stop <-chan struct{}) error
```
打包生成新的块

包含：
## 检测并判断是否需要delay
```
// If we're amongst the recent signers, wait for the next block
	for seen, recent := range snap.Recents {
		if recent == signer {
			// Signer is among recents, only wait if the current block doesn't shift it out
			if limit := uint64(len(snap.Signers)/2 + 1); number < limit || seen > number-limit {
				log.Info("Signed recently, must wait for others")
				return nil
			}
		}
	}
```
检测是否是最近出块列表中的signer,若是则需要等待。
若不为自己的出块轮次，则时间增加一个delay.
## 计算难度
```
	// Sweet, the protocol permits us to sign the block, wait for our time
	delay := time.Unix(int64(header.Time), 0).Sub(time.Now()) // nolint: gosimple
	if header.Difficulty.Cmp(diffNoTurn) == 0 {
		// It's not our turn explicitly to sign, delay it a bit
		wiggle := time.Duration(len(snap.Signers)/2+1) * wiggleTime
		delay += time.Duration(rand.Int63n(int64(wiggle)))

		log.Trace("Out-of-turn signing requested", "wiggle", common.PrettyDuration(wiggle))
	}
```

# CalcDifficulty
```
func CalcDifficulty(snap *Snapshot, signer common.Address) *big.Int
```
计算难度。
难度根据前一个块和当前signer计算。
即不同signer的序号不同，是否轮到当前signer出块会导致不同的出块难度。

# 返回上一个块的哈希
```
// SealHash returns the hash of a block prior to it being sealed.
func (c *Clique) SealHash(header *types.Header) common.Hash {
	return SealHash(header)
}
```