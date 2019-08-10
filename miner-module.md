# miner模块
## agent.go (版本更新过后，这个文件已经被删除）
agent 是具体执行挖矿的对象。 它执行的流程就是，接受计算好了的区块头， 计算mixhash和nonce， 把挖矿好的区块头返回。

构造CpuAgent, 一般情况下不会使用CPU来进行挖矿，一般来说挖矿都是使用的专门的GPU进行挖矿。

	type CpuAgent struct {
		mu sync.Mutex
	
		workCh        chan *Work       // 接受挖矿任务的通道
		stop          chan struct{}
		quitCurrentOp chan struct{}
		returnCh      chan<- *Result   // 挖矿完成后的返回channel
	
		chain  consensus.ChainReader // 获取区块链的信息
		engine consensus.Engine      // 一致性引擎，这里指的是Pow引擎
	
		isMining int32 // isMining indicates whether the agent is currently mining
	}
	
	func NewCpuAgent(chain consensus.ChainReader, engine consensus.Engine) *CpuAgent {
		miner := &CpuAgent{
			chain:  chain,
			engine: engine,
			stop:   make(chan struct{}, 1),
			workCh: make(chan *Work, 1),
		}
		return miner
	}

设置返回值channel和得到Work的channel， 方便外界传值和得到返回信息。

	func (self *CpuAgent) Work() chan<- *Work            { return self.workCh }
	func (self *CpuAgent) SetReturnCh(ch chan<- *Result) { self.returnCh = ch }

启动和消息循环，如果已经启动挖矿，那么直接退出， 否则启动update 这个goroutine
update 从workCh接受任务，进行挖矿，或者是接受退出信息，退出。
	
	func (self *CpuAgent) Start() {
		if !atomic.CompareAndSwapInt32(&self.isMining, 0, 1) {
			return // agent already started
		}
		go self.update()
	}
	
	func (self *CpuAgent) update() {
	out:
		for {
			select {
			case work := <-self.workCh:
				self.mu.Lock()
				if self.quitCurrentOp != nil {
					close(self.quitCurrentOp)
				}
				self.quitCurrentOp = make(chan struct{})
				go self.mine(work, self.quitCurrentOp)
				self.mu.Unlock()
			case <-self.stop:
				self.mu.Lock()
				if self.quitCurrentOp != nil {
					close(self.quitCurrentOp)
					self.quitCurrentOp = nil
				}
				self.mu.Unlock()
				break out
			}
		}
	}

mine, 挖矿，调用一致性引擎进行挖矿， 如果挖矿成功，把消息发送到returnCh上面。
	
	func (self *CpuAgent) mine(work *Work, stop <-chan struct{}) {
		if result, err := self.engine.Seal(self.chain, work.Block, stop); result != nil {
			log.Info("Successfully sealed new block", "number", result.Number(), "hash", result.Hash())
			self.returnCh <- &Result{work, result}
		} else {
			if err != nil {
				log.Warn("Block sealing failed", "err", err)
			}
			self.returnCh <- nil
		}
	}
GetHashRate， 这个函数返回当前的HashRate。

	func (self *CpuAgent) GetHashRate() int64 {
		if pow, ok := self.engine.(consensus.PoW); ok {
			return int64(pow.Hashrate())
		}
		return 0
	}


## remote_agent.go (版本更新过后，这个文件已经被删除）
remote_agent 提供了一套RPC接口，可以实现远程矿工进行采矿的功能。 比如我有一个矿机，矿机内部没有运行以太坊节点，矿机首先从remote_agent获取当前的任务，然后进行挖矿计算，当挖矿完成后，提交计算结果，完成挖矿。 

数据结构和构造

	type RemoteAgent struct {
		mu sync.Mutex
	
		quitCh   chan struct{}
		workCh   chan *Work  		// 接受任务
		returnCh chan<- *Result		// 结果返回
	
		chain       consensus.ChainReader
		engine      consensus.Engine
		currentWork *Work	//当前的任务
		work        map[common.Hash]*Work // 当前还没有提交的任务，正在计算
	
		hashrateMu sync.RWMutex
		hashrate   map[common.Hash]hashrate  // 正在计算的任务的hashrate
	
		running int32 // running indicates whether the agent is active. Call atomically
	}
	
	func NewRemoteAgent(chain consensus.ChainReader, engine consensus.Engine) *RemoteAgent {
		return &RemoteAgent{
			chain:    chain,
			engine:   engine,
			work:     make(map[common.Hash]*Work),
			hashrate: make(map[common.Hash]hashrate),
		}
	}

启动和停止
	
	func (a *RemoteAgent) Start() {
		if !atomic.CompareAndSwapInt32(&a.running, 0, 1) {
			return
		}
		a.quitCh = make(chan struct{})
		a.workCh = make(chan *Work, 1)
		go a.loop(a.workCh, a.quitCh)
	}
	
	func (a *RemoteAgent) Stop() {
		if !atomic.CompareAndSwapInt32(&a.running, 1, 0) {
			return
		}
		close(a.quitCh)
		close(a.workCh)
	}
得到输入输出的channel，这个和agent.go一样。

	func (a *RemoteAgent) Work() chan<- *Work {
		return a.workCh
	}
	
	func (a *RemoteAgent) SetReturnCh(returnCh chan<- *Result) {
		a.returnCh = returnCh
	}

