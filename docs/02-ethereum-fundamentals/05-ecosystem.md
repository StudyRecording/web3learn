# 2.5 以太坊生态系统

## 生态系统概览

```
┌─────────────────────────────────────────────────────────────────┐
│                     以太坊生态系统                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  DeFi (去中心化金融)                                            │
│  ├── DEX: Uniswap, Curve, Balancer                             │
│  ├── 借贷: Aave, Compound, Maker                               │
│  ├── 稳定币: DAI, USDC, USDT                                   │
│  └── 衍生品: dYdX, GMX, Synthetix                              │
│                                                                 │
│  NFT (非同质化代币)                                             │
│  ├── 市场: OpenSea, Blur, LooksRare                            │
│  ├── 艺术品: CryptoPunks, BAYC, Pudgy Penguins                 │
│  └── 基础设施: ENS, POAP, Mirror                               │
│                                                                 │
│  DAO (去中心化组织)                                             │
│  ├── 治理: Uniswap DAO, Aave DAO                               │
│  ├── 工具: Snapshot, Tally, Gnosis Safe                        │
│  └── 投资: BitDAO, ConstitutionDAO                             │
│                                                                 │
│  基础设施                                                       │
│  ├── 节点: Infura, Alchemy, QuickNode                          │
│  ├── 存储: IPFS, Arweave, Filecoin                             │
│  ├── 索引: The Graph, Dune Analytics                           │
│  └── 安全: OpenZeppelin, Trail of Bits                         │
│                                                                 │
│  Layer 2                                                        │
│  ├── Rollups: Arbitrum, Optimism, zkSync                       │
│  └── 侧链: Polygon                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## DeFi 协议

### Uniswap (DEX)

```
Uniswap - 去中心化交易所

核心功能:
├── 代币兑换
├── 流动性提供
├── 价格发现
└── 无需许可

版本:
├── V2: 简单 AMM
├── V3: 集中流动性
└── V4: Hooks (即将推出)

合约地址 (主网):
├── Router: 0x68b3465833fb72A70ecDF485E0e4C7bD8665Fc45
├── Factory: 0x1F98431c8aD98523631AE4a59f267346ea31F984
└── Quoter: 0xb27308f9F90D607463bb33eA1BeBb41C27CE5AB6
```

```solidity
// Uniswap V3 交互示例
interface IUniswapV3Router {
    function exactInputSingle(
        ExactInputSingleParams calldata params
    ) external payable returns (uint256 amountOut);
    
    struct ExactInputSingleParams {
        address tokenIn;
        address tokenOut;
        uint24 fee;
        address recipient;
        uint256 deadline;
        uint256 amountIn;
        uint256 amountOutMinimum;
        uint160 sqrtPriceLimitX96;
    }
}

// 使用示例
function swapToken(
    address tokenIn,
    address tokenOut,
    uint256 amountIn
) external {
    IERC20(tokenIn).approve(ROUTER, amountIn);
    
    IUniswapV3Router.ExactInputSingleParams memory params = 
        IUniswapV3Router.ExactInputSingleParams({
            tokenIn: tokenIn,
            tokenOut: tokenOut,
            fee: 3000,  // 0.3%
            recipient: msg.sender,
            deadline: block.timestamp,
            amountIn: amountIn,
            amountOutMinimum: 0,  // 实际使用时设置滑点保护
            sqrtPriceLimitX96: 0
        });
    
    router.exactInputSingle(params);
}
```

### Aave (借贷)

```
Aave - 去中心化借贷协议

核心功能:
├── 存款赚取利息
├── 抵押借贷
├── 闪电贷
└── 跨链借贷

支持资产:
├── ETH, WBTC
├── USDC, USDT, DAI
├── stETH, wstETH
└── 更多...

合约地址:
├── Pool: 0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2
└── Pool Addresses Provider: 0x2f39d218133EFaB8F2B819B1066c7E434Ad62E85
```

```solidity
// Aave 交互示例
interface IPool {
    function deposit(
        address asset,
        uint256 amount,
        address onBehalfOf,
        uint16 referralCode
    ) external;
    
    function borrow(
        address asset,
        uint256 amount,
        uint256 interestRateMode,
        uint16 referralCode,
        address onBehalfOf
    ) external;
    
    function repay(
        address asset,
        uint256 amount,
        uint256 interestRateMode,
        address onBehalfOf
    ) external returns (uint256);
}

// 存款
function depositToAave(address asset, uint256 amount) external {
    IERC20(asset).approve(POOL, amount);
    IPool(POOL).deposit(asset, amount, msg.sender, 0);
}

// 借款
function borrowFromAave(
    address asset, 
    uint256 amount
) external {
    IPool(POOL).borrow(
        asset,
        amount,
        2,  // 可变利率
        0,
        msg.sender
    );
}
```

### MakerDAO (稳定币)

```
MakerDAO - 去中心化稳定币 DAI

核心机制:
├── 抵押债仓 (CDP)
├── 超额抵押生成 DAI
├── 稳定费率
└── 清算机制

抵押品:
├── ETH (stETH)
├── WBTC
├── USDC
└── 更多...

关键合约:
├── DAI: 0x6B175474E89094C44Da98b954EesddfAE
├── VAT: 核心账本
└── JUG: 利率累加器
```

## NFT 生态

### OpenSea

```
OpenSea - NFT 市场

功能:
├── NFT 买卖
├── 拍卖
├── 集合创建
└── 跨链支持

协议:
├── Seaport (开源协议)
└── Wyvern (旧版)

合约:
└── Seaport: 0x00000000006c3852cbEf3e08E8dF289169EdE581
```

### ENS (以太坊域名服务)

```
ENS - 以太坊域名服务

