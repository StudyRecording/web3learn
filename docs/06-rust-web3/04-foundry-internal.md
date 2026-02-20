# Foundry 内部原理

> 深入理解 Rust 编写的 Web3 开发工具

## Foundry 架构概览

Foundry 是用 Rust 编写的以太坊开发工具链，其模块化架构非常值得学习。

```
┌─────────────────────────────────────────────────────────────┐
│                    Foundry 工具链                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  forge          编译、测试、部署合约                         │
│    │                                                        │
│    ├── forge-core (核心逻辑)                                │
│    ├── evm (EVM 实现)                                       │
│    └── utils (工具函数)                                     │
│                                                             │
│  cast           命令行 RPC 客户端                           │
│    │                                                        │
│    ├── alloy (以太坊交互)                                   │
│    └── providers (RPC 提供者)                               │
│                                                             │
│  anvil          本地测试节点                                │
│    │                                                        │
│    ├── anvil-core (节点核心)                                │
│    ├── evm (EVM 执行)                                       │
│    └── rpc (RPC 服务)                                       │
│                                                             │
│  chisel        Solidity REPL                                │
│    │                                                        │
│    └── solc-compiler (编译器接口)                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 核心依赖库

```
foundry/
├── crates/
│   ├── evm/                    # EVM 实现
│   │   ├── evm-core/           # 核心 EVM 逻辑
│   │   ├── evm-executors/      # 执行器
│   │   └── evm-trace/          # 追踪功能
│   │
│   ├── forge/                  # forge 命令
│   │   ├── forge/              # CLI 入口
│   │   └── forge-core/         # 测试核心逻辑
│   │
│   ├── anvil/                  # anvil 节点
│   │   ├── anvil/              # CLI 入口
│   │   ├── anvil-core/         # 节点核心
│   │   └── anvil-rpc/          # RPC 实现
│   │
│   ├── cast/                   # cast 命令
│   │
│   ├── config/                 # 配置管理
│   ├── solc/                   # Solidity 编译器接口
│   ├── test-utils/             # 测试工具
│   └── utils/                  # 通用工具
│
├── ethers-rs/                  # 以太坊 Rust 库
└── ethers-solc/                # Solidity 编译
```

## EVM 执行流程

### 核心 EVM 结构

```rust
pub struct EVM<DB> {
    pub db: DB,
    pub env: Env,
    pub handler: Handler,
}

pub struct Env {
    pub cfg: CfgEnv,
    pub block: BlockEnv,
    pub tx: TxEnv,
}

pub struct BlockEnv {
    pub number: U256,
    pub coinbase: Address,
    pub timestamp: U256,
    pub difficulty: U256,
    pub prevrandao: Option<H256>,
    pub basefee: U256,
    pub gas_limit: U256,
}

pub struct TxEnv {
    pub caller: Address,
    pub gas_limit: u64,
    pub gas_price: U256,
    pub transact_to: TransactTo,
    pub value: U256,
    pub data: Bytes,
    pub nonce: Option<u64>,
    pub chain_id: Option<u64>,
}
```

### 交易执行

```rust
impl<DB: Database> EVM<DB> {
    pub fn transact(&mut self) -> Result<ResultAndState, EVMError<DB::Error>> {
        self.handler.execute(
            &mut self.db,
            &mut self.env,
        )
    }
}

impl Handler {
    pub fn execute<DB: Database>(
        &self,
        db: &mut DB,
        env: &mut Env,
    ) -> Result<ResultAndState, EVMError<DB::Error>> {
        let pre_exec = self.pre_execution(db, env)?;
        
        let exec_result = self.execution(db, env, pre_exec)?;
        
        let post_exec = self.post_execution(db, env, exec_result)?;
        
        Ok(post_exec)
    }
}
```

### forge 测试流程

```rust
pub struct TestRunner {
    evm: EVM<Backend>,
    traces: Vec<CallTraceNode>,
}

