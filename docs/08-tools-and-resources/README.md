# 附录：工具与资源

## 开发工具

### IDE 和编辑器

```
推荐 IDE:

VS Code 插件:
├── Solidity (Nomic Foundation)
├── Hardhat Solidity
├── Prettier Solidity
└── Error Lens

在线 IDE:
├── Remix - https://remix.ethereum.org
│   ├── 适合快速原型开发
│   ├── 内置编译、部署、调试
│   └── 无需本地环境
│
└── ChainIDE - https://chainide.com
    └── 多链支持
```

### 开发框架

```
Hardhat:
├── 官网: https://hardhat.org
├── 文档: https://hardhat.org/docs
├── GitHub: https://github.com/NomicFoundation/hardhat
└── 特点: JavaScript/TypeScript，插件丰富

Foundry:
├── 官网: https://getfoundry.sh
├── 文档: https://book.getfoundry.sh
├── GitHub: https://github.com/foundry-rs/foundry
└── 特点: Rust 编写，速度极快
```

### 测试和调试

```
测试工具:
├── Hardhat Test - 内置 Mocha
├── Foundry Test - Solidity 测试
├── Waffle - 另一种测试框架
└── Synpress - E2E 测试

调试工具:
├── Hardhat Console
├── Foundry Cast
├── Tenderly - https://tenderly.co
│   ├── 交易模拟
│   ├── Gas 分析
│   └── 调试器
└── Remix Debugger
```

## 安全工具

### 静态分析

```
Slither:
├── 安装: pip install slither-analyzer
├── 使用: slither .
├── 检测: 常见漏洞、代码优化
└── GitHub: https://github.com/crytic/slither

Mythril:
├── 安装: pip install mythril
├── 使用: myth analyze contract.sol
├── 检测: 安全漏洞
└── GitHub: https://github.com/ConsenSys/mythril

Securify2:
├── 在线: https://securify.chainsecurity.com
└── 检测: 安全属性违反
```

### 模糊测试

```
Foundry Fuzz:
├── 内置模糊测试
├── 使用: testFuzz_ 前缀
└── 示例: function testFuzz_Transfer(uint256 amount)

Echidna:
├── 安装: https://github.com/crytic/echidna
├── 使用: echidna-test contract.sol
└── 特点: 属性测试

Certora:
├── 官网: https://www.certora.com
├── 特点: 形式化验证
└── 使用: CVL 规范语言
```

### 审计资源

```
审计公司:
├── Trail of Bits - https://www.trailofbits.com
├── OpenZeppelin - https://openzeppelin.com/security-audits
├── Consensys Diligence - https://consensys.net/diligence
├── Certik - https://www.certik.com
├── PeckShield - https://peckshield.com
└── SlowMist - https://www.slowmist.com

审计报告:
├── GitHub Audit Reports
├── Rekt News - https://rekt.news
└── SlowMist Hacked - https://hacked.slowmist.io
```

## 基础设施

### 节点服务

```
Infura:
├── 官网: https://infura.io
├── 免费额度: 每天 100,000 请求
└── 支持链: ETH, Polygon, Arbitrum 等

Alchemy:
├── 官网: https://alchemy.com
├── 免费额度: 每月 300M 计算单位
└── 特点: 开发工具丰富

QuickNode:
├── 官网: https://quicknode.com
└── 特点: 高性能

Ankr:
├── 官网: https://ankr.com
└── 特点: 去中心化节点
```

### 数据索引

```
The Graph:
├── 官网: https://thegraph.com
├── 文档: https://thegraph.com/docs
└── 特点: 去中心化索引

Dune Analytics:
├── 官网: https://dune.com
├── 特点: SQL 查询
└── 用途: 数据可视化

Goldsky:
├── 官网: https://goldsky.com
└── 特点: 实时索引
```

### 存储

```
IPFS:
├── 官网: https://ipfs.io
├── 文档: https://docs.ipfs.io
└── 特点: 分布式存储

Arweave:
├── 官网: https://arweave.org
├── 特点: 永久存储
└── 用途: NFT 元数据

Filecoin:
├── 官网: https://filecoin.io
└── 特点: 激励存储

Pinata:
├── 官网: https://pinata.cloud
└── 特点: IPFS 托管服务
```

## 库和框架

### 智能合约库

```
OpenZeppelin:
├── 官网: https://openzeppelin.com
├── GitHub: https://github.com/OpenZeppelin/openzeppelin-contracts
├── npm: @openzeppelin/contracts
└── 提供: ERC 标准、安全工具、代理模式

Solmate:
├── GitHub: https://github.com/transmissions11/solmate
├── 特点: Gas 优化
└── 提供: 精简的 ERC 实现

Solady:
├── GitHub: https://github.com/Vectorized/solady
├── 特点: 极致 Gas 优化
└── 提供: 高效的合约实现
```

### 前端库

```
ethers.js:
├── 文档: https://docs.ethers.org
├── npm: ethers
└── 特点: 完整的以太坊交互库

viem:
├── 文档: https://viem.sh
├── npm: viem
└── 特点: 轻量、类型安全

wagmi:
├── 文档: https://wagmi.sh
├── npm: wagmi
└── 特点: React Hooks

web3.js:
├── 文档: https://web3js.readthedocs.io
├── npm: web3
└── 特点: 早期库，逐渐被 ethers 替代
```

### Rust 库