loop方法,和agent.go里面做的工作比较类似， 当接收到任务的时候，就存放在currentWork字段里面。 如果84秒还没有完成一个工作，那么就删除这个工作， 如果10秒没有收到hashrate的报告，那么删除这个追踪/。
	
	// loop monitors mining events on the work and quit channels, updating the internal
	// state of the rmeote miner until a termination is requested.
	//
	// Note, the reason the work and quit channels are passed as parameters is because
	// RemoteAgent.Start() constantly recreates these channels, so the loop code cannot
	// assume data stability in these member fields.
	func (a *RemoteAgent) loop(workCh chan *Work, quitCh chan struct{}) {
		ticker := time.NewTicker(5 * time.Second)
		defer ticker.Stop()
	
		for {
			select {
			case <-quitCh:
				return
			case work := <-workCh:
				a.mu.Lock()
				a.currentWork = work
				a.mu.Unlock()
			case <-ticker.C:
				// cleanup
				a.mu.Lock()
				for hash, work := range a.work {
					if time.Since(work.createdAt) > 7*(12*time.Second) {
						delete(a.work, hash)
					}
				}
				a.mu.Unlock()
	
				a.hashrateMu.Lock()
				for id, hashrate := range a.hashrate {
					if time.Since(hashrate.ping) > 10*time.Second {
						delete(a.hashrate, id)
					}
				}
				a.hashrateMu.Unlock()
			}
		}
	}

GetWork，这个方法由远程矿工调用，获取当前的挖矿任务。
	
	func (a *RemoteAgent) GetWork() ([3]string, error) {
		a.mu.Lock()
		defer a.mu.Unlock()
	
		var res [3]string
	
		if a.currentWork != nil {
			block := a.currentWork.Block
	
			res[0] = block.HashNoNonce().Hex()
			seedHash := ethash.SeedHash(block.NumberU64())
			res[1] = common.BytesToHash(seedHash).Hex()
			// Calculate the "target" to be returned to the external miner
			n := big.NewInt(1)
			n.Lsh(n, 255)
			n.Div(n, block.Difficulty())
			n.Lsh(n, 1)
			res[2] = common.BytesToHash(n.Bytes()).Hex()
	
			a.work[block.HashNoNonce()] = a.currentWork
			return res, nil
		}
		return res, errors.New("No work available yet, don't panic.")
	}

SubmitWork, 远程矿工会调用这个方法提交挖矿的结果。 对结果进行验证之后提交到returnCh

	// SubmitWork tries to inject a pow solution into the remote agent, returning
	// whether the solution was accepted or not (not can be both a bad pow as well as
	// any other error, like no work pending).
	func (a *RemoteAgent) SubmitWork(nonce types.BlockNonce, mixDigest, hash common.Hash) bool {
		a.mu.Lock()
		defer a.mu.Unlock()
	
		// Make sure the work submitted is present
		work := a.work[hash]
		if work == nil {
			log.Info("Work submitted but none pending", "hash", hash)
			return false
		}
		// Make sure the Engine solutions is indeed valid
		result := work.Block.Header()
		result.Nonce = nonce
		result.MixDigest = mixDigest
	
		if err := a.engine.VerifySeal(a.chain, result); err != nil {
			log.Warn("Invalid proof-of-work submitted", "hash", hash, "err", err)
			return false
		}
		block := work.Block.WithSeal(result)
	
		// Solutions seems to be valid, return to the miner and notify acceptance
		a.returnCh <- &Result{work, block}
		delete(a.work, hash)
	
		return true
	}

SubmitHashrate, 提交hash算力

	func (a *RemoteAgent) SubmitHashrate(id common.Hash, rate uint64) {
		a.hashrateMu.Lock()
		defer a.hashrateMu.Unlock()
	
		a.hashrate[id] = hashrate{time.Now(), rate}
	}


## unconfirmed.go

unconfirmed是一个数据结构，用来跟踪用户本地的挖矿信息的，比如挖出了一个块，那么等待足够的后续区块确认之后(5个)，再查看本地挖矿的区块是否包含在规范的区块链内部。

数据结构
	
	// headerRetriever is used by the unconfirmed block set to verify whether a previously
	// mined block is part of the canonical chain or not.
	// headerRetriever由未确认的块组使用，以验证先前挖掘的块是否是规范链的一部分。
	type headerRetriever interface {
		// GetHeaderByNumber retrieves the canonical header associated with a block number.
		GetHeaderByNumber(number uint64) *types.Header
	}
	
	// unconfirmedBlock is a small collection of metadata about a locally mined block
	// that is placed into a unconfirmed set for canonical chain inclusion tracking.
	// unconfirmedBlock 是本地挖掘区块的一个小的元数据的集合，用来放入未确认的集合用来追踪本地挖掘的区块是否被包含进入规范的区块链
	type unconfirmedBlock struct {
		index uint64
		hash  common.Hash
	}
	
	// unconfirmedBlocks implements a data structure to maintain locally mined blocks
	// have have not yet reached enough maturity to guarantee chain inclusion. It is
	// used by the miner to provide logs to the user when a previously mined block
	// has a high enough guarantee to not be reorged out of te canonical chain.	
	// unconfirmedBlocks 实现了一个数据结构，用来管理本地挖掘的区块，这些区块还没有达到足够的信任度来证明他们已经被规范的区块链接受。 它用来给矿工提供信息，以便他们了解他们之前挖到的区块是否被包含进入了规范的区块链。
	type unconfirmedBlocks struct {
		chain  headerRetriever // Blockchain to verify canonical status through 需要验证的区块链 用这个接口来获取当前的规范的区块头信息
		depth  uint            // Depth after which to discard previous blocks 经过多少个区块之后丢弃之前的区块
		blocks *ring.Ring      // Block infos to allow canonical chain cross checks // 区块信息，以允许规范链交叉检查
		lock   sync.RWMutex    // Protects the fields from concurrent access
	}

	// newUnconfirmedBlocks returns new data structure to track currently unconfirmed blocks.
	func newUnconfirmedBlocks(chain headerRetriever, depth uint) *unconfirmedBlocks {
		return &unconfirmedBlocks{
			chain: chain,
			depth: depth,
		}
	}

