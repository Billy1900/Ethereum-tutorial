## contract.go
contract 代表了以太坊 state database里面的一个合约。包含了合约代码，调用参数。


结构
	
	// ContractRef is a reference to the contract's backing object
	type ContractRef interface {
		Address() common.Address
	}
	
	// AccountRef implements ContractRef.
	//
	// Account references are used during EVM initialisation and
	// it's primary use is to fetch addresses. Removing this object
	// proves difficult because of the cached jump destinations which
	// are fetched from the parent contract (i.e. the caller), which
	// is a ContractRef.
	type AccountRef common.Address
	
	// Address casts AccountRef to a Address
	func (ar AccountRef) Address() common.Address { return (common.Address)(ar) }
	
	// Contract represents an ethereum contract in the state database. It contains
	// the the contract code, calling arguments. Contract implements ContractRef
	type Contract struct {
		// CallerAddress is the result of the caller which initialised this
		// contract. However when the "call method" is delegated this value
		// needs to be initialised to that of the caller's caller.
		// CallerAddress是初始化这个合约的人。 如果是delegate，这个值被设置为调用者的调用者。
		CallerAddress common.Address
		caller        ContractRef
		self          ContractRef
	
		jumpdests destinations // result of JUMPDEST analysis.  JUMPDEST指令的分析
	
		Code     []byte  //代码
		CodeHash common.Hash  //代码的HASH
		CodeAddr *common.Address //代码地址
		Input    []byte     // 入参
	
		Gas   uint64  		// 合约还有多少Gas
		value *big.Int      
	
		Args []byte  //好像没有使用
	
		DelegateCall bool  
	}

构造
	
	// NewContract returns a new contract environment for the execution of EVM.
	func NewContract(caller ContractRef, object ContractRef, value *big.Int, gas uint64) *Contract {
		c := &Contract{CallerAddress: caller.Address(), caller: caller, self: object, Args: nil}
	
		if parent, ok := caller.(*Contract); ok {
			// Reuse JUMPDEST analysis from parent context if available.
			// 如果 caller 是一个合约，说明是合约调用了我们。 jumpdests设置为caller的jumpdests
			c.jumpdests = parent.jumpdests
		} else {
			c.jumpdests = make(destinations)
		}
	
		// Gas should be a pointer so it can safely be reduced through the run
		// This pointer will be off the state transition
		c.Gas = gas
		// ensures a value is set
		c.value = value
	
		return c
	}

AsDelegate将合约设置为委托调用并返回当前合同（用于链式调用）

	// AsDelegate sets the contract to be a delegate call and returns the current
	// contract (for chaining calls)
	func (c *Contract) AsDelegate() *Contract {
		c.DelegateCall = true
		// NOTE: caller must, at all times be a contract. It should never happen
		// that caller is something other than a Contract.
		parent := c.caller.(*Contract)
		c.CallerAddress = parent.CallerAddress
		c.value = parent.value
	
		return c
	}
		
GetOp  用来获取下一跳指令
	
	// GetOp returns the n'th element in the contract's byte array
	func (c *Contract) GetOp(n uint64) OpCode {
		return OpCode(c.GetByte(n))
	}
	
	// GetByte returns the n'th byte in the contract's byte array
	func (c *Contract) GetByte(n uint64) byte {
		if n < uint64(len(c.Code)) {
			return c.Code[n]
		}
	
		return 0
	}

	// Caller returns the caller of the contract.
	//
	// Caller will recursively call caller when the contract is a delegate
	// call, including that of caller's caller.
	func (c *Contract) Caller() common.Address {
		return c.CallerAddress
	}
UseGas使用Gas。 
	
	// UseGas attempts the use gas and subtracts it and returns true on success
	func (c *Contract) UseGas(gas uint64) (ok bool) {
		if c.Gas < gas {
			return false
		}
		c.Gas -= gas
		return true
	}
	
	// Address returns the contracts address
	func (c *Contract) Address() common.Address {
		return c.self.Address()
	}
	
	// Value returns the contracts value (sent to it from it's caller)
	func (c *Contract) Value() *big.Int {
		return c.value
	}
SetCode	，SetCallCode 设置代码。

	// SetCode sets the code to the contract
	func (self *Contract) SetCode(hash common.Hash, code []byte) {
		self.Code = code
		self.CodeHash = hash
	}
	
	// SetCallCode sets the code of the contract and address of the backing data
	// object
	func (self *Contract) SetCallCode(addr *common.Address, hash common.Hash, code []byte) {
		self.Code = code
		self.CodeHash = hash
		self.CodeAddr = addr
	}


