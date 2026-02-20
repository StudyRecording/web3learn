# Alloy 库使用

> 用 Rust 与以太坊交互

## Alloy 简介

Alloy 是 Paradigm 开发的新一代以太坊 Rust 库，旨在替代 ethers-rs。它提供了类型安全、高性能的以太坊交互能力。

```
┌─────────────────────────────────────────────────────────────┐
│                    Alloy 核心组件                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  alloy-provider      - RPC 提供者（与节点通信）              │
│  alloy-contract      - 合约交互抽象                          │
│  alloy-signer        - 交易签名                              │
│  alloy-signer-wallet - 本地钱包签名                          │
│  alloy-signer-ledger - Ledger 硬件钱包                       │
│  alloy-rpc-client    - 底层 RPC 客户端                       │
│  alloy-network       - 网络类型定义                          │
│  alloy-primitives    - 基础类型（Address, U256, Bytes等）    │
│  alloy-sol-types     - Solidity 类型绑定                     │
│  alloy-json-rpc      - JSON-RPC 协议实现                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 项目配置

```toml
# Cargo.toml
[dependencies]
alloy = { version = "0.6", features = ["full", "node-bindings"] }
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
anyhow = "1"
```

## 基础连接

```rust
use alloy::{
    providers::{Provider, ProviderBuilder, RootProvider},
    transports::http::{Http, Client},
};
use url::Url;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let rpc_url = "https://eth.llamarpc.com".parse::<Url>()?;
    
    let provider = ProviderBuilder::new().on_http(rpc_url);
    
    let block_number = provider.get_block_number().await?;
    println!("Current block: {}", block_number);
    
    let chain_id = provider.get_chain_id().await?;
    println!("Chain ID: {}", chain_id);
    
    let gas_price = provider.get_gas_price().await?;
    println!("Gas price: {} wei", gas_price);
    
    Ok(())
}
```

**对比 ethers.js：**

```javascript
const { ethers } = require("ethers");

const provider = new ethers.JsonRpcProvider("https://eth.llamarpc.com");

async function main() {
  const blockNumber = await provider.getBlockNumber();
  console.log("Current block:", blockNumber);
  
  const chainId = (await provider.getNetwork()).chainId;
  console.log("Chain ID:", chainId);
  
  const gasPrice = (await provider.getFeeData()).gasPrice;
  console.log("Gas price:", gasPrice.toString(), "wei");
}

main();
```

## 基础类型

```rust
use alloy::primitives::{Address, U256, Bytes, B256, FixedBytes};

fn primitives_demo() {
    let address: Address = "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045".parse()?;
    println!("Address: {}", address);
    
    let amount = U256::from(1_000_000_000_000_000_000u128);
    let doubled = amount * U256::from(2);
    println!("Doubled: {}", doubled);
    
    let bytes = Bytes::from(vec![0x01, 0x02, 0x03]);
    println!("Bytes: {:?}", bytes);
    
    let hash: B256 = "0x1234...".parse()?;
    println!("Hash: {}", hash);
    
    Ok(())
}
```

## 查询余额和账户信息

```rust
use alloy::providers::{Provider, ProviderBuilder};
use alloy::primitives::{Address, U256};
use url::Url;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let rpc_url = "https://eth.llamarpc.com".parse::<Url>()?;
    let provider = ProviderBuilder::new().on_http(rpc_url);
    
    let address: Address = "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045".parse()?;
    
    let balance = provider.get_balance(address).await?;
    println!("Balance: {} wei", balance);
    println!("Balance: {} ETH", balance / U256::from(10u128.pow(18)));
    
    let tx_count = provider.get_transaction_count(address).await?;
    println!("Transaction count: {}", tx_count);
    
    let code = provider.get_code_at(address).await?;
    println!("Is contract: {}", !code.is_empty());
    
    Ok(())
}
```

## 读取合约状态

### 使用 Sol! 宏定义接口

```rust
use alloy::{sol, providers::{Provider, ProviderBuilder}};
use alloy::primitives::Address;
use url::Url;