插入跟踪区块, 当矿工挖到一个区块的时候调用， index是区块的高度， hash是区块的hash值。
	
	
	// Insert adds a new block to the set of unconfirmed ones.
	func (set *unconfirmedBlocks) Insert(index uint64, hash common.Hash) {
		// If a new block was mined locally, shift out any old enough blocks
		// 如果一个本地的区块挖到了，那么移出已经超过depth的区块
		set.Shift(index)
	
		// Create the new item as its own ring
		// 循环队列的操作。
		item := ring.New(1)
		item.Value = &unconfirmedBlock{
			index: index,
			hash:  hash,
		}
		// Set as the initial ring or append to the end
		set.lock.Lock()
		defer set.lock.Unlock()
	
		if set.blocks == nil {
			set.blocks = item
		} else {
			// 移动到循环队列的最后一个元素插入item
			set.blocks.Move(-1).Link(item)
		}
		// Display a log for the user to notify of a new mined block unconfirmed
		log.Info("🔨 mined potential block", "number", index, "hash", hash)
	}
Shift方法会删除那些index超过传入的index-depth的区块，并检查他们是否在规范的区块链中。
	
	// Shift drops all unconfirmed blocks from the set which exceed the unconfirmed sets depth
	// allowance, checking them against the canonical chain for inclusion or staleness
	// report.
	func (set *unconfirmedBlocks) Shift(height uint64) {
		set.lock.Lock()
		defer set.lock.Unlock()
	
		for set.blocks != nil {
			// Retrieve the next unconfirmed block and abort if too fresh
			// 因为blocks中的区块都是按顺序排列的。排在最开始的肯定是最老的区块。
			// 所以每次只需要检查最开始的那个区块，如果处理完了，就从循环队列里面摘除。
			next := set.blocks.Value.(*unconfirmedBlock)
			if next.index+uint64(set.depth) > height { // 如果足够老了。
				break
			}
			// Block seems to exceed depth allowance, check for canonical status
			// 查询 那个区块高度的区块头
			header := set.chain.GetHeaderByNumber(next.index)
			switch {
			case header == nil:
				log.Warn("Failed to retrieve header of mined block", "number", next.index, "hash", next.hash)
			case header.Hash() == next.hash: // 如果区块头就等于我们自己，
				log.Info("🔗 block reached canonical chain", "number", next.index, "hash", next.hash)
			default: // 否则说明我们在侧链上面。
				log.Info("⑂ block  became a side fork", "number", next.index, "hash", next.hash)
			}
			// Drop the block out of the ring
			// 从循环队列删除
			if set.blocks.Value == set.blocks.Next().Value {
				// 如果当前的值就等于我们自己，说明只有循环队列只有一个元素，那么设置未nil
				set.blocks = nil
			} else {
				// 否则移动到最后，然后删除一个，再移动到最前。
				set.blocks = set.blocks.Move(-1)
				set.blocks.Unlink(1)
				set.blocks = set.blocks.Move(1)
			}
		}
	}

## worker.go


environment is the worker's current environment and holds all of the current state information.
<pre>type environment struct {
	signer types.Signer

	state     *state.StateDB // apply state changes here
	ancestors mapset.Set     // ancestor set (used for checking uncle parent validity)
	family    mapset.Set     // family set (used for checking uncle invalidity)
	uncles    mapset.Set     // uncle set
	tcount    int            // tx count in cycle
	gasPool   *core.GasPool  // available gas used to pack transactions

	header   *types.Header
	txs      []*types.Transaction
	receipts []*types.Receipt
}</pre>
task contains all information for consensus engine sealing and result submitting.
<pre>type task struct {
	receipts  []*types.Receipt
	state     *state.StateDB
	block     *types.Block
	createdAt time.Time
}</pre>
worker is the main object which takes care of applying messages to the new state and gathering the sealing result.
	
	// 工作者是负责将消息应用到新状态的主要对象
	type worker struct {
		config *params.ChainConfig
		engine consensus.Engine
		mu sync.Mutex
		// update loop
		mux          *event.TypeMux
		txCh         chan core.TxPreEvent		// 用来接受txPool里面的交易的通道
		txSub        event.Subscription			// 用来接受txPool里面的交易的订阅器
		chainHeadCh  chan core.ChainHeadEvent	// 用来接受区块头的通道
		chainHeadSub event.Subscription
		chainSideCh  chan core.ChainSideEvent	// 用来接受一个区块链从规范区块链移出的通道
		chainSideSub event.Subscription
		wg           sync.WaitGroup
	
		agents map[Agent]struct{}				// 所有的agent
		recv   chan *Result						// agent会把结果发送到这个通道
	
		eth     Backend							// eth的协议
		chain   *core.BlockChain				// 区块链
		proc    core.Validator					// 区块链验证器
		chainDb ethdb.Database					// 区块链数据库
	
		coinbase common.Address					// 挖矿者的地址
		extra    []byte							// 
		
		snapshotMu    sync.RWMutex				// 快照 RWMutex（快照读写锁）
		snapshotBlock *types.Block				// 快照 Block
		snapshotState *state.StateDB				// 快照 StateDB
		
		currentMu sync.Mutex
		current   *Work
	
		uncleMu        sync.Mutex
		possibleUncles map[common.Hash]*types.Block	//可能的叔父节点
	
		unconfirmed *unconfirmedBlocks // set of locally mined blocks pending canonicalness confirmations
	
		// atomic status counters
		mining int32
		atWork int32
	}

构造newworker
<pre>func newWorker(config *params.ChainConfig, engine consensus.Engine, coinbase common.Address, eth Backend, mux *event.TypeMux) *worker {
		worker := &worker{
			config:         config,
			engine:         engine,
			eth:            eth,
			mux:            mux,
			txCh:           make(chan core.TxPreEvent, txChanSize), // 4096
			chainHeadCh:    make(chan core.ChainHeadEvent, chainHeadChanSize), // 10
			chainSideCh:    make(chan core.ChainSideEvent, chainSideChanSize), // 10
			chainDb:        eth.ChainDb(),
			recv:           make(chan *Result, resultQueueSize), // 10
			chain:          eth.BlockChain(),
			proc:           eth.BlockChain().Validator(),
			possibleUncles: make(map[common.Hash]*types.Block),
			coinbase:       coinbase,
			agents:         make(map[Agent]struct{}),
			unconfirmed:    newUnconfirmedBlocks(eth.BlockChain(), miningLogAtDepth),
		}
		// Subscribe TxPreEvent for tx pool
		worker.txSub = eth.TxPool().SubscribeTxPreEvent(worker.txCh)
		// Subscribe events for blockchain
		worker.chainHeadSub = eth.BlockChain().SubscribeChainHeadEvent(worker.chainHeadCh)
		worker.chainSideSub = eth.BlockChain().SubscribeChainSideEvent(worker.chainSideCh)
		go worker.update()
	
		go worker.wait()
		worker.commitNewWork()
	
		return worker
	}</pre>