impl TestRunner {
    pub fn run_test(
        &mut self,
        contract: &Contract,
        test_fn: &str,
    ) -> TestResult {
        let calldata = self.encode_test_call(contract, test_fn);
        
        self.evm.env.tx.data = calldata;
        self.evm.env.tx.transact_to = TransactTo::Call(contract.address);
        
        let result = self.evm.transact();
        
        TestResult {
            success: result.is_ok() && self.check_assertions(&result),
            gas_used: result.gas_used(),
            traces: self.traces.clone(),
            logs: result.logs(),
        }
    }
}
```

## Solidity 编译器集成

### solc 接口

```rust
pub struct Solc {
    pub solc_path: PathBuf,
    pub version: Version,
}

impl Solc {
    pub fn compile(
        &self,
        input: CompilerInput,
    ) -> Result<CompilerOutput, SolcError> {
        let input_json = serde_json::to_string(&input)?;
        
        let output = self.solc_path
            .with_stdin(Stdio::piped())
            .args(&["--standard-json"])
            .output()?;
        
        let output: CompilerOutput = serde_json::from_slice(&output.stdout)?;
        Ok(output)
    }
}

#[derive(Serialize)]
pub struct CompilerInput {
    pub language: String,
    pub sources: BTreeMap<PathBuf, Source>,
    pub settings: Settings,
}

#[derive(Deserialize)]
pub struct CompilerOutput {
    pub errors: Vec<Error>,
    pub sources: BTreeMap<PathBuf, SourceFile>,
    pub contracts: BTreeMap<PathBuf, BTreeMap<String, Contract>>,
}

#[derive(Deserialize)]
pub struct Contract {
    pub abi: Vec<AbiEntry>,
    pub bin: Option<String>,
    pub bin_runtime: Option<String>,
    pub bytecode: Option<Bytecode>,
    pub deployed_bytecode: Option<Bytecode>,
}
```

### 编译流程

```rust
pub struct ProjectCompiler {
    project: Project,
    solc: Solc,
}

impl ProjectCompiler {
    pub fn compile(&self) -> Result<ArtifactOutput, CompileError> {
        let sources = self.collect_sources()?;
        
        let input = CompilerInput {
            language: "Solidity".to_string(),
            sources: sources.iter()
                .map(|(path, content)| {
                    (path.clone(), Source { content: content.clone() })
                })
                .collect(),
            settings: self.project.settings(),
        };
        
        let output = self.solc.compile(input)?;
        
        if output.has_errors() {
            return Err(CompileError::from_output(output));
        }
        
        let artifacts = self.build_artifacts(output)?;
        Ok(artifacts)
    }
    
    fn collect_sources(&self) -> Result<BTreeMap<PathBuf, String>, IoError> {
        let mut sources = BTreeMap::new();
        
        for entry in glob(&self.project.sources_pattern())? {
            let path = entry?;
            let content = std::fs::read_to_string(&path)?;
            sources.insert(path, content);
        }
        
        Ok(sources)
    }
}
```

## 模糊测试（Fuzz Testing）

### 测试框架

```rust
pub struct FuzzTestRunner {
    strategy: Box<dyn Strategy>,
    runs: usize,
}

pub trait Strategy {
    fn generate(&self) -> Vec<DynamicValue>;
}

pub struct DynSolStrategy {
    param_types: Vec<DynSolType>,
}

impl Strategy for DynSolStrategy {
    fn generate(&self) -> Vec<DynamicValue> {
        self.param_types
            .iter()
            .map(|ty| self.random_value_for_type(ty))
            .collect()
    }
}