## evm.go

结构


	// Context provides the EVM with auxiliary information. Once provided
	// it shouldn't be modified.
	// 上下文为EVM提供辅助信息。 一旦提供，不应该修改。
	type Context struct {
		// CanTransfer returns whether the account contains
		// sufficient ether to transfer the value
		// CanTransfer 函数返回账户是否有足够的ether用来转账
		CanTransfer CanTransferFunc
		// Transfer transfers ether from one account to the other
		// Transfer 用来从一个账户给另一个账户转账
		Transfer TransferFunc
		// GetHash returns the hash corresponding to n
		// GetHash用来返回入参n对应的hash值
		GetHash GetHashFunc
	
		// Message information
		// 用来提供Origin的信息 sender的地址
		Origin   common.Address // Provides information for ORIGIN
		// 用来提供GasPrice信息
		GasPrice *big.Int       // Provides information for GASPRICE
	
		// Block information
		Coinbase    common.Address // Provides information for COINBASE
		GasLimit    *big.Int       // Provides information for GASLIMIT
		BlockNumber *big.Int       // Provides information for NUMBER
		Time        *big.Int       // Provides information for TIME
		Difficulty  *big.Int       // Provides information for DIFFICULTY
	}
	
	// EVM is the Ethereum Virtual Machine base object and provides
	// the necessary tools to run a contract on the given state with
	// the provided context. It should be noted that any error
	// generated through any of the calls should be considered a
	// revert-state-and-consume-all-gas operation, no checks on
	// specific errors should ever be performed. The interpreter makes
	// sure that any errors generated are to be considered faulty code.
	// EVM是以太坊虚拟机基础对象，并提供必要的工具，以使用提供的上下文运行给定状态的合约。
	// 应该指出的是，任何调用产生的任何错误都应该被认为是一种回滚修改状态和消耗所有GAS操作，
	// 不应该执行对具体错误的检查。 解释器确保生成的任何错误都被认为是错误的代码。
	// The EVM should never be reused and is not thread safe.
	type EVM struct {
		// Context provides auxiliary blockchain related information
		Context
		// StateDB gives access to the underlying state
		StateDB StateDB
		// Depth is the current call stack
		// 当前的调用堆栈
		depth int
	
		// chainConfig contains information about the current chain
		// 包含了当前的区块链的信息
		chainConfig *params.ChainConfig
		// chain rules contains the chain rules for the current epoch
		chainRules params.Rules
		// virtual machine configuration options used to initialise the
		// evm.
		vmConfig Config
		// global (to this context) ethereum virtual machine
		// used throughout the execution of the tx.
		interpreter *Interpreter
		// abort is used to abort the EVM calling operations
		// NOTE: must be set atomically
		abort int32
	}
	
Context 给 EVM 提供运行合约的上下文信息，
- 其中 CanTransfer 是返回账户是否有足够余额的函数，
- Transfer 可以用来完成转账操作，GetHash 返回第 n 个区块的哈希值。
- EVM 结构体中稍值得一提的是 interpreter，它根据代码以及 jump_table 中对应的指令逐条执行合约.

构造函数
	
	// NewEVM retutrns a new EVM . The returned EVM is not thread safe and should
	// only ever be used *once*.
	func NewEVM(ctx Context, statedb StateDB, chainConfig *params.ChainConfig, vmConfig Config) *EVM {
		evm := &EVM{
			Context:     ctx,
			StateDB:     statedb,
			vmConfig:    vmConfig,
			chainConfig: chainConfig,
			chainRules:  chainConfig.Rules(ctx.BlockNumber),
		}
	
		evm.interpreter = NewInterpreter(evm, vmConfig)
		return evm
	}
	
	// Cancel cancels any running EVM operation. This may be called concurrently and
	// it's safe to be called multiple times.
	func (evm *EVM) Cancel() {
		atomic.StoreInt32(&evm.abort, 1)
	}