worker.mainloop (以前版本是worker.update) 会监听 ChainHeadEvent，ChainSideEvent，TxPreEvent 3个事件。

通过 chainHeadCh, chainSideCh, txCh 这3个 channel 来实现。ChainHeadEvent 事件指的是区块链中已经加入一个新的区块作为链头，这时候 worker 会开始挖掘下一个区块(在代码库中搜索 ChainHeadEvent，可以在 blockchain.go 中的 L1191 看到该事件是怎么触发的)。ChainSideEvent 指区块链中加入了一个新区块作为当前链头的分支，woker 会把这个区块放在 possibleUncles 数组，作为下一个挖掘区块可能的 Uncle 之一。当一个新的交易 tx 被加入 TxPool 中，会触发 TxPreEvent，如果这时 worker 没有在挖矿，那么开始执行，并把 tx 加入到 Work.txs 数组中，下次挖掘新区块可以使用
	
	func (w *worker) mainLoop() {
	defer w.txsSub.Unsubscribe()
	defer w.chainHeadSub.Unsubscribe()
	defer w.chainSideSub.Unsubscribe()

	for {
		select {
		case req := <-w.newWorkCh:
			w.commitNewWork(req.interrupt, req.noempty, req.timestamp)

		case ev := <-w.chainSideCh:
			// Short circuit for duplicate side blocks
			if _, exist := w.localUncles[ev.Block.Hash()]; exist {
				continue
			}
			if _, exist := w.remoteUncles[ev.Block.Hash()]; exist {
				continue
			}
			// Add side block to possible uncle block set depending on the author.
			if w.isLocalBlock != nil && w.isLocalBlock(ev.Block) {
				w.localUncles[ev.Block.Hash()] = ev.Block
			} else {
				w.remoteUncles[ev.Block.Hash()] = ev.Block
			}
			// If our mining block contains less than 2 uncle blocks,
			// add the new uncle block if valid and regenerate a mining block.
			if w.isRunning() && w.current != nil && w.current.uncles.Cardinality() < 2 {
				start := time.Now()
				if err := w.commitUncle(w.current, ev.Block.Header()); err == nil {
					var uncles []*types.Header
					w.current.uncles.Each(func(item interface{}) bool {
						hash, ok := item.(common.Hash)
						if !ok {
							return false
						}
						uncle, exist := w.localUncles[hash]
						if !exist {
							uncle, exist = w.remoteUncles[hash]
						}
						if !exist {
							return false
						}
						uncles = append(uncles, uncle.Header())
						return false
					})
					w.commit(uncles, nil, true, start)
				}
			}

		case ev := <-w.txsCh:
			// Apply transactions to the pending state if we're not mining.
			//
			// Note all transactions received may not be continuous with transactions
			// already included in the current mining block. These transactions will
			// be automatically eliminated.
			if !w.isRunning() && w.current != nil {
				// If block is already full, abort
				if gp := w.current.gasPool; gp != nil && gp.Gas() < params.TxGas {
					continue
				}
				w.mu.RLock()
				coinbase := w.coinbase
				w.mu.RUnlock()

				txs := make(map[common.Address]types.Transactions)
				for _, tx := range ev.Txs {
					acc, _ := types.Sender(w.current.signer, tx)
					txs[acc] = append(txs[acc], tx)
				}
				txset := types.NewTransactionsByPriceAndNonce(w.current.signer, txs)
				tcount := w.current.tcount
				w.commitTransactions(txset, coinbase, nil)
				// Only update the snapshot if any new transactons were added
				// to the pending block
				if tcount != w.current.tcount {
					w.updateSnapshot()
				}
			} else {
				// If clique is running in dev mode(period is 0), disable
				// advance sealing here.
				if w.chainConfig.Clique != nil && w.chainConfig.Clique.Period == 0 {
					w.commitNewWork(nil, true, time.Now().Unix())
				}
			}
			atomic.AddInt32(&w.newTxs, int32(len(ev.Txs)))

		// System stopped
		case <-w.exitCh:
			return
		case <-w.txsSub.Err():
			return
		case <-w.chainHeadSub.Err():
			return
		case <-w.chainSideSub.Err():
			return
		}
	}
}

worker.wait 执行挖完一个区块后的操作，通过 Result 这个 chan 实现，agent 完成挖矿后，从 chan 中获取 Block 和 Work 对象，Block 会被写到数据库中，加入本地的区块链，成为新的链头。完成这个操作后，会发送一条 NewMinedBlockEvent 事件，其他节点会决定是否接受这个新区块成为区块链新的链头。
<pre><code>func (self *worker) wait() {
	for {
		mustCommitNewWork := true
		for result := range self.recv {
			atomic.AddInt32(&self.atWork, -1)
			if result == nil {
				continue
			}
			block := result.Block
			work := result.Work
			for _, r := range work.receipts {
				for _, l := range r.Logs {
					l.BlockHash = block.Hash()
				}
			}
			for _, log := range work.state.Logs() {
				log.BlockHash = block.Hash()
			}
			stat, err := self.chain.WriteBlockWithState(block, work.receipts, work.state)
			if err != nil {
				log.Error("Failed writing block to chain", "err", err)
				continue
			}
			if stat == core.CanonStatTy {
				mustCommitNewWork = false
			}
			log.Error("I got new block")
			self.mux.Post(core.NewMinedBlockEvent{Block: block})
			var (
				events []interface{}
				logs   = work.state.Logs()
			)
			events = append(events, core.ChainEvent{Block: block, Hash: block.Hash(), Logs: logs})
			if stat == core.CanonStatTy {
				events = append(events, core.ChainHeadEvent{Block: block})
			}
			self.chain.PostChainEvents(events, logs)
			self.unconfirmed.(block.NumberU64(), block.Hash())
			if mustCommitNewWork {
				self.commitNewWork()
			}
		}
	}
}</code></pre>