impl FuzzTestRunner {
    pub fn run(
        &mut self,
        test_fn: &TestFunction,
        evm: &mut EVM<Backend>,
    ) -> FuzzTestResult {
        let mut successes = 0;
        let mut failures = Vec::new();
        
        for run in 0..self.runs {
            let inputs = self.strategy.generate();
            
            let calldata = test_fn.encode(&inputs);
            evm.env.tx.data = calldata;
            
            match evm.transact() {
                Ok(result) if result.is_success() => {
                    successes += 1;
                }
                Ok(result) => {
                    failures.push(FuzzFailure {
                        run,
                        inputs: inputs.clone(),
                        reason: result.reason(),
                    });
                }
                Err(e) => {
                    failures.push(FuzzFailure {
                        run,
                        inputs,
                        reason: format!("EVM error: {:?}", e),
                    });
                }
            }
        }
        
        FuzzTestResult {
            runs: self.runs,
            successes,
            failures,
        }
    }
}
```

### 模糊测试参数生成

```rust
impl FuzzParamGenerator {
    pub fn generate_for_type(ty: &DynSolType) -> DynamicValue {
        match ty {
            DynSolType::Uint(bits) => {
                let max = U256::from(2).pow(U256::from(*bits));
                let value = rand::thread_rng().gen_range(0..max);
                DynamicValue::Uint(value)
            }
            DynSolType::Int(bits) => {
                let max = I256::from(2).pow(I256::from(*bits - 1));
                let value = rand::thread_rng().gen_range(-max..max);
                DynamicValue::Int(value)
            }
            DynSolType::Address => {
                let bytes: [u8; 20] = rand::thread_rng().gen();
                DynamicValue::Address(Address::from(bytes))
            }
            DynSolType::Bool => {
                DynamicValue::Bool(rand::thread_rng().gen())
            }
            DynSolType::Bytes => {
                let len = rand::thread_rng().gen_range(0..1024);
                let mut bytes = vec![0u8; len];
                rand::thread_rng().fill_bytes(&mut bytes);
                DynamicValue::Bytes(Bytes::from(bytes))
            }
            DynSolType::String => {
                let s = rand::thread_rng()
                    .sample_iter(&Alphanumeric)
                    .take(rand::thread_rng().gen_range(0..100))
                    .map(char::from)
                    .collect::<String>();
                DynamicValue::String(s)
            }
            DynSolType::Array(inner) => {
                let len = rand::thread_rng().gen_range(0..10);
                let arr = (0..len)
                    .map(|_| Self::generate_for_type(inner))
                    .collect();
                DynamicValue::Array(arr)
            }
            DynSolType::Tuple(types) => {
                let tuple = types
                    .iter()
                    .map(|t| Self::generate_for_type(t))
                    .collect();
                DynamicValue::Tuple(tuple)
            }
            _ => DynamicValue::default(),
        }
    }
}
```

## Anvil 本地节点

### 核心架构

```rust
pub struct AnvilNode {
    pub backend: Backend,
    pub evm: EVM<Backend>,
    pub miner: Miner,
    pub rpc: RpcServer,
}

pub struct Backend {
    storage: Storage,
    state: State,
    block: BlockInfo,
}

impl Backend {
    pub fn new(config: AnvilConfig) -> Self {
        let genesis = config.genesis_block();
        
        Self {
            storage: Storage::new(),
            state: State::from_genesis(genesis),
            block: BlockInfo::genesis(),
        }
    }
    
    pub fn execute_transaction(
        &mut self,
        tx: Transaction,
    ) -> Result<TransactionResult, Error> {
        let from = tx.from;
        let nonce = self.state.nonce(from);
        
        if tx.nonce != nonce {
            return Err(Error::InvalidNonce);
        }
        
        let gas_cost = tx.gas_price * U256::from(tx.gas_limit);
        let balance = self.state.balance(from);
        
        if balance < gas_cost {
            return Err(Error::InsufficientBalance);
        }
        
        let result = self.evm.transact(tx)?;
        
        self.state.apply(&result.state)?;
        
        Ok(result)
    }
}

pub struct Miner {
    mode: MiningMode,
    pending_transactions: Vec<Transaction>,
}

pub enum MiningMode {
    Auto,
    Interval(Duration),
    Manual,
}