合约创建 Create 会创建一个新的合约。
- 首先会进行一系列验证，调用栈的深度不能超过 1024；调用的账户有足够多的余额；对调用者地址的 nonce+1，通过地址和 nonce 生成合约地址，通过合约地址获取合约哈希值，确保调用的地址不能已经存在合约。
- 接着利用 StateDB 创建一个快照，如果之后的调用出现问题可以回滚。在进行了这一系列初始化之后，发起一笔转账操作，发送方地址余额减 value 值，合约账户的余额加 value 值，接着通过调用 contract 的 SetCallCode ，根据发送方地址，合约地址，金额 value，gas，合约代码，代码哈希初始化合约对象，
- 然后调用 run(evm, contract, nil)执行合约的初始化代码，这个生成的代码有一定的长度限制，当合约创建成功，没有错误返回，则计算存储代码所需的 gas，如果没有足够 gas 则进行报错。
- 之后就是错误处理了，如果存在错误，需要回滚到之前创建的状态快照，没有错误则返回创建成功。可以看到，合约代码通过 state 模块的 SetCode，保存在账户中 codehash 指向的存储区域，这部分的代码都属于对世界状态的修改。
<pre>
	// Create creates a new contract using code as deployment code.
	func (evm *EVM) Create(caller ContractRef, code []byte, gas uint64, value *big.Int) (ret []byte, contractAddr common.Address, leftOverGas uint64, err error) {
		// Depth check execution. Fail if we're trying to execute above the
		// limit.
		if evm.depth > int(params.CallCreateDepth) {
			return nil, common.Address{}, gas, ErrDepth
		}
		if !evm.CanTransfer(evm.StateDB, caller.Address(), value) {
			return nil, common.Address{}, gas, ErrInsufficientBalance
		}
		// Ensure there's no existing contract already at the designated address
		// 确保特定的地址没有合约存在
		nonce := evm.StateDB.GetNonce(caller.Address())
		evm.StateDB.SetNonce(caller.Address(), nonce+1)
	
		contractAddr = crypto.CreateAddress(caller.Address(), nonce)
		contractHash := evm.StateDB.GetCodeHash(contractAddr)
		if evm.StateDB.GetNonce(contractAddr) != 0 || (contractHash != (common.Hash{}) && contractHash != emptyCodeHash) { //如果已经存在
			return nil, common.Address{}, 0, ErrContractAddressCollision
		}
		// Create a new account on the state
		snapshot := evm.StateDB.Snapshot()  //创建一个StateDB的快照，以便回滚
		evm.StateDB.CreateAccount(contractAddr) //创建账户
		if evm.ChainConfig().IsEIP158(evm.BlockNumber) {
			evm.StateDB.SetNonce(contractAddr, 1) //设置nonce
		}
		evm.Transfer(evm.StateDB, caller.Address(), contractAddr, value)  //转账
	
		// initialise a new contract and set the code that is to be used by the
		// E The contract is a scoped evmironment for this execution context
		// only.
		contract := NewContract(caller, AccountRef(contractAddr), value, gas)
		contract.SetCallCode(&contractAddr, crypto.Keccak256Hash(code), code)
	
		if evm.vmConfig.NoRecursion && evm.depth > 0 {
			return nil, contractAddr, gas, nil
		}
		ret, err = run(evm, snapshot, contract, nil) //执行合约的初始化代码
		// check whether the max code size has been exceeded
		// 检查初始化生成的代码的长度不超过限制
		maxCodeSizeExceeded := evm.ChainConfig().IsEIP158(evm.BlockNumber) && len(ret) > params.MaxCodeSize
		// if the contract creation ran successfully and no errors were returned
		// calculate the gas required to store the code. If the code could not
		// be stored due to not enough gas set an error and let it be handled
		// by the error checking condition below.
		//如果合同创建成功并且没有错误返回，则计算存储代码所需的GAS。 如果由于没有足够的GAS而导致代码不能被存储设置错误，并通过下面的错误检查条件来处理。
		if err == nil && !maxCodeSizeExceeded {
			createDataGas := uint64(len(ret)) * params.CreateDataGas
			if contract.UseGas(createDataGas) {
				evm.StateDB.SetCode(contractAddr, ret)
			} else {
				err = ErrCodeStoreOutOfGas
			}
		}
	
		// When an error was returned by the EVM or when setting the creation code
		// above we revert to the snapshot and consume any gas remaining. Additionally
		// when we're in homestead this also counts for code storage gas errors.
		// 当错误返回我们回滚修改，
		if maxCodeSizeExceeded || (err != nil && (evm.ChainConfig().IsHomestead(evm.BlockNumber) || err != ErrCodeStoreOutOfGas)) {
			evm.StateDB.RevertToSnapshot(snapshot)
			if err != errExecutionReverted {
				contract.UseGas(contract.Gas)
			}
		}
		// Assign err if contract code size exceeds the max while the err is still empty.
		if maxCodeSizeExceeded && err == nil {
			err = errMaxCodeSizeExceeded
		}
		return ret, contractAddr, contract.Gas, err
	}</pre>