commitNewWork 的作用是完成待挖掘区块的组装，最后通过 func (self *worker) push(p *Package) 让 agent 开始工作。具体来说，首先获取以系统当前时间作为新区块的时间，但要确保父区块的时间要早于新区块时间，否则进行 sleep 操作；接着构造区块头，确定父区块哈希值，当前区块编号，Gas 消耗数，附加数据，时间等，区块头的其他属性会在公式算法中确定；然后调用 engine.Prepare，准备好 Header 对象；处理 DAO 硬分叉的情况，增加附加数据；再接下来会从交易池里获取交易，加入到新区块的交易列表中，从 possibleUncles 获取叔区块；最后调用一致性引擎的 Finalize() 方法，给区块头增加 Root, TxHash, ReceiptHash 等属性，将创建的 Package 通过 channel 发送给 agent，进行挖矿操作。

	
	func (self *worker) commitNewWork() {
		self.mu.Lock()
		defer self.mu.Unlock()
		self.uncleMu.Lock()
		defer self.uncleMu.Unlock()
		self.currentMu.Lock()
		defer self.currentMu.Unlock()
	
		tstart := time.Now()
		parent := self.chain.CurrentBlock()
	
		tstamp := tstart.Unix()
		if parent.Time().Cmp(new(big.Int).SetInt64(tstamp)) >= 0 { // 不能出现比parent的时间还少的情况
			tstamp = parent.Time().Int64() + 1
		}
		// this will ensure we're not going off too far in the future
		// 我们的时间不要超过现在的时间太远， 那么等待一段时间， 
		// 感觉这个功能完全是为了测试实现的， 如果是真实的挖矿程序，应该不会等待。
		if now := time.Now().Unix(); tstamp > now+1 {
			wait := time.Duration(tstamp-now) * time.Second
			log.Info("Mining too far in the future", "wait", common.PrettyDuration(wait))
			time.Sleep(wait)
		}
	
		num := parent.Number()
		header := &types.Header{
			ParentHash: parent.Hash(),
			Number:     num.Add(num, common.Big1),
			GasLimit:   core.CalcGasLimit(parent),
			GasUsed:    new(big.Int),
			Extra:      self.extra,
			Time:       big.NewInt(tstamp),
		}
		// Only set the coinbase if we are mining (avoid spurious block rewards)
		// 只有当我们挖矿的时候才设置coinbase(避免虚假的块奖励？ TODO 没懂)
		if atomic.LoadInt32(&self.mining) == 1 {
			header.Coinbase = self.coinbase
		}
		if err := self.engine.Prepare(self.chain, header); err != nil {
			log.Error("Failed to prepare header for mining", "err", err)
			return
		}
		// If we are care about TheDAO hard-fork check whether to override the extra-data or not
		// 根据我们是否关心DAO硬分叉来决定是否覆盖额外的数据。
		if daoBlock := self.config.DAOForkBlock; daoBlock != nil {
			// Check whether the block is among the fork extra-override range
			// 检查区块是否在 DAO硬分叉的范围内   [daoblock,daoblock+limit]
			limit := new(big.Int).Add(daoBlock, params.DAOForkExtraRange)
			if header.Number.Cmp(daoBlock) >= 0 && header.Number.Cmp(limit) < 0 {
				// Depending whether we support or oppose the fork, override differently
				if self.config.DAOForkSupport { // 如果我们支持DAO 那么设置保留的额外的数据
					header.Extra = common.CopyBytes(params.DAOForkBlockExtra)
				} else if bytes.Equal(header.Extra, params.DAOForkBlockExtra) {
					header.Extra = []byte{} // If miner opposes, don't let it use the reserved extra-data // 否则不使用保留的额外数据
				}
			}
		}
		// Could potentially happen if starting to mine in an odd state.
		err := self.makeCurrent(parent, header) // 用新的区块头来设置当前的状态
		if err != nil {
			log.Error("Failed to create mining context", "err", err)
			return
		}
		// Create the current work task and check any fork transitions needed
		work := self.current
		if self.config.DAOForkSupport && self.config.DAOForkBlock != nil && self.config.DAOForkBlock.Cmp(header.Number) == 0 {
			misc.ApplyDAOHardFork(work.state)  // 把DAO里面的资金转移到指定的账户。
		}
		pending, err := self.eth.TxPool().Pending() //得到阻塞的资金
		if err != nil {
			log.Error("Failed to fetch pending transactions", "err", err)
			return
		}
		// 创建交易。 这个方法后续介绍
		txs := types.NewTransactionsByPriceAndNonce(self.current.signer, pending)
		// 提交交易 这个方法后续介绍	
		work.commitTransactions(self.mux, txs, self.chain, self.coinbase)
	
		// compute uncles for the new block.
		var (
			uncles    []*types.Header
			badUncles []common.Hash
		)
		for hash, uncle := range self.possibleUncles {
			if len(uncles) == 2 {
				break
			}
			if err := self.commitUncle(work, uncle.Header()); err != nil {
				log.Trace("Bad uncle found and will be removed", "hash", hash)
				log.Trace(fmt.Sprint(uncle))
	
				badUncles = append(badUncles, hash)
			} else {
				log.Debug("Committing new uncle to block", "hash", hash)
				uncles = append(uncles, uncle.Header())
			}
		}
		for _, hash := range badUncles {
			delete(self.possibleUncles, hash)
		}
		// Create the new block to seal with the consensus engine
		// 使用给定的状态来创建新的区块，Finalize会进行区块奖励等操作
		if work.Block, err = self.engine.Finalize(self.chain, header, work.state, work.txs, uncles, work.receipts); err != nil {
			log.Error("Failed to finalize block for sealing", "err", err)
			return
		}
		// We only care about logging if we're actually mining.
		// 
		if atomic.LoadInt32(&self.mining) == 1 {
			log.Info("Commit new mining work", "number", work.Block.Number(), "txs", work.tcount, "uncles", len(uncles), "elapsed", common.PrettyDuration(time.Since(tstart)))
			self.unconfirmed.Shift(work.Block.NumberU64() - 1)
		}
		self.push(work)
	}

