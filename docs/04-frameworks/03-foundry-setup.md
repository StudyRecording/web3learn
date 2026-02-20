# Foundry 环境搭建

Foundry 是用 Rust 编写的极速以太坊开发框架。作为有 Rust 背景的开发者，你会欣赏它的设计哲学和性能。

## 什么是 Foundry？

Foundry 由四个核心工具组成：

| 工具 | 功能 | 类比 |
|------|------|------|
| **forge** | 编译、测试、部署 | cargo + cargo test |
| **cast** | 与区块链交互 | ethers.js CLI |
| **anvil** | 本地测试网络 | ganache/hardhat node |
| **chisel** | Solidity REPL | node/python REPL |

## 安装 Foundry

### 方式一：foundryup（推荐）

```bash
# macOS/Linux
curl -L https://foundry.paradigm.xyz | bash

# 将 foundry 添加到 PATH
source ~/.bashrc  # 或 ~/.zshrc

# 安装最新版本
foundryup

# 安装特定版本
foundryup --version v0.3.0
```

```powershell
# Windows PowerShell
Invoke-WebRequest -Uri https://foundry.paradigm.xyz -OutFile foundryup.ps1
.\foundryup.ps1
```

### 方式二：从源码编译（Rust 开发者推荐）

```bash
# 克隆仓库
git clone https://github.com/foundry-rs/foundry.git
cd foundry

# 使用 cargo 安装
cargo install --path ./crates/forge --profile local --force
cargo install --path ./crates/cast --profile local --force
cargo install --path ./crates/anvil --profile local --force
cargo install --path ./crates/chisel --profile local --force
```

### 验证安装

```bash
# 检查版本
forge --version
cast --version
anvil --version

# 输出示例：
# forge 0.2.0 (e04875b 2024-01-15T00:00:00.000000000Z)
# cast 0.2.0 (e04875b 2024-01-15T00:00:00.000000000Z)
# anvil 0.2.0 (e04875b 2024-01-15T00:00:00.000000000Z)
```

## 创建 Foundry 项目

### 初始化项目

```bash
# 创建新项目
forge init my-foundry-project

cd my-foundry-project
```

输出：

```
Initializing /path/to/my-foundry-project...
Installing forge-std in "/path/to/my-foundry-project/lib/forge-std" (url: Some("https://github.com/foundry-rs/forge-std"), tag: None)
    Installed forge-std
    Initialized forge project
```

### 项目结构

```
my-foundry-project/
├── lib/                    # 依赖库（git submodules）
│   └── forge-std/         # Foundry 标准库
├── src/                    # 合约源代码
│   └── Counter.sol        # 示例合约
├── test/                   # 测试文件
│   └── Counter.t.sol      # 测试合约
├── script/                 # 部署脚本
│   └── Counter.s.sol      # 部署脚本
├── out/                    # 编译产物
├── broadcast/              # 部署记录
├── foundry.toml           # Foundry 配置
└── .gitmodules            # Git 子模块配置
```

### 与 Hardhat/Java 项目对比

| Foundry | Hardhat | Java | 说明 |
|---------|---------|------|------|
| src/ | contracts/ | src/main/java/ | 源代码 |
| test/ | test/ | src/test/java/ | 测试代码 |
| lib/ | node_modules/ | target/... | 依赖 |
| foundry.toml | hardhat.config.js | pom.xml | 配置文件 |
| script/ | scripts/ | - | 部署脚本 |
| out/ | artifacts/ | target/ | 编译产物 |

## 配置详解

### 基础配置

```toml
# foundry.toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]

# Solidity 版本
solc = "0.8.24"

# 优化器
optimizer = true
optimizer_runs = 200

# EVM 版本
evm_version = "paris"

# 测试配置
verbosity = 3
fuzz = { runs = 256 }
invariant = { runs = 256 }
```

### 完整配置示例