Call方法, 无论我们转账或者是执行合约代码都会调用到这里， 同时合约里面的call指令也会执行到这里。
- 和 Create 方法类似，不过 Create 方法的资金转移发生在创建合约用户账户和合约账户之间，而 Call 方法的资金转移发生在合约发送方和合约接收方之间。
- Call 方法需要先检查合约调用深度；确保账户有足够余额；调用 Call 方法的可能是一个转账操作，也可能是一个运行合约的操作，所以接下来会通过 StateDB 查看指定的地址是否存在，如果不存在的话，接着查看该地址是否为内置合约，这些预编译的合约在 core/vm/constracts.go 中定义，主要是用于加密操作，如果本地确实没有合约接收方的账户，创建一个接收方的账户，更新本地的状态数据库。
- 接着 evm 会调用 Transfer 方法（即 Context 里的 TransferFunc）进行转账。最后，通过 StateDB 的 GetCode 拿到该地址对应的代码，通过 run(evm, contract, input) 运行合约，如果是单纯的转账，通过 GetCode 拿到的代码是空，自然也没有合约的运行。这一步完成后，就可以返回执行结果了。合约产生的 gas 总数会加入到矿工账户，作为矿工收入。

<pre>	
	// Call executes the contract associated with the addr with the given input as
	// parameters. It also handles any necessary value transfer required and takes
	// the necessary steps to create accounts and reverses the state in case of an
	// execution error or failed value transfer.
	
	// Call 执行与给定的input作为参数与addr相关联的合约。 
	// 它还处理所需的任何必要的转账操作，并采取必要的步骤来创建帐户
	// 并在任意错误的情况下回滚所做的操作。

	func (evm *EVM) Call(caller ContractRef, addr common.Address, input []byte, gas uint64, value *big.Int) (ret []byte, leftOverGas uint64, err error) {
		if evm.vmConfig.NoRecursion && evm.depth > 0 {
			return nil, gas, nil
		}
	
		// Fail if we're trying to execute above the call depth limit
		//  调用深度最多1024
		if evm.depth > int(params.CallCreateDepth) {
			return nil, gas, ErrDepth
		}
		// Fail if we're trying to transfer more than the available balance
		// 查看我们的账户是否有足够的金钱。
		if !evm.Context.CanTransfer(evm.StateDB, caller.Address(), value) {
			return nil, gas, ErrInsufficientBalance
		}
	
		var (
			to       = AccountRef(addr)
			snapshot = evm.StateDB.Snapshot()
		)
		if !evm.StateDB.Exist(addr) { // 查看指定地址是否存在
			// 如果地址不存在，查看是否是 native go的合约， native go的合约在
			// contracts.go 文件里面
			precompiles := PrecompiledContractsHomestead
			if evm.ChainConfig().IsByzantium(evm.BlockNumber) {
				precompiles = PrecompiledContractsByzantium
			}
			if precompiles[addr] == nil && evm.ChainConfig().IsEIP158(evm.BlockNumber) && value.Sign() == 0 {
				// 如果不是指定的合约地址， 并且value的值为0那么返回正常，而且这次调用没有消耗Gas
				return nil, gas, nil
			}
			// 负责在本地状态创建addr
			evm.StateDB.CreateAccount(addr)
		}
		// 执行转账
		evm.Transfer(evm.StateDB, caller.Address(), to.Address(), value)
	
		// initialise a new contract and set the code that is to be used by the
		// E The contract is a scoped environment for this execution context
		// only.
		contract := NewContract(caller, to, value, gas)
		contract.SetCallCode(&addr, evm.StateDB.GetCodeHash(addr), evm.StateDB.GetCode(addr))
	
		ret, err = run(evm, snapshot, contract, input)
		// When an error was returned by the EVM or when setting the creation code
		// above we revert to the snapshot and consume any gas remaining. Additionally
		// when we're in homestead this also counts for code storage gas errors.
		if err != nil {
			evm.StateDB.RevertToSnapshot(snapshot)
			if err != errExecutionReverted { 
				// 如果是由revert指令触发的错误，因为ICO一般设置了人数限制或者资金限制
				// 在大家抢购的时候很可能会触发这些限制条件，导致被抽走不少钱。这个时候
				// 又不能设置比较低的GasPrice和GasLimit。因为要速度快。
				// 那么不会使用剩下的全部Gas，而是只会使用代码执行的Gas
				// 不然会被抽走 GasLimit *GasPrice的钱，那可不少。
				contract.UseGas(contract.Gas)
			}
		}
		return ret, contract.Gas, err
	}