impl Miner {
    pub fn mine_block(
        &mut self,
        backend: &mut Backend,
    ) -> Result<Block, Error> {
        let mut block_txs = Vec::new();
        
        for tx in self.pending_transactions.drain(..) {
            match backend.execute_transaction(tx.clone()) {
                Ok(result) => {
                    block_txs.push((tx, result));
                }
                Err(e) => {
                    tracing::warn!("Transaction failed: {:?}", e);
                }
            }
        }
        
        let block = backend.create_block(block_txs)?;
        Ok(block)
    }
}
```

### RPC 服务

```rust
pub struct RpcServer {
    node: Arc<Mutex<AnvilNode>>,
}

impl RpcServer {
    pub async fn handle_request(
        &self,
        method: &str,
        params: Value,
    ) -> Result<Value, RpcError> {
        match method {
            "eth_blockNumber" => {
                let node = self.node.lock().await;
                Ok(json!(node.backend.block.number))
            }
            "eth_getBalance" => {
                let address: Address = serde_json::from_value(params[0].clone())?;
                let node = self.node.lock().await;
                Ok(json!(node.backend.state.balance(address)))
            }
            "eth_sendTransaction" => {
                let tx: TransactionRequest = serde_json::from_value(params[0].clone())?;
                let mut node = self.node.lock().await;
                let hash = node.send_transaction(tx)?;
                Ok(json!(hash))
            }
            "eth_call" => {
                let call: CallRequest = serde_json::from_value(params[0].clone())?;
                let node = self.node.lock().await;
                let result = node.call(call)?;
                Ok(json!(result))
            }
            "eth_getTransactionReceipt" => {
                let hash: H256 = serde_json::from_value(params[0].clone())?;
                let node = self.node.lock().await;
                let receipt = node.get_receipt(hash)?;
                Ok(serde_json::to_value(receipt)?)
            }
            "evm_mine" => {
                let mut node = self.node.lock().await;
                node.mine()?;
                Ok(json!(null))
            }
            "anvil_impersonateAccount" => {
                let address: Address = serde_json::from_value(params[0].clone())?;
                let mut node = self.node.lock().await;
                node.impersonate(address)?;
                Ok(json!(null))
            }
            _ => Err(RpcError::MethodNotFound(method.to_string())),
        }
    }
}
```

## Cast 命令实现

```rust
pub struct Cast {
    provider: RootProvider<Http<Client>>,
}

impl Cast {
    pub async fn call(
        &self,
        to: Address,
        sig: &str,
        args: &[String],
    ) -> Result<String, CastError> {
        let (selector, types) = parse_signature(sig)?;
        
        let encoded_args = encode_args(&types, args)?;
        let calldata = [&selector[..], &encoded_args[..]].concat();
        
        let result = self.provider
            .call(
                TransactionRequest {
                    to: Some(to),
                    data: Some(calldata.into()),
                    ..Default::default()
                },
                None,
            )
            .await?;
        
        let decoded = decode_output(&types, &result)?;
        Ok(format_tokens(&decoded))
    }
    
    pub async fn send(
        &self,
        from: Address,
        to: Address,
        sig: &str,
        args: &[String],
        value: Option<U256>,
    ) -> Result<TxHash, CastError> {
        let (selector, types) = parse_signature(sig)?;
        let encoded_args = encode_args(&types, args)?;
        let calldata = [&selector[..], &encoded_args[..]].concat();
        
        let tx = TransactionRequest {
            from: Some(from),
            to: Some(to),
            value,
            data: Some(calldata.into()),
            ..Default::default()
        };
        
        let pending = self.provider.send_transaction(tx).await?;
        Ok(pending.tx_hash())
    }
    