功能:
├── .eth 域名注册
├── 地址解析
├── 存储文本记录
└── 子域名管理

示例:
├── vitalik.eth -> 0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045
├── myapp.eth -> 合约地址
└── ipfs://... -> 网站

合约:
├── ENS Registry: 0x00000000000C2E074eC69A0dFb2997BA6C7d2e1e
└── Public Resolver: 0x4976fb03C32e5B8cfe2b6cCB31c09Ba78EBaBa41
```

```solidity
// ENS 解析示例
interface ENS {
    function resolver(bytes32 node) external view returns (address);
}

interface Resolver {
    function addr(bytes32 node) external view returns (address);
    function text(bytes32 node, string calldata key) external view returns (string memory);
}

function resolveENS(string memory name) public view returns (address) {
    bytes32 node = keccak256(abi.encodePacked(
        keccak256(abi.encodePacked("eth")),
        keccak256(abi.encodePacked(name))
    ));
    
    address resolverAddress = ENS(ENS_REGISTRY).resolver(node);
    return Resolver(resolverAddress).addr(node);
}
```

## 基础设施

### The Graph (数据索引)

```
The Graph - 去中心化索引协议

用途:
├── 索引链上数据
├── GraphQL 查询
├── 实时数据
└── 去中心化

工作流程:
1. 定义 Subgraph (数据模式)
2. 部署到网络
3. 索引器同步数据
4. 通过 GraphQL 查询
```

```graphql
# Uniswap V3 Subgraph 查询示例
query {
  pools(
    where: { token0: "0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2" }
    orderBy: totalValueLockedUSD
    orderDirection: desc
    first: 10
  ) {
    id
    token0 {
      symbol
    }
    token1 {
      symbol
    }
    totalValueLockedUSD
    volumeUSD
  }
}
```

### OpenZeppelin (安全合约库)

```
OpenZeppelin - 安全合约库

提供:
├── ERC-20, ERC-721, ERC-1155 实现
├── 访问控制
├── 安全工具
├── 代理模式
└── 升级模式

使用:
bun add @openzeppelin/contracts
```

```solidity
// OpenZeppelin 使用示例
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/security/Pausable.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract MyToken is ERC20, Ownable, Pausable {
    constructor() ERC20("My Token", "MTK") {}
    
    function mint(address to, uint256 amount) public onlyOwner whenNotPaused {
        _mint(to, amount);
    }
    
    function pause() public onlyOwner {
        _pause();
    }
    
    function unpause() public onlyOwner {
        _unpause();
    }
}
```

### Chainlink (预言机)

```
Chainlink - 去中心化预言机

提供:
├── 价格数据
├── 随机数
├── 外部 API 调用
└── 自动化执行

价格预言机:
├── ETH/USD
├── BTC/USD
└── 更多交易对...
```

```solidity
// Chainlink 价格数据示例
import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

contract PriceConsumer {
    AggregatorV3Interface internal priceFeed;
    
    constructor() {
        // ETH/USD 价格预言机
        priceFeed = AggregatorV3Interface(
            0x5f4eC3Df9cbd43714FE2740f5E3616155c5b8419
        );
    }
    
    function getLatestPrice() public view returns (int256) {
        (
            uint80 roundId,
            int256 price,
            uint256 startedAt,
            uint256 updatedAt,
            uint80 answeredInRound
        ) = priceFeed.latestRoundData();
        
        return price;  // 返回 ETH 价格 (8位小数)
    }
}
```

## 开发工具

### 开发框架

```
开发框架对比

Hardhat:
├── JavaScript/TypeScript
├── 插件丰富
├── 调试友好
└── 社区活跃

Foundry:
├── Rust 编写，速度快
├── Solidity 测试
├── 内置模糊测试
└── 现代开发者首选
```

### 分析工具

```
Dune Analytics:
├── SQL 查询链上数据
├── 可视化仪表板
└── 社区共享查询

Tenderly:
├── 交易模拟
├── Gas 分析
├── 调试器
└── 告警

DefiLlama:
├── TVL 数据
├── 协议分析
└── 跨链数据
```

## 安全资源

### 审计公司

```
知名审计公司:
├── Trail of Bits
├── OpenZeppelin
├── Consensys Diligence
├── Certik
├── PeckShield
└── SlowMist
```

### 安全工具

```
静态分析:
├── Slither
├── Mythril
└── Securify2

模糊测试:
├── Foundry Fuzz
├── Echidna
└── Harvey

形式化验证:
├── Certora
└── KEVM
```

### 漏洞资源

```
学习资源:
├── Rekt News (攻击案例)
├── SlowMist Hacked Archive
├── DeFiHackLabs
└── Damn Vulnerable DeFi (练习)
```

## 学习资源

### 官方文档

```
以太坊官方:
├── ethereum.org
├── Solidity 文档
├── OpenZeppelin 文档
└── Foundry Book
```

### 社区资源

```
中文社区:
├── 登链社区 (learnblockchain.cn)
├── WTF Academy
├── 深潮 TechFlow
└── 链闻

英文社区:
├── Ethereum Stack Exchange
├── Reddit r/ethereum
├── Twitter Crypto Twitter
└── Discord servers
```

### 实战项目

```
推荐学习项目:
├── Uniswap V2/V3 (DEX)
├── Aave (借贷)
├── Compound (借贷)
├── ENS (域名)
└── Gnosis Safe (多签钱包)
```

## 下一步

恭喜完成第二章！

现在你已经掌握了以太坊的基础知识，接下来让我们学习智能合约开发：

[第三章：Solidity 智能合约开发 →](../03-solidity/README.md)