</pre>

剩下的三个函数 CallCode, DelegateCall, 和 StaticCall，这三个函数不能由外部调用，只能由Opcode触发。


CallCode

	// CallCode differs from Call in the sense that it executes the given address'
	// code with the caller as context.
	// CallCode与Call不同的地方在于它使用caller的context来执行给定地址的代码。
	
	func (evm *EVM) CallCode(caller ContractRef, addr common.Address, input []byte, gas uint64, value *big.Int) (ret []byte, leftOverGas uint64, err error) {
		if evm.vmConfig.NoRecursion && evm.depth > 0 {
			return nil, gas, nil
		}
	
		// Fail if we're trying to execute above the call depth limit
		if evm.depth > int(params.CallCreateDepth) {
			return nil, gas, ErrDepth
		}
		// Fail if we're trying to transfer more than the available balance
		if !evm.CanTransfer(evm.StateDB, caller.Address(), value) {
			return nil, gas, ErrInsufficientBalance
		}
	
		var (
			snapshot = evm.StateDB.Snapshot()
			to       = AccountRef(caller.Address())  //这里是最不同的地方 to的地址被修改为caller的地址了 而且没有转账的行为
		)
		// initialise a new contract and set the code that is to be used by the
		// E The contract is a scoped evmironment for this execution context
		// only.
		contract := NewContract(caller, to, value, gas)
		contract.SetCallCode(&addr, evm.StateDB.GetCodeHash(addr), evm.StateDB.GetCode(addr))
	
		ret, err = run(evm, snapshot, contract, input)
		if err != nil {
			evm.StateDB.RevertToSnapshot(snapshot)
			if err != errExecutionReverted {
				contract.UseGas(contract.Gas)
			}
		}
		return ret, contract.Gas, err
	}

DelegateCall

	// DelegateCall differs from CallCode in the sense that it executes the given address'
	// code with the caller as context and the caller is set to the caller of the caller.
	// DelegateCall 和 CallCode不同的地方在于 caller被设置为 caller的caller
	func (evm *EVM) DelegateCall(caller ContractRef, addr common.Address, input []byte, gas uint64) (ret []byte, leftOverGas uint64, err error) {
		if evm.vmConfig.NoRecursion && evm.depth > 0 {
			return nil, gas, nil
		}
		// Fail if we're trying to execute above the call depth limit
		if evm.depth > int(params.CallCreateDepth) {
			return nil, gas, ErrDepth
		}
	
		var (
			snapshot = evm.StateDB.Snapshot()
			to       = AccountRef(caller.Address()) 
		)
	
		// Initialise a new contract and make initialise the delegate values
		// 标识为AsDelete()
		contract := NewContract(caller, to, nil, gas).AsDelegate() 
		contract.SetCallCode(&addr, evm.StateDB.GetCodeHash(addr), evm.StateDB.GetCode(addr))
	
		ret, err = run(evm, snapshot, contract, input)
		if err != nil {
			evm.StateDB.RevertToSnapshot(snapshot)
			if err != errExecutionReverted {
				contract.UseGas(contract.Gas)
			}
		}
		return ret, contract.Gas, err
	}
	
	// StaticCall executes the contract associated with the addr with the given input
	// as parameters while disallowing any modifications to the state during the call.
	// Opcodes that attempt to perform such modifications will result in exceptions
	// instead of performing the modifications.
	// StaticCall不允许执行任何修改状态的操作，
	
	func (evm *EVM) StaticCall(caller ContractRef, addr common.Address, input []byte, gas uint64) (ret []byte, leftOverGas uint64, err error) {
		if evm.vmConfig.NoRecursion && evm.depth > 0 {
			return nil, gas, nil
		}
		// Fail if we're trying to execute above the call depth limit
		if evm.depth > int(params.CallCreateDepth) {
			return nil, gas, ErrDepth
		}
		// Make sure the readonly is only set if we aren't in readonly yet
		// this makes also sure that the readonly flag isn't removed for
		// child calls.
		if !evm.interpreter.readOnly {
			evm.interpreter.readOnly = true
			defer func() { evm.interpreter.readOnly = false }()
		}
	
		var (
			to       = AccountRef(addr)
			snapshot = evm.StateDB.Snapshot()
		)
		// Initialise a new contract and set the code that is to be used by the
		// EVM. The contract is a scoped environment for this execution context
		// only.
		contract := NewContract(caller, to, new(big.Int), gas)
		contract.SetCallCode(&addr, evm.StateDB.GetCodeHash(addr), evm.StateDB.GetCode(addr))
	
		// When an error was returned by the EVM or when setting the creation code
		// above we revert to the snapshot and consume any gas remaining. Additionally
		// when we're in Homestead this also counts for code storage gas errors.
		ret, err = run(evm, snapshot, contract, input)
		if err != nil {
			evm.StateDB.RevertToSnapshot(snapshot)
			if err != errExecutionReverted {
				contract.UseGas(contract.Gas)
			}
		}
		return ret, contract.Gas, err
	}