sol! {
    #[sol(rpc)]
    interface ERC20 {
        function name() external view returns (string);
        function symbol() external view returns (string);
        function decimals() external view returns (uint8);
        function totalSupply() external view returns (uint256);
        function balanceOf(address owner) external view returns (uint256);
        
        event Transfer(address indexed from, address indexed to, uint256 value);
        event Approval(address indexed owner, address indexed spender, uint256 value);
    }
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let rpc_url = "https://eth.llamarpc.com".parse::<Url>()?;
    let provider = ProviderBuilder::new().on_http(rpc_url);
    
    let usdc_address: Address = "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48".parse()?;
    let erc20 = ERC20::new(usdc_address, provider);
    
    let name = erc20.name().call().await?._0;
    println!("Name: {}", name);
    
    let symbol = erc20.symbol().call().await?._0;
    println!("Symbol: {}", symbol);
    
    let decimals = erc20.decimals().call().await?._0;
    println!("Decimals: {}", decimals);
    
    let total_supply = erc20.totalSupply().call().await?._0;
    println!("Total supply: {}", total_supply);
    
    let owner: Address = "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045".parse()?;
    let balance = erc20.balanceOf(owner).call().await?._0;
    println!("Balance: {}", balance);
    
    Ok(())
}
```

**对比 ethers.js：**

```javascript
const { ethers } = require("ethers");

const ERC20_ABI = [
  "function name() view returns (string)",
  "function symbol() view returns (string)",
  "function decimals() view returns (uint8)",
  "function totalSupply() view returns (uint256)",
  "function balanceOf(address) view returns (uint256)"
];

async function main() {
  const provider = new ethers.JsonRpcProvider("https://eth.llamarpc.com");
  
  const usdcAddress = "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48";
  const contract = new ethers.Contract(usdcAddress, ERC20_ABI, provider);
  
  const name = await contract.name();
  const symbol = await contract.symbol();
  const decimals = await contract.decimals();
  const totalSupply = await contract.totalSupply();
  
  console.log({ name, symbol, decimals, totalSupply });
}
```

## 发送交易

### 创建钱包

```rust
use alloy::{
    network::{EthereumWallet, TransactionBuilder},
    primitives::Address,
    providers::{Provider, ProviderBuilder},
    signers::{Signer, local::PrivateKeySigner},
};
use url::Url;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let signer: PrivateKeySigner = "0x...private_key...".parse()?;
    let wallet = EthereumWallet::from(signer);
    
    let rpc_url = "https://eth.llamarpc.com".parse::<Url>()?;
    let provider = ProviderBuilder::new()
        .wallet(wallet)
        .on_http(rpc_url);
    
    let from_address = provider.default_signer_address();
    println!("From: {}", from_address);
    
    let to_address: Address = "0xRecipientAddress".parse()?;
    
    let tx = alloy::rpc::types::TransactionRequest::default()
        .with_to(to_address)
        .with_value(alloy::primitives::U256::from(1_000_000_000_000_000_000u128));
    
    let pending_tx = provider.send_transaction(tx).await?;
    println!("Transaction hash: {}", pending_tx.tx_hash());
    
    let receipt = pending_tx.get_receipt().await?;
    println!("Transaction confirmed in block: {}", receipt.block_number.unwrap());
    
    Ok(())
}
```

### 合约写入操作

```rust
use alloy::{sol, providers::{Provider, ProviderBuilder}, signers::local::PrivateKeySigner, network::EthereumWallet};
use alloy::primitives::Address;
use url::Url;

sol! {
    #[sol(rpc)]
    interface SimpleToken {
        function transfer(address to, uint256 amount) external returns (bool);
        function approve(address spender, uint256 amount) external returns (bool);
        function mint(address to, uint256 amount) external;
        
        error InsufficientBalance();
        error InsufficientAllowance();
    }
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let signer: PrivateKeySigner = "0x...private_key...".parse()?;
    let wallet = EthereumWallet::from(signer);
    
    let rpc_url = "https://rpc-url".parse::<Url>()?;
    let provider = ProviderBuilder::new()
        .wallet(wallet)
        .on_http(rpc_url);
    
    let token_address: Address = "0xTokenAddress".parse()?;
    let token = SimpleToken::new(token_address, &provider);
    
    let recipient: Address = "0xRecipientAddress".parse()?;
    let amount = alloy::primitives::U256::from(1_000_000u128);
    
    let builder = token.transfer(recipient, amount);
    let result = builder.send().await?;
    
    println!("Transaction hash: {}", result.tx_hash());
    
    let receipt = result.get_receipt().await?;
    println!("Status: {:?}", receipt.status());
    
    Ok(())
}
```

## 监听事件

```rust
use alloy::{sol, providers::{Provider, ProviderBuilder, filter::Filter}};
use alloy::primitives::Address;
use futures::StreamExt;
use url::Url;