    pub async fn estimate_gas(
        &self,
        from: Address,
        to: Address,
        sig: &str,
        args: &[String],
    ) -> Result<U256, CastError> {
        let (selector, types) = parse_signature(sig)?;
        let encoded_args = encode_args(&types, args)?;
        let calldata = [&selector[..], &encoded_args[..]].concat();
        
        let gas = self.provider
            .estimate_gas(
                TransactionRequest {
                    from: Some(from),
                    to: Some(to),
                    data: Some(calldata.into()),
                    ..Default::default()
                },
                None,
            )
            .await?;
        
        Ok(gas)
    }
}
```

## 追踪（Tracing）系统

```rust
#[derive(Debug, Clone)]
pub struct CallTrace {
    pub depth: usize,
    pub caller: Address,
    pub to: Address,
    pub value: U256,
    pub data: Bytes,
    pub gas_used: u64,
    pub status: bool,
    pub output: Bytes,
    pub calls: Vec<CallTrace>,
}

pub struct Tracer {
    traces: Vec<CallTrace>,
    current_depth: usize,
}

impl Tracer {
    pub fn on_call(
        &mut self,
        caller: Address,
        to: Address,
        value: U256,
        data: Bytes,
    ) {
        let trace = CallTrace {
            depth: self.current_depth,
            caller,
            to,
            value,
            data,
            gas_used: 0,
            status: false,
            output: Bytes::default(),
            calls: Vec::new(),
        };
        
        self.traces.push(trace);
        self.current_depth += 1;
    }
    
    pub fn on_return(
        &mut self,
        status: bool,
        output: Bytes,
        gas_used: u64,
    ) {
        self.current_depth -= 1;
        
        if let Some(trace) = self.traces.last_mut() {
            trace.status = status;
            trace.output = output;
            trace.gas_used = gas_used;
        }
    }
    
    pub fn format_traces(&self) -> String {
        let mut output = String::new();
        
        for trace in &self.traces {
            self.format_trace(trace, 0, &mut output);
        }
        
        output
    }
    
    fn format_trace(
        &self,
        trace: &CallTrace,
        indent: usize,
        output: &mut String,
    ) {
        let indent_str = "  ".repeat(indent);
        
        output.push_str(&format!(
            "{}[{}] {}::{} [gas: {}]\n",
            indent_str,
            if trace.status { "SUCCESS" } else { "FAIL" },
            trace.caller,
            trace.to,
            trace.gas_used
        ));
        
        for sub_trace in &trace.calls {
            self.format_trace(sub_trace, indent + 1, output);
        }
    }
}
```

## 学习资源

```
┌─────────────────────────────────────────────────────────────┐
│              Foundry 源码学习路径                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 入门：cast 命令                                         │
│     crates/cast/src/lib.rs                                  │
│     - 简单的 RPC 调用                                       │
│     - ABI 编码/解码                                         │
│     - 类型转换                                              │
│                                                             │
│  2. 进阶：solc 编译                                         │
│     crates/solc/src/lib.rs                                  │
│     - 编译器调用                                            │
│     - 输入/输出处理                                         │
│     - 依赖解析                                              │
│                                                             │
│  3. 核心：EVM 实现                                          │
│     crates/evm/                                             │
│     - EVM 执行                                              │
│     - 追踪系统                                              │
│     - 测试框架                                              │
│                                                             │
│  4. 高级：anvil 节点                                        │
│     crates/anvil/                                           │
│     - 完整节点实现                                          │
│     - RPC 服务                                              │
│     - 状态管理                                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 关键代码位置

| 功能 | 文件路径 |
|------|----------|
| EVM 执行 | `crates/evm/evm-core/src/` |
| 测试框架 | `crates/forge/forge-core/src/` |
| 模糊测试 | `crates/forge/forge-core/src/fuzz.rs` |
| Gas 快照 | `crates/forge/forge-core/src/snapshot.rs` |
| Solidity 编译 | `crates/solc/src/lib.rs` |
| RPC 服务 | `crates/anvil/anvil-rpc/src/` |
| 追踪解码 | `crates/evm/evm-trace/src/decoder.rs` |

[下一节：链下工具开发 →](05-offchain-tools.md)