```toml
# foundry.toml

# 默认配置
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
cache_path = "cache"
broadcast_path = "broadcast"

# 编译器配置
solc = "0.8.24"
auto_detect_solc = false
optimizer = true
optimizer_runs = 200
via_ir = false

# EVM 版本（支持：paris, shanghai, cancun）
evm_version = "paris"

# 重映射
remappings = [
    "@openzeppelin/contracts=lib/openzeppelin-contracts/contracts",
    "forge-std/=lib/forge-std/src/",
    "solmate/=lib/solmate/src/",
]

# 测试配置
verbosity = 3
fuzz = { runs = 256, max_test_rejects = 65536 }
invariant = { runs = 256, depth = 15 }

# Gas 报告
gas_reports = ["*"]
gas_reports_ignore = []

# 格式化
fmt = { line_length = 120, tab_width = 4, bracket_spacing = false }

# 代码检查
[profile.default.fuzz]
runs = 256

# 测试网络配置
[profile.default.rpc_storage_caching]
chains = [1, 5, 11155111]
endpoints = "all"

# 开发环境配置
[profile.dev]
verbosity = 4
fuzz = { runs = 100 }

# 生产环境配置
[profile.prod]
optimizer = true
optimizer_runs = 1000
via_ir = true

# CI 配置
[profile.ci]
fuzz = { runs = 1000 }
invariant = { runs = 500 }

# 网络端点（从环境变量读取）
[rpc_endpoints]
mainnet = "${MAINNET_RPC_URL}"
sepolia = "${SEPOLIA_RPC_URL}"
polygon = "${POLYGON_RPC_URL}"
localhost = "http://localhost:8545"

# Etherscan API 密钥
[etherscan]
mainnet = { key = "${ETHERSCAN_API_KEY}" }
sepolia = { key = "${ETHERSCAN_API_KEY}" }
polygon = { key = "${POLYGONSCAN_API_KEY}" }
```

### 配置优先级

Foundry 配置按以下优先级加载：

1. 命令行参数（最高优先级）
2. 环境变量
3. `foundry.toml`
4. 默认值（最低优先级）

## 依赖管理

### 安装依赖

Foundry 使用 git submodules 管理依赖：

```bash
# 安装 OpenZeppelin
forge install OpenZeppelin/openzeppelin-contracts --no-commit

# 安装特定版本
forge install OpenZeppelin/openzeppelin-contracts@v5.0.0 --no-commit

# 安装 Solmate（简洁的合约实现）
forge install transmissions11/solmate --no-commit

# 安装 Solady（汇编优化实现）
forge install Vectorized/solady --no-commit

# 安装 Uniswap V3
forge install Uniswap/v3-core --no-commit
forge install Uniswap/v3-periphery --no-commit
```

### 重映射（Remappings）

为了让 Solidity 正确找到依赖，需要配置重映射：

```toml
# foundry.toml
[profile.default]
remappings = [
    "@openzeppelin/contracts=lib/openzeppelin-contracts/contracts",
    "forge-std/=lib/forge-std/src/",
    "solmate/=lib/solmate/src/",
]
```

或创建 `remappings.txt`：

```
# remappings.txt
@openzeppelin/contracts=lib/openzeppelin-contracts/contracts
forge-std/=lib/forge-std/src/
solmate/=lib/solmate/src/
```

### 自动生成重映射

```bash
forge remappings > remappings.txt
```

### 更新依赖

```bash
# 更新所有依赖
forge update

# 更新特定依赖
forge update lib/openzeppelin-contracts

# 删除依赖
rm -rf lib/some-dependency
# 然后手动更新 .gitmodules
```

## 环境变量

创建 `.env` 文件：

```bash
# .env
# RPC URLs
MAINNET_RPC_URL=https://eth-mainnet.g.alchemy.com/v2/your-key
SEPOLIA_RPC_URL=https://eth-sepolia.g.alchemy.com/v2/your-key
POLYGON_RPC_URL=https://polygon-mainnet.g.alchemy.com/v2/your-key

# 私钥（切勿提交！）
PRIVATE_KEY=your_private_key_here

# Etherscan API
ETHERSCAN_API_KEY=your_key
POLYGONSCAN_API_KEY=your_key

# 其他配置
FOUNDRY_PROFILE=dev
```

加载环境变量：

```bash
# macOS/Linux
source .env

# 或使用 dotenv
# Foundry 会自动加载 .env 文件
```

## 编译合约

```bash
# 编译所有合约
forge build

# 清理编译产物
forge clean

# 重新编译
forge build --force

# 只编译特定合约
forge build src/SimpleToken.sol

# 输出 JSON 格式
forge build --extra-output abi,bin,bin-runtime

# 监视模式（文件变化时自动编译）
forge build --watch
```

编译产物在 `out/` 目录：

```
out/
├── SimpleToken.sol/
│   ├── SimpleToken.json         # 完整编译产物
│   └── SimpleToken.0.8.24.json  # 带版本的产物
└── build-info/
    └── 12345.json               # 编译元数据
```

## 常用命令

```bash
# 创建项目
forge init <project-name>

# 编译
forge build

# 测试
forge test

# 测试（详细输出）
forge test -vvvv

# 测试特定文件
forge test --match-path test/SimpleToken.t.sol

# 测试特定合约
forge test --match-contract SimpleTokenTest

# 测试特定测试函数
forge test --match-test testTransfer

# 运行模糊测试
forge test --fuzz

# Gas 快照
forge snapshot

# 格式化代码
forge fmt

# 验证合约
forge verify-contract <address> <contract> --watch

# 查看合约大小
forge build --sizes

# 生成 ABI
forge inspect SimpleToken abi

# 生成 bytecode
forge inspect SimpleToken bytecode

# 转换格式
cast --to-hex 100
cast --to-dec 0x64
cast --to-wei 1 ether

# 查看帮助
forge --help
cast --help
```