sol! {
    event Transfer(address indexed from, address indexed to, uint256 value);
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let rpc_url = "wss://eth-mainnet.g.alchemy.com/v2/YOUR_KEY".parse::<Url>()?;
    let provider = ProviderBuilder::new().on_ws(rpc_url).await?;
    
    let token_address: Address = "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48".parse()?;
    
    let filter = Filter::new()
        .address(token_address)
        .event_signature(Transfer::SIGNATURE_HASH);
    
    let mut stream = provider.subscribe_logs(&filter).await?.into_stream();
    
    println!("Listening for Transfer events...");
    
    while let Some(log) = stream.next().await {
        let transfer = Transfer::decode_log_data(log.data(), true)?;
        println!(
            "Transfer: {} -> {}: {}",
            transfer.from, transfer.to, transfer.value
        );
    }
    
    Ok(())
}
```

**对比 ethers.js：**

```javascript
const { ethers } = require("ethers");

async function main() {
  const wsProvider = new ethers.WebSocketProvider("wss://eth-mainnet.g.alchemy.com/v2/YOUR_KEY");
  
  const tokenAddress = "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48";
  const filter = {
    address: tokenAddress,
    topics: [ethers.id("Transfer(address,address,uint256)")]
  };
  
  wsProvider.on(filter, (log) => {
    const iface = new ethers.Interface(["event Transfer(address indexed from, address indexed to, uint256 value)"]);
    const event = iface.parseLog(log);
    console.log(`Transfer: ${event.args.from} -> ${event.args.to}: ${event.args.value}`);
  });
}
```

## 查询历史事件

```rust
use alloy::{sol, providers::{Provider, ProviderBuilder, filter::Filter}};
use alloy::primitives::Address;
use url::Url;

sol! {
    event Transfer(address indexed from, address indexed to, uint256 value);
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let rpc_url = "https://eth.llamarpc.com".parse::<Url>()?;
    let provider = ProviderBuilder::new().on_http(rpc_url);
    
    let token_address: Address = "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48".parse()?;
    
    let current_block = provider.get_block_number().await?;
    
    let filter = Filter::new()
        .address(token_address)
        .event_signature(Transfer::SIGNATURE_HASH)
        .from_block(current_block - 1000)
        .to_block(current_block);
    
    let logs = provider.get_logs(&filter).await?;
    
    println!("Found {} Transfer events", logs.len());
    
    for log in logs.iter().take(10) {
        let transfer = Transfer::decode_log_data(&log.data, true)?;
        println!(
            "Block {}: {} -> {}: {}",
            log.block_number.unwrap_or(0),
            transfer.from,
            transfer.to,
            transfer.value
        );
    }
    
    Ok(())
}
```

## 实战：批量查询工具

```rust
use alloy::{
    sol,
    providers::{Provider, ProviderBuilder},
    primitives::{Address, U256},
};
use std::collections::HashMap;
use url::Url;

sol! {
    #[sol(rpc)]
    interface ERC20 {
        function name() external view returns (string);
        function symbol() external view returns (string);
        function decimals() external view returns (uint8);
        function balanceOf(address owner) external view returns (uint256);
    }
}

struct TokenInfo {
    address: Address,
    name: String,
    symbol: String,
    decimals: u8,
    balance: U256,
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let rpc_url = "https://eth.llamarpc.com".parse::<Url>()?;
    let provider = ProviderBuilder::new().on_http(rpc_url);
    
    let wallet: Address = "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045".parse()?;
    
    let tokens = vec![
        "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48", // USDC
        "0xdAC17F958D2ee523a2206206994597C13D831ec7", // USDT
        "0x6B175474E89094C44Da98b954EescdeCB5166BAC", // DAI
        "0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984", // UNI
    ];
    
    let mut results = Vec::new();
    
    for token_str in tokens {
        let token_address: Address = token_str.parse()?;
        let erc20 = ERC20::new(token_address, &provider);
        
        let (name, symbol, decimals, balance) = tokio::try_join!(
            async { Ok::<_, anyhow::Error>(erc20.name().call().await?._0) },
            async { Ok::<_, anyhow::Error>(erc20.symbol().call().await?._0) },
            async { Ok::<_, anyhow::Error>(erc20.decimals().call().await?._0) },
            async { Ok::<_, anyhow::Error>(erc20.balanceOf(wallet).call().await?._0) },
        )?;
        
        results.push(TokenInfo {
            address: token_address,
            name,
            symbol,
            decimals,
            balance,
        });
    }
    