push方法，如果我们没有在挖矿，那么直接返回，否则把任务送给每一个agent
	
	// push sends a new work task to currently live miner agents.
	func (self *worker) push(work *Work) {
		if atomic.LoadInt32(&self.mining) != 1 {
			return
		}
		for agent := range self.agents {
			atomic.AddInt32(&self.atWork, 1)
			if ch := agent.Work(); ch != nil {
				ch <- work
			}
		}
	}

makeCurrent，为当前的周期创建一个新的环境。
	
	// makeCurrent creates a new environment for the current cycle.
	// 
	func (self *worker) makeCurrent(parent *types.Block, header *types.Header) error {
		state, err := self.chain.StateAt(parent.Root())
		if err != nil {
			return err
		}
		work := &Work{
			config:    self.config,
			signer:    types.NewEIP155Signer(self.config.ChainId),
			state:     state,
			ancestors: set.New(),
			family:    set.New(),
			uncles:    set.New(),
			header:    header,
			createdAt: time.Now(),
		}
	
		// when 08 is processed ancestors contain 07 (quick block)
		for _, ancestor := range self.chain.GetBlocksFromHash(parent.Hash(), 7) {
			for _, uncle := range ancestor.Uncles() {
				work.family.Add(uncle.Hash())
			}
			work.family.Add(ancestor.Hash())
			work.ancestors.Add(ancestor.Hash())
		}
	
		// Keep track of transactions which return errors so they can be removed
		work.tcount = 0
		self.current = work
		return nil
	}

commitTransactions
	
	func (env *Work) commitTransactions(mux *event.TypeMux, txs *types.TransactionsByPriceAndNonce, bc *core.BlockChain, coinbase common.Address) {
		// 由于是打包新的区块中交易，所以将总 gasPool 初始化为 env.header.GasLimit
		if env.gasPool == nil {
			env.gasPool = new(core.GasPool).AddGas(env.header.GasLimit)
		}
	
		var coalescedLogs []*types.Log
	
		for {
			// If we don't have enough gas for any further transactions then we're done
			// 如果当前区块中所有 Gas 消耗已经使用完，则退出打包交易
			if env.gasPool.Gas() < params.TxGas {
				log.Trace("Not enough gas for further transactions", "have", env.gasPool, "want", params.TxGas)
				break
			}
					
			// Retrieve the next transaction and abort if all done
			// 检索下一笔交易，如果交易集合为空则退出 commit
			tx := txs.Peek()
			if tx == nil {
				break
			}
			// Error may be ignored here. The error has already been checked
			// during transaction acceptance is the transaction pool.
			//
			// We use the eip155 signer regardless of the current hf.
			from, _ := types.Sender(env.signer, tx)
			// Check whether the tx is replay protected. If we're not in the EIP155 hf
			// phase, start ignoring the sender until we do.
			// 请参考 https://github.com/ethereum/EIPs/blob/master/EIPS/eip-155.md
			// DAO事件发生后，以太坊分裂为ETH和ETC,因为两个链上的东西一摸一样，所以在ETC
			// 上面发生的交易可以拿到ETH上面进行重放， 反之亦然。 所以Vitalik提出了EIP155来避免这种情况。
			if tx.Protected() && !env.config.IsEIP155(env.header.Number) {
				log.Trace("Ignoring reply protected transaction", "hash", tx.Hash(), "eip155", env.config.EIP155Block)
	
				txs.Pop()
				continue
			}
			// Start executing the transaction
			env.state.Prepare(tx.Hash(), common.Hash{}, env.tcount)
			// 执行交易
			err, logs := env.commitTransaction(tx, bc, coinbase, gp)
			switch err {
			case core.ErrGasLimitReached:
				// Pop the current out-of-gas transaction without shifting in the next from the account
				// 弹出整个账户的所有交易， 不处理用户的下一个交易。
				log.Trace("Gas limit exceeded for current block", "sender", from)
				txs.Pop()
	
			case core.ErrNonceTooLow:
				// New head notification data race between the transaction pool and miner, shift
				// 移动到用户的下一个交易
				log.Trace("Skipping transaction with low nonce", "sender", from, "nonce", tx.Nonce())
				txs.Shift()
	
			case core.ErrNonceTooHigh:
				// Reorg notification data race between the transaction pool and miner, skip account =
				// 跳过这个账户
				log.Trace("Skipping account with hight nonce", "sender", from, "nonce", tx.Nonce())
				txs.Pop()
	
			case nil:
				// Everything ok, collect the logs and shift in the next transaction from the same account
				coalescedLogs = append(coalescedLogs, logs...)
				env.tcount++
				txs.Shift()
	
			default:
				// Strange error, discard the transaction and get the next in line (note, the
				// nonce-too-high clause will prevent us from executing in vain).
				// 其他奇怪的错误，跳过这个交易。
				log.Debug("Transaction failed, account skipped", "hash", tx.Hash(), "err", err)
				txs.Shift()
			}
		}
	
		if len(coalescedLogs) > 0 || env.tcount > 0 {
			// make a copy, the state caches the logs and these logs get "upgraded" from pending to mined
			// logs by filling in the block hash when the block was mined by the local miner. This can
			// cause a race condition if a log was "upgraded" before the PendingLogsEvent is processed.
			// 因为需要把log发送出去，而这边在挖矿完成后需要对log进行修改，所以拷贝一份发送出去，避免争用。
			cpy := make([]*types.Log, len(coalescedLogs))
			for i, l := range coalescedLogs {
				cpy[i] = new(types.Log)
				*cpy[i] = *l
			}
			go func(logs []*types.Log, tcount int) {
				if len(logs) > 0 {
					mux.Post(core.PendingLogsEvent{Logs: logs})
				}
				if tcount > 0 {
					mux.Post(core.PendingStateEvent{})
				}
			}(cpy, env.tcount)
		}
	}