<pre><code>func (in *Interpreter) Run(contract *Contract, input []byte) (ret []byte, err error) {
	in.evm.depth++
	defer func() { in.evm.depth-- }()
	in.returnData = nil
	if len(contract.Code) == 0 {
		return nil, nil
	}
	var (
		op    OpCode
		mem   = NewMemory()
		stack = newstack()
		pc   = uint64(0)
		cost uint64
		pcCopy  uint64
		gasCopy uint64
		logged  bool
	)
	contract.Input = input
	if in.cfg.Debug {
		defer func() {
			if err != nil {
				if !logged {
					in.cfg.Tracer.CaptureState(in.evm, pcCopy, op, gasCopy, cost, mem, stack, contract, in.evm.depth, err)
				} else {
					in.cfg.Tracer.CaptureFault(in.evm, pcCopy, op, gasCopy, cost, mem, stack, contract, in.evm.depth, err)
				}
			}
		}()
	}
	for atomic.LoadInt32(&in.evm.abort) == 0 {
		if in.cfg.Debug {
			logged, pcCopy, gasCopy = false, pc, contract.Gas
		}
		
		op = contract.GetOp(pc)
		operation := in.cfg.JumpTable[op]
		if !operation.valid {
			return nil, fmt.Errorf("invalid opcode 0x%x", int(op))
		}
		if err := operation.validateStack(stack); err != nil {
			return nil, err
		}
		if err := in.enforceRestrictions(op, operation, stack); err != nil {
			return nil, err
		}
		var memorySize uint64
		if operation.memorySize != nil {
			memSize, overflow := bigUint64(operation.memorySize(stack))
			if overflow {
				return nil, errGasUintOverflow
			}
			if memorySize, overflow = math.SafeMul(toWordSize(memSize), 32); overflow {
				return nil, errGasUintOverflow
			}
		}
		cost, err = operation.gasCost(in.gasTable, in.evm, contract, stack, mem, memorySize)
		if err != nil || !contract.UseGas(cost) {
			return nil, ErrOutOfGas
		}
		if memorySize > 0 {
			mem.Resize(memorySize)
		}
		if in.cfg.Debug {
			in.cfg.Tracer.CaptureState(in.evm, pc, op, gasCopy, cost, mem, stack, contract, in.evm.depth, err)
			logged = true
		}
		res, err := operation.execute(&pc, in.evm, contract, mem, stack)
		if verifyPool {
			verifyIntegerPool(in.intPool)
		}
		if operation.returns {
			in.returnData = res
		}
		switch {
		case err != nil:
			return nil, err
		case operation.reverts:
			return res, errExecutionReverted
		case operation.halts:
			return res, nil
		case !operation.jumps:
			pc++
		}
	}
	return nil, nil
}</code></pre>
Run 方法会循环执行合约的代码，主要逻辑在 for 循环中，直到遇到 STOP，RETURN，SELFDESTRUCT 指令被执行，或者是遇到任意错误，或者说 done 标志被父 context 设置。这个循环才会结束。

大致的执行过程是首先通过 op = contract.GetOp(pc) 拿到下一个需要执行的指令，接着通过 operation := in.cfg.JumpTable[op] 拿到对应的 operation，operation 的类型在 core/vm/jump_table.go 中定义，其中包括指令对应的方法，计算 gas 的方法，验证栈溢出的方法，计算内存的方法等。进行一系列检查后再通过 operation.execute 执行指令。最后一个指令的执行结果会决定整个合约代码的执行结果。