## 与 Hardhat 混合使用

Foundry 可以与 Hardhat 无缝集成：

```bash
# 在 Hardhat 项目中初始化 Foundry
forge install foundry-rs/forge-std --no-commit

# 创建 foundry.toml
cat > foundry.toml << EOF
[profile.default]
src = "contracts"
out = "out"
libs = ["node_modules", "lib"]
EOF
```

```toml
# foundry.toml（与 Hardhat 混合使用）
[profile.default]
src = "contracts"
out = "artifacts"
libs = ["node_modules", "lib"]
cache_path = "cache_hardhat"
```

## VSCode 集成

### 扩展配置

```json
// .vscode/settings.json
{
  "solidity.compileUsingRemoteVersion": "v0.8.24",
  "solidity.packageDefaultDependenciesContractsDirectory": "src",
  "solidity.packageDefaultDependenciesDirectory": ["lib", "node_modules"],
  "solidity.remappings": [
    "@openzeppelin/contracts=lib/openzeppelin-contracts/contracts",
    "forge-std/=lib/forge-std/src/"
  ],
  "solidity.formatter": "forge",
  "[solidity]": {
    "editor.defaultFormatter": "JuanBlanco.solidity",
    "editor.formatOnSave": true
  }
}
```

### 推荐扩展

1. **Solidity** (Juan Blanco)
2. **Even Better TOML** (tamasfe)
3. **crates** (Seray Uzgur) - 如果从源码编译

## Anvil 本地节点

Anvil 是 Foundry 提供的本地测试网络：

```bash
# 启动默认配置
anvil

# 指定端口
anvil --port 8546

# 指定链 ID
anvil --chain-id 31337

# 指定初始账户余额
anvil --balance 10000

# 分叉主网
anvil --fork-url $MAINNET_RPC_URL

# 分叉到特定区块
anvil --fork-url $MAINNET_RPC_URL --fork-block-number 15000000

# 输出账户
anvil --accounts 10

# 静默模式
anvil --silent
```

输出示例：

```
Available Accounts
==================
(0) 0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266 (10000 ETH)
(1) 0x70997970C51812dc3A010C7d01b50e0d17dc79C8 (10000 ETH)
...

Private Keys
==================
(0) 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
(1) 0x59c6995e998f97a5a0044966f0945389dc9e86dae88c7a8412f4603b6b78690d
...
```

## Chisel REPL

Chisel 是 Solidity 的交互式 REPL：

```bash
chisel
```

```solidity
➜ address(1).balance
Type: uint256
├ Hex: 0x00
└ Decimal: 0

➜ uint256(100) + 50
Type: uint256
├ Hex: 0x96
└ Decimal: 150

➜ keccak256(abi.encodePacked("hello"))
Type: bytes32
├ Hex: 0x1c8aff950685c2ed4bc3174f3472287b56d9517b9c948127319a09a7a36deac8

➜ !help  // 查看帮助
➜ !clear // 清除状态
➜ !quit  // 退出
```

## 故障排除

### 常见问题

1. **编译器版本问题**
```bash
# 安装特定版本的 solc
forge solc install 0.8.24

# 使用系统 solc
forge build --use 0.8.24
```

2. **依赖找不到**
```bash
# 检查重映射
forge remappings

# 手动添加
echo "@openzeppelin/=lib/openzeppelin-contracts/" >> remappings.txt
```

3. **权限问题（Windows）**
```powershell
# 允许脚本执行
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

4. **内存不足**
```bash
# 增加 Node.js 内存（如果使用 fork）
export NODE_OPTIONS="--max-old-space-size=8192"
```

## 与 Rust 的对比

作为 Rust 开发者，你会发现很多相似之处：

| 概念 | Rust | Foundry |
|------|------|---------|
| 包管理 | Cargo.toml | foundry.toml |
| 依赖 | Cargo.lock | .gitmodules |
| 测试 | #[test] | function test...() |
| 构建产物 | target/ | out/ |
| 文档 | cargo doc | forge doc |
| 格式化 | cargo fmt | forge fmt |
| 检查 | cargo check | forge build --check |

## 下一步

Foundry 环境搭建完成，继续学习 [Foundry 开发流程](./04-foundry-development.md)，体验使用 Solidity 编写测试的优雅方式。