commitTransaction执行ApplyTransaction
	
	func (env *Work) commitTransaction(tx *types.Transaction, bc *core.BlockChain, coinbase common.Address, gp *core.GasPool) (error, []*types.Log) {
		snap := env.state.Snapshot()
	
		receipt, _, err := core.ApplyTransaction(env.config, bc, &coinbase, gp, env.state, env.header, tx, env.header.GasUsed, vm.Config{})
		if err != nil {
			env.state.RevertToSnapshot(snap)
			return err, nil
		}
		env.txs = append(env.txs, tx)
		env.receipts = append(env.receipts, receipt)
	
		return nil, receipt.Logs
	}

wait函数用来接受挖矿的结果然后写入本地区块链，同时通过eth协议广播出去。
	
	func (self *worker) wait() {
		for {
			mustCommitNewWork := true
			for result := range self.recv {
				atomic.AddInt32(&self.atWork, -1)
	
				if result == nil {
					continue
				}
				block := result.Block
				work := result.Work
	
				// Update the block hash in all logs since it is now available and not when the
				// receipt/log of individual transactions were created.
				for _, r := range work.receipts {
					for _, l := range r.Logs {
						l.BlockHash = block.Hash()
					}
				}
				for _, log := range work.state.Logs() {
					log.BlockHash = block.Hash()
				}
				stat, err := self.chain.WriteBlockAndState(block, work.receipts, work.state)
				if err != nil {
					log.Error("Failed writing block to chain", "err", err)
					continue
				}
				// check if canon block and write transactions
				if stat == core.CanonStatTy { // 说明已经插入到规范的区块链
					// implicit by posting ChainHeadEvent
					// 因为这种状态下，会发送ChainHeadEvent，会触发上面的update里面的代码，这部分代码会commitNewWork，所以在这里就不需要commit了。
					mustCommitNewWork = false
				}	
				// Broadcast the block and announce chain insertion event
				// 广播区块，并且申明区块链插入事件。
				self.mux.Post(core.NewMinedBlockEvent{Block: block})
				var (
					events []interface{}
					logs   = work.state.Logs()
				)
				events = append(events, core.ChainEvent{Block: block, Hash: block.Hash(), Logs: logs})
				if stat == core.CanonStatTy {
					events = append(events, core.ChainHeadEvent{Block: block})
				}
				self.chain.PostChainEvents(events, logs)
	
				// Insert the block into the set of pending ones to wait for confirmations
				// 插入本地跟踪列表， 查看后续的确认状态。
				self.unconfirmed.Insert(block.NumberU64(), block.Hash())
	
				if mustCommitNewWork { // TODO ? 
					self.commitNewWork()
				}
			}
		}
	}


## miner.go
miner用来对worker进行管理， 订阅外部事件，控制worker的启动和停止。

数据结构
	
Backend wraps all methods required for mining.
<pre>type Backend interface {
	BlockChain() *core.BlockChain
	TxPool() *core.TxPool
}</pre>
Config is the configuration parameters of mining.
<pre>type Config struct {
	Etherbase common.Address `toml:",omitempty"` // Public address for block mining rewards (default = first account)
	Notify    []string       `toml:",omitempty"` // HTTP URL list to be notified of new work packages(only useful in ethash).
	ExtraData hexutil.Bytes  `toml:",omitempty"` // Block extra data set by the miner
	GasFloor  uint64         // Target gas floor for mined blocks.
	GasCeil   uint64         // Target gas ceiling for mined blocks.
	GasPrice  *big.Int       // Minimum gas price for mining a transaction
	Recommit  time.Duration  // The time interval for miner to re-create mining work.
	Noverify  bool           // Disable remote mining solution verification(only useful in ethash).
}</pre>
Miner creates blocks and searches for proof-of-work values.
<pre>type Miner struct {
	mux      *event.TypeMux
	worker   *worker
	coinbase common.Address
	eth      Backend
	engine   consensus.Engine
	exitCh   chan struct{}

	canStart    int32 // can start indicates whether we can start the mining operation
	shouldStart int32 // should start indicates whether we should start after sync
}</pre>


构造, 创建了一个启动了miner的update goroutine

	func New(eth Backend, config *params.ChainConfig, mux *event.TypeMux, engine consensus.Engine) *Miner {
		miner := &Miner{
			eth:      eth,
			mux:      mux,
			engine:   engine,
			worker:   newWorker(config, engine, common.Address{}, eth, mux),
			canStart: 1,
		}
		miner.Register(NewCpuAgent(eth.BlockChain(), engine))
		go miner.update()
	
		return miner
	}