```
alloy:
├── GitHub: https://github.com/alloy-rs/alloy
├── 特点: 官方 Rust 库
└── 用途: 与以太坊交互

ethers-rs:
├── GitHub: https://github.com/gakonst/ethers-rs
├── 特点: Rust 以太坊库
└── 用途: 开发工具、索引器

Solana SDK:
├── 文档: https://docs.solana.com
└── 用途: Solana 开发

Substrate:
├── 文档: https://substrate.io
└── 用途: 构建自定义区块链
```

## 预言机

```
Chainlink:
├── 官网: https://chain.link
├── 文档: https://docs.chain.link
├── 提供: 价格数据、随机数、API 调用
└── 地址: https://docs.chain.link/data-feeds/price-feeds/addresses

Pyth Network:
├── 官网: https://pyth.network
└── 特点: 高频金融数据

UMA:
├── 官网: https://umaproject.org
└── 特点: 乐观预言机
```

## 区块链浏览器

```
主网:
├── Etherscan - https://etherscan.io (Ethereum)
├── Polygonscan - https://polygonscan.com (Polygon)
├── Arbiscan - https://arbiscan.io (Arbitrum)
├── Optimistic Etherscan - https://optimistic.etherscan.io
├── BscScan - https://bscscan.com (BSC)
└── Solscan - https://solscan.io (Solana)

测试网:
├── Sepolia Etherscan - https://sepolia.etherscan.io
└── 其他测试网浏览器同理
```

## 钱包

```
浏览器钱包:
├── MetaMask - https://metamask.io
│   ├── 最流行的钱包
│   └── 支持多链
│
├── Rabby - https://rabby.io
│   ├── 更安全的替代
│   └── 交易预览
│
└── Rainbow - https://rainbow.me
    └── 移动端钱包

硬件钱包:
├── Ledger - https://www.ledger.com
└── Trezor - https://trezor.io

多签钱包:
├── Gnosis Safe - https://safe.global
└── 特点: 企业级多签
```

## 学习资源

### 官方文档

```
以太坊:
├── ethereum.org - https://ethereum.org/zh/developers
├── Solidity 文档 - https://docs.soliditylang.org
├── EVM 操作码 - https://www.evm.codes
└── Yellow Paper - https://ethereum.github.io/yellowpaper

其他链:
├── Solana 文档 - https://docs.solana.com
├── Polygon 文档 - https://docs.polygon.technology
└── Arbitrum 文档 - https://docs.arbitrum.io
```

### 教程和课程

```
中文:
├── 登链社区 - https://learnblockchain.cn
├── WTF Academy - https://www.wtf.academy
│   ├── Solidity 入门
│   ├── ethers.js 教程
│   └── 免费开源
│
└── 深潮学院 - https://www.techflow.cn

英文:
├── CryptoZombies - https://cryptozombies.io
├── Alchemy University - https://www.alchemy.com/university
├── Buildspace - https://buildspace.so
└── LearnWeb3 DAO - https://learnweb3.io
```

### 社区

```
论坛:
├── Ethereum Stack Exchange - https://ethereum.stackexchange.com
├── Reddit r/ethereum - https://reddit.com/r/ethereum
└── Discourse 论坛

Discord:
├── Ethereum Discord
├── Uniswap Discord
├── Aave Discord
└── Foundry Discord

Twitter/X:
├── @ethereum
├── @VitalikButerin
├── @paradigm
└── 关注相关项目账号
```

### 新闻和资讯

```
新闻网站:
├── CoinDesk - https://coindesk.com
├── The Block - https://theblock.co
├── Decrypt - https://decrypt.co
└── 深潮 TechFlow - https://techflow.cn

播客:
├── Bankless - https://bankless.com
├── Unchained - https://unchainedpodcast.com
└── Epicenter - https://epicenter.tv

新闻简报:
├── Week in Ethereum News
├── The Defiant
└── Bankless Newsletter
```

## 实用工具网站

```
Gas 追踪:
├── Etherscan Gas Tracker - https://etherscan.io/gastracker
├── ETH Gas Station - https://ethgasstation.info
└── Gas Now - https://gasnow.org

地址工具:
├── Revoke.cash - 撤销授权
├── ENS - https://app.ens.domains
└── EtherScan - 地址查询

数据工具:
├── Dune Analytics - https://dune.com
├── Nansen - https://nansen.ai
├── DappRadar - https://dappradar.com
└── DeFiLlama - https://defillama.com

部署工具:
├── Remix - 快速部署
├── Hardhat Ignition - 部署管理
└── Foundry Script - 部署脚本

合约验证:
├── Etherscan Verify - 区块链浏览器验证
├── Sourcify - https://sourcify.dev
└── Hardhat Verify 插件
```

## 示例代码仓库

```
学习项目:
├── WTF Solidity - https://github.com/AmazingAng/WTF-Solidity
├── Solidity by Example - https://solidity-by-example.org
├── OpenZeppelin Learn - https://docs.openzeppelin.com/learn
└── Damn Vulnerable DeFi - https://www.damnvulnerabledefi.xyz

生产级代码:
├── Uniswap V3 - https://github.com/Uniswap/v3-core
├── Aave V3 - https://github.com/aave/aave-v3-core
├── Compound - https://github.com/compound-finance/compound-protocol
└── Lido - https://github.com/lidofinance/lido-dao
```

---

教程到此结束！希望这些资源对你的 Web3 学习之旅有所帮助。

如有问题，欢迎在社区中提问和交流。祝学习顺利！