    println!("Token Balances for {}", wallet);
    println!("{:-<60}", "");
    
    for info in results {
        let divisor = U256::from(10u128.pow(info.decimals as u32));
        let human_balance = info.balance / divisor;
        
        println!(
            "{} ({}) - {} {} (decimals: {})",
            info.symbol,
            info.name,
            human_balance,
            info.symbol,
            info.decimals
        );
    }
    
    Ok(())
}
```

## 实战：交易监控机器人

```rust
use alloy::{
    providers::{Provider, ProviderBuilder},
    primitives::Address,
    rpc::types::Block,
};
use futures::StreamExt;
use url::Url;

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let rpc_url = "wss://eth-mainnet.g.alchemy.com/v2/YOUR_KEY".parse::<Url>()?;
    let provider = ProviderBuilder::new().on_ws(rpc_url).await?;
    
    let monitored: Address = "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045".parse()?;
    
    let mut stream = provider.subscribe_blocks().await?.into_stream();
    
    println!("Monitoring blocks for transactions involving: {}", monitored);
    
    while let Some(block) = stream.next().await {
        let block: Block = block;
        
        for tx in block.transactions {
            if let Some(from) = tx.from {
                if from == monitored {
                    println!(
                        "[{}] TX from monitored address: {}",
                        block.header.number,
                        tx.hash
                    );
                }
            }
            
            if let Some(to) = tx.to {
                if to == monitored {
                    println!(
                        "[{}] TX to monitored address: {}",
                        block.header.number,
                        tx.hash
                    );
                }
            }
        }
    }
    
    Ok(())
}
```

## ABI 编码/解码

```rust
use alloy::{sol, sol_types::SolValue};

sol! {
    struct UserInfo {
        address user;
        uint256 amount;
        uint256 timestamp;
    }
    
    function deposit(address user, uint256 amount) external returns (uint256);
}

fn main() -> anyhow::Result<()> {
    let user: alloy::primitives::Address = "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045".parse()?;
    let amount = alloy::primitives::U256::from(1000);
    
    let call_data = depositCall::new((user, amount)).abi_encode();
    println!("Encoded call data: 0x{}", hex::encode(&call_data));
    
    let info = UserInfo {
        user,
        amount,
        timestamp: alloy::primitives::U256::from(1234567890),
    };
    
    let encoded = info.abi_encode();
    println!("Encoded struct: 0x{}", hex::encode(&encoded));
    
    let decoded: UserInfo = UserInfo::abi_decode(&encoded)?;
    println!("Decoded: user={}, amount={}, timestamp={}", 
        decoded.user, decoded.amount, decoded.timestamp);
    
    Ok(())
}
```

## 类型对照表

| Solidity | Alloy Rust | ethers.js |
|----------|------------|-----------|
| address | Address | string |
| uint256 | U256 | BigInt |
| bytes | Bytes | Uint8Array |
| bytes32 | FixedBytes<32> | string |
| bool | bool | boolean |
| string | String | string |
| address[] | Vec<Address> | string[] |
| mapping | - | - |
| struct | struct (sol!) | object |
| enum | u8 | number |

## 关键差异总结

```
┌─────────────────────────────────────────────────────────────┐
│         ethers.js vs Alloy 关键差异                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 类型安全                                                 │
│     ethers.js: JavaScript 动态类型                          │
│     Alloy: Rust 静态类型，编译时检查                         │
│                                                             │
│  2. ABI 定义                                                 │
│     ethers.js: JSON ABI 或字符串                            │
│     Alloy: sol! 宏，原生 Rust 语法                          │
│                                                             │
│  3. 错误处理                                                 │
│     ethers.js: try-catch                                    │
│     Alloy: Result<T, E> 类型                                │
│                                                             │
│  4. 异步模型                                                 │
│     ethers.js: Promise                                      │
│     Alloy: async/await + tokio                              │
│                                                             │
│  5. 性能                                                     │
│     ethers.js: Node.js 运行时                               │
│     Alloy: 原生 Rust 性能                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

[下一节：Foundry 内部原理 →](04-foundry-internal.md)
