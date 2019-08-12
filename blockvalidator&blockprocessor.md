# block_validatore.go

## core/ Block_Validator.go/ValidateBody()
<pre>func (v *BlockValidator) ValidateBody(block *types.Block) error {
    // Check whether the block's known, and if not, that it's linkable
    if v.bc.HasBlockAndState(block.Hash(), block.NumberU64()) {
        return ErrKnownBlock
    }
    if !v.bc.HasBlockAndState(block.ParentHash(), block.NumberU64()-1) {
        if !v.bc.HasBlock(block.ParentHash(), block.NumberU64()-1) {
            return consensus.ErrUnknownAncestor
        }
        return consensus.ErrPrunedAncestor
    }
    // Header validity is known at this point, check the uncles and transactions
    header := block.Header()
    if err := v.engine.VerifyUncles(v.bc, block); err != nil {
        return err
    }
    if hash := types.CalcUncleHash(block.Uncles()); hash != header.UncleHash {
        return fmt.Errorf("uncle root hash mismatch: have %x, want %x", hash, header.UncleHash)
    }
    if hash := types.DeriveSha(block.Transactions()); hash != header.TxHash {
        return fmt.Errorf("transaction root hash mismatch: have %x, want %x", hash, header.TxHash)
    }
    return nil
}</pre>
这段代码主要是用来验证区块内容的。
- 首先判断当前数据库中是否已经包含了该区块，如果已经有了的话返回错误。
- 接着判断当前数据库中是否包含该区块的父块，如果没有的话返回错误。
- 然后验证叔块的有效性及其hash值，最后计算块中交易的hash值并验证是否和区块头中的hash值一致。

## core/BlockValidator.ValidateState()
<pre>func (v *BlockValidator) ValidateState(block, parent *types.Block, statedb *state.StateDB, receipts types.Receipts, usedGas uint64) error {
    header := block.Header()
    if block.GasUsed() != usedGas {
        return fmt.Errorf("invalid gas used (remote: %d local: %d)", block.GasUsed(), usedGas)
    }
    // Validate the received block's bloom with the one derived from the generated receipts.
    // For valid blocks this should always validate to true.
    rbloom := types.CreateBloom(receipts)
    if rbloom != header.Bloom {
        return fmt.Errorf("invalid bloom (remote: %x  local: %x)", header.Bloom, rbloom)
    }
    // Tre receipt Trie's root (R = (Tr [[H1, R1], ... [Hn, R1]]))
    receiptSha := types.DeriveSha(receipts)
    if receiptSha != header.ReceiptHash {
        return fmt.Errorf("invalid receipt root hash (remote: %x local: %x)", header.ReceiptHash, receiptSha)
    }
    // Validate the state root against the received state root and throw
    // an error if they don't match.
    if root := statedb.IntermediateRoot(v.config.IsEIP158(header.Number)); header.Root != root {
        return fmt.Errorf("invalid merkle root (remote: %x local: %x)", header.Root, root)
    }
    return nil
}</pre>

这部分代码主要是用来验证区块中和状态转换相关的字段是否正确，包含以下几个部分：

- 判断刚刚执行交易消耗的gas值是否和区块头中的值相同
- 根据刚刚执行交易获得的交易回执创建Bloom过滤器，判断是否和区块头中的Bloom过滤器相同（Bloom过滤器是一个2048位的字节数组）
- 判断交易回执的hash值是否和区块头中的值相同
- 计算StateDB中的MPT的Merkle Root，判断是否和区块头中的值相同

至此，区块验证流程就走完了，新区块将被写入数据库，同时更新世界状态。


# block_processor.go

## core/BlockProcessor.Process()
<pre>func (p *StateProcessor) Process(block *types.Block, statedb *state.StateDB, cfg vm.Config) (types.Receipts, []*types.Log, uint64, error) {
    var (
        receipts types.Receipts
        usedGas  = new(uint64)
        header   = block.Header()
        allLogs  []*types.Log
        gp       = new(GasPool).AddGas(block.GasLimit())
    )
    // Mutate the the block and state according to any hard-fork specs
    if p.config.DAOForkSupport && p.config.DAOForkBlock != nil && p.config.DAOForkBlock.Cmp(block.Number()) == 0 {
        misc.ApplyDAOHardFork(statedb)
    }
    // Iterate over and process the individual transactions
    for i, tx := range block.Transactions() {
        statedb.Prepare(tx.Hash(), block.Hash(), i)
        receipt, _, err := ApplyTransaction(p.config, p.bc, nil, gp, statedb, header, tx, usedGas, cfg)
        if err != nil {
            return nil, nil, 0, err
        }
        receipts = append(receipts, receipt)
        allLogs = append(allLogs, receipt.Logs...)
    }
    // Finalize the block, applying any consensus engine specific extras (e.g. block rewards)
    p.engine.Finalize(p.bc, header, statedb, block.Transactions(), block.Uncles(), receipts)

    return receipts, allLogs, *usedGas, nil
}</pre>
这段代码其实跟挖矿代码中执行交易是一模一样的，首先调用Prepare()计算难度值，然后调用ApplyTransaction()执行交易并获取交易回执和消耗的gas值，最后通过Finalize()生成区块。

值得注意的是，传进来的StateDB是父块的世界状态，执行交易会改变这些状态，为下一步验证状态转移相关的字段做准备。