update订阅了downloader的事件， 注意这个goroutine是一个一次性的循环， 只要接收到一次downloader的downloader.DoneEvent或者 downloader.FailedEvent事件， 就会设置canStart为1. 并退出循环， 这是为了避免黑客恶意的 DOS攻击，让你不断的处于异常状态
	
	// update keeps track of the downloader events. Please be aware that this is a one shot type of update loop.
	// It's entered once and as soon as `Done` or `Failed` has been broadcasted the events are unregistered and
	// the loop is exited. This to prevent a major security vuln where external parties can DOS you with blocks
	// and halt your mining operation for as long as the DOS continues.
	func (self *Miner) update() {
		events := self.mux.Subscribe(downloader.StartEvent{}, downloader.DoneEvent{}, downloader.FailedEvent{})
	out:
		for ev := range events.Chan() {
			switch ev.Data.(type) {
			case downloader.StartEvent:
				atomic.StoreInt32(&self.canStart, 0)
				if self.Mining() {
					self.Stop()
					atomic.StoreInt32(&self.shouldStart, 1)
					log.Info("Mining aborted due to sync")
				}
			case downloader.DoneEvent, downloader.FailedEvent:
				shouldStart := atomic.LoadInt32(&self.shouldStart) == 1
	
				atomic.StoreInt32(&self.canStart, 1)
				atomic.StoreInt32(&self.shouldStart, 0)
				if shouldStart {
					self.Start(self.coinbase)
				}
				// unsubscribe. we're only interested in this event once
				events.Unsubscribe()
				// stop immediately and ignore all further pending events
				break out
			}
		}
	}

在 Miner struct 中有一个 worker 类型成员变量，它指向 worker 中的 Work struct，当我们需要开始挖矿时，我们通过 miner.Start() 开始（见 eth/backend.go 中的 StartMining，L358）。在设置好 coinbase 和等待网络同步完成后，继续调用 self.worker.start()。
	
	func (self *Miner) Start(coinbase common.Address) {
		atomic.StoreInt32(&self.shouldStart, 1)  // shouldStart 是是否应该启动
		self.worker.setEtherbase(coinbase)	     
		self.coinbase = coinbase
	
		if atomic.LoadInt32(&self.canStart) == 0 {  // canStart是否能够启动，
			log.Info("Network syncing, will start miner afterwards")
			return
		}
		atomic.StoreInt32(&self.mining, 1)
	
		log.Info("Starting mining operation")
		self.worker.start()  // 启动worker 开始挖矿
		self.worker.commitNewWork()  //提交新的挖矿任务。
	}

## stress_ethash.go
makeGenesis creates a custom Ethash genesis block based on some pre-defined faucet accounts.
<pre>func makeGenesis(faucets []*ecdsa.PrivateKey) *core.Genesis {
	genesis := core.DefaultTestnetGenesisBlock()
	genesis.Difficulty = params.MinimumDifficulty
	genesis.GasLimit = 25000000

	genesis.Config.ChainID = big.NewInt(18)
	genesis.Config.EIP150Hash = common.Hash{}

	genesis.Alloc = core.GenesisAlloc{}
	for _, faucet := range faucets {
		genesis.Alloc[crypto.PubkeyToAddress(faucet.PublicKey)] = core.GenesisAccount{
			Balance: new(big.Int).Exp(big.NewInt(2), big.NewInt(128), nil),
		}
	}
	return genesis
}</pre>
makeminer 主要是创建一个miner进行挖矿

func makeMiner(genesis *core.Genesis) (*node.Node, error)

主要的逻辑在main中，这就是挖矿的逻辑：

func main() {
	log.Root().SetHandler(log.LvlFilterHandler(log.LvlInfo, log.StreamHandler(os.Stderr, log.TerminalFormat(true))))
	fdlimit.Raise(2048)

	// Generate a batch of accounts to seal and fund with
	faucets := make([]*ecdsa.PrivateKey, 128)
	for i := 0; i < len(faucets); i++ {
		faucets[i], _ = crypto.GenerateKey()
	}
	// Pre-generate the ethash mining DAG so we don't race
	ethash.MakeDataset(1, filepath.Join(os.Getenv("HOME"), ".ethash"))

	// Create an Ethash network based off of the Ropsten config
	genesis := makeGenesis(faucets)

	var (
		nodes  []*node.Node
		enodes []*enode.Node
	)
	for i := 0; i < 4; i++ {
		// Start the node and wait until it's up
		node, err := makeMiner(genesis)
		if err != nil {
			panic(err)
		}
		defer node.Close()

		for node.Server().NodeInfo().Ports.Listener == 0 {
			time.Sleep(250 * time.Millisecond)
		}
		// Connect the node to al the previous ones
		for _, n := range enodes {
			node.Server().AddPeer(n)
		}
		// Start tracking the node and it's enode
		nodes = append(nodes, node)
		enodes = append(enodes, node.Server().Self())

		// Inject the signer key and start sealing with it
		store := node.AccountManager().Backends(keystore.KeyStoreType)[0].(*keystore.KeyStore)
		if _, err := store.NewAccount(""); err != nil {
			panic(err)
		}
	}
	// Iterate over all the nodes and start signing with them
	time.Sleep(3 * time.Second)

	for _, node := range nodes {
		var ethereum *eth.Ethereum
		if err := node.Service(&ethereum); err != nil {
			panic(err)
		}
		if err := ethereum.StartMining(1); err != nil {
			panic(err)
		}
	}
	time.Sleep(3 * time.Second)

	// Start injecting transactions from the faucets like crazy
	nonces := make([]uint64, len(faucets))
	for {
		index := rand.Intn(len(faucets))

		// Fetch the accessor for the relevant signer
		var ethereum *eth.Ethereum
		if err := nodes[index%len(nodes)].Service(&ethereum); err != nil {
			panic(err)
		}
		// Create a self transaction and inject into the pool
		tx, err := types.SignTx(types.NewTransaction(nonces[index], crypto.PubkeyToAddress(faucets[index].PublicKey), new(big.Int), 21000, big.NewInt(100000000000+rand.Int63n(65536)), nil), types.HomesteadSigner{}, faucets[index])
		if err != nil {
			panic(err)
		}
		if err := ethereum.TxPool().AddLocal(tx); err != nil {
			panic(err)
		}
		nonces[index]++

		// Wait if we're too saturated
		if pend, _ := ethereum.TxPool().Stats(); pend > 2048 {
			time.Sleep(100 * time.Millisecond)
		}
	}
}
