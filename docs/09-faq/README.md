# 附录：常见问题 FAQ

## 基础问题

### 1. 智能合约部署后可以修改吗？

**答**：智能合约部署后代码不可修改，但可以通过以下方式实现"升级"：

```solidity
// 方法1: 代理模式
contract Proxy {
    address public implementation;
    
    function upgrade(address newImplementation) external onlyOwner {
        implementation = newImplementation;
    }
    
    fallback() external payable {
        address impl = implementation;
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
}

// 方法2: 数据分离模式
// 逻辑合约 + 数据合约
```

### 2. 如何向没有 payable 函数的合约发送 ETH？

**答**：通过合约的 `receive()` 或 `fallback()` 函数：

```solidity
contract ReceiveETH {
    // 接收 ETH
    receive() external payable {
        // 当发送纯 ETH 时调用
    }
    
    // 回退函数
    fallback() external payable {
        // 当调用不存在的函数时调用
    }
    
    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
}

// 发送方式:
// 1. transfer (不推荐，Gas 限制 2300)
// payable(contractAddress).transfer(1 ether);

// 2. send (不推荐，Gas 限制 2300)
// bool success = payable(contractAddress).send(1 ether);

// 3. call (推荐)
// (bool success, ) = contractAddress.call{value: 1 ether}("");
// require(success, "Transfer failed");
```

### 3. `private` 变量真的私有吗？

**答**：不是。所有链上数据都是公开的，`private` 只是限制合约内访问：

```solidity
contract PrivateExample {
    uint256 private secretNumber = 123;
    // 任何人都可通过以下方式读取：
    // web3.eth.getStorageAt(contractAddress, 0)
}

// 读取私有变量的方法：
// 1. 使用 web3.eth.getStorageAt()
// 2. 使用 cast storage <address> --rpc-url <url>
// 3. 区块链浏览器查看存储槽
```

### 4. 如何在合约中生成随机数？

**答**：链上无法生成真正的随机数，常见方案：

```solidity
// ❌ 不安全的方式
function random() public view returns (uint256) {
    return uint256(keccak256(abi.encodePacked(
        block.timestamp,  // 可被矿工操控
        block.difficulty, // 可被矿工操控
        msg.sender
    )));
}

// ✓ 方案1: Chainlink VRF
import "@chainlink/contracts/src/v0.8/VRFConsumerBase.sol";

contract RandomNumber is VRFConsumerBase {
    bytes32 internal keyHash;
    uint256 internal fee;
    
    function getRandomNumber() public returns (bytes32 requestId) {
        return requestRandomness(keyHash, fee);
    }
    
    function fulfillRandomness(
        bytes32 requestId,
        uint256 randomness
    ) internal override {
        // 使用随机数
    }
}

// ✓ 方案2: Commit-Reveal 方案
contract CommitReveal {
    struct Commit {
        bytes32 commitment;
        uint256 revealBlock;
    }
    
    mapping(address => Commit) public commits;
    
    function commit(bytes32 commitment) external {
        commits[msg.sender] = Commit({
            commitment: commitment,
            revealBlock: block.number + 10
        });
    }
    
    function reveal(uint256 nonce, bytes32 data) external {
        Commit storage c = commits[msg.sender];
        require(block.number >= c.revealBlock, "Too early");
        require(
            keccak256(abi.encodePacked(nonce, data)) == c.commitment,
            "Invalid reveal"
        );
        // 使用 data 作为随机数来源
    }
}
```

## Gas 相关

### 5. 为什么我的交易 Gas 费这么高？

**答**：可能的原因和解决方案：

```
Gas 费高的原因:
├── 网络拥堵
│   └── 解决: 等待低峰期、使用 L2
│
├── 合约操作昂贵
│   ├── 存储写入 (20,000 gas)
│   ├── 循环操作
│   └── 解决: 优化代码、使用瞬态存储
│
├── Gas Price 设置过高
│   └── 解决: 使用 Gas 预测工具
│
└── 复杂合约调用
    └── 解决: 批量操作、优化调用路径
```

### 6. 如何优化合约 Gas 消耗？

**答**：常见优化技巧：

```solidity
// ❌ 低效写法
contract Inefficient {
    uint256 public count;
    string public name;
    
    function loop() public {
        for (uint256 i = 0; i < count; i++) {  // count 每次循环都读存储
            // ...
        }
    }
    
    function store(string memory _name) public {
        name = _name;  // 存储长字符串
    }
}

// ✓ 高效写法
contract Efficient {
    uint256 public count;
    string public name;
    
    function loop() public {
        uint256 _count = count;  // 缓存到内存
        for (uint256 i = 0; i < _count; ++i) {
            // ...
        }
    }
    
    function store(string calldata _name) public {  // 使用 calldata
        name = _name;
    }
}

// Gas 优化清单:
// 1. 使用 uint256 而非更小的整数类型
// 2. 缓存存储变量到内存
// 3. 使用 calldata 代替 memory
// 4. 打包状态变量到一个槽位
// 5. 使用事件代替存储历史数据
// 6. 使用映射代替数组查找
// 7. 使用瞬态存储 (EIP-1153)
// 8. 避免不必要的零初始化
```

## 安全问题

### 7. 什么是重入攻击？如何防止？

**答**：重入攻击是最常见的智能合约漏洞：

```solidity
// ❌ 有漏洞的代码
contract Vulnerable {
    mapping(address => uint256) public balances;
    
    function withdraw() public {
        uint256 amount = balances[msg.sender];
        
        // 先转账，后更新状态 = 危险！
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success);
        
        balances[msg.sender] = 0;  // 攻击者可重入，此时余额未清零
    }
}

// ✓ 安全的代码 (Checks-Effects-Interactions 模式)
contract Secure {
    mapping(address => uint256) public balances;
    bool private _locked;
    
    modifier nonReentrant() {
        require(!_locked, "Reentrancy detected");
        _locked = true;
        _;
        _locked = false;
    }
    
    function withdraw() public nonReentrant {
        uint256 amount = balances[msg.sender];
        require(amount > 0, "No balance");
        
        // 1. 先更新状态 (Effects)
        balances[msg.sender] = 0;
        
        // 2. 再进行外部调用 (Interactions)
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}

// ✓ 使用瞬态存储的重入锁 (Solidity 0.8.28+)
contract ModernSecure {
    mapping(address => uint256) public balances;
    bool transient _locked;
    
    modifier nonReentrant() {
        require(!_locked, "Reentrancy detected");
        _locked = true;
        _;
        _locked = false;  // Gas 成本仅 100 vs 存储 20,000+
    }
}
```

### 8. `tx.origin` 为什么不应该用于认证？

**答**：`tx.origin` 可被钓鱼攻击利用：

```solidity
// ❌ 不安全的代码
contract Wallet {
    address public owner;
    
    constructor() {
        owner = msg.sender;
    }
    
    function transfer(address to, uint256 amount) public {
        require(tx.origin == owner, "Not owner");  // 危险！
        payable(to).transfer(amount);
    }
}

// 攻击场景:
// 1. 攻击者部署恶意合约 Malicious
// 2. 诱骗 owner 调用 Malicious.attack()
// 3. Malicious.attack() 调用 Wallet.transfer()
// 4. 此时 tx.origin == owner，攻击成功

// ✓ 安全的代码
contract SafeWallet {
    address public owner;
    
    constructor() {
        owner = msg.sender;
    }
    
    function transfer(address to, uint256 amount) public {
        require(msg.sender == owner, "Not owner");  // 安全
        payable(to).transfer(amount);
    }
}
```

## 开发问题

### 9. `call`, `delegatecall`, `staticcall` 有什么区别？

**答**：

```solidity
// call - 普通调用
// - 独立的执行上下文
// - 使用被调用合约的存储
// - msg.sender 是调用者
// - msg.value 是发送的值

contract A {
    uint256 public value;
    
    function callB(address b) public {
        (bool success, ) = b.call(abi.encodeWithSignature("setValue(1)"));
        // B 的存储被修改，A 的存储不变
    }
}

// delegatecall - 委托调用
// - 使用调用者的执行上下文
// - 使用调用者的存储
// - msg.sender 是原始调用者
// - msg.value 是原始发送的值

contract Proxy {
    address public implementation;
    uint256 public value;  // 存储在 Proxy
    
    function delegateCall() public {
        (bool success, ) = implementation.delegatecall(
            abi.encodeWithSignature("setValue(1)")
        );
        // implementation 的代码执行，但修改的是 Proxy 的存储
    }
}

// staticcall - 静态调用
// - 只读调用
// - 不能修改状态
// - view/pure 函数使用此方式

contract Reader {
    function readValue(address a) public view returns (uint256) {
        (bool success, bytes memory data) = a.staticcall(
            abi.encodeWithSignature("getValue()")
        );
        return abi.decode(data, (uint256));
    }
}
```

### 10. 如何处理大整数计算溢出？

**答**：Solidity 0.8+ 内置溢出检查：

```solidity
// Solidity 0.8+ 自动溢出检查
contract SafeMath {
    function add(uint256 a, uint256 b) public pure returns (uint256) {
        return a + b;  // 溢出时自动 revert
    }
    
    // 如需跳过检查 (明确不会溢出时)
    function unsafeAdd(uint256 a, uint256 b) public pure returns (uint256) {
        unchecked {
            return a + b;  // 不检查溢出
        }
    }
}

// Solidity 0.7 及以下需要 SafeMath
// import "@openzeppelin/contracts/utils/math/SafeMath.sol";
```

### 11. 合约大小限制是多少？如何解决？

**答**：合约最大 24KB，解决方案：

```solidity
// 解决合约大小限制:

// 1. 使用代理模式
// 主合约变小，逻辑放在可升级的实现合约中

// 2. 分离逻辑到库
library TokenLogic {
    function transfer(
        mapping(address => uint256) storage balances,
        address to,
        uint256 amount
    ) external {
        balances[msg.sender] -= amount;
        balances[to] += amount;
    }
}

contract Token {
    using TokenLogic for mapping(address => uint256);
    mapping(address => uint256) public balances;
    
    function transfer(address to, uint256 amount) external {
        balances.transfer(to, amount);
    }
}

// 3. 使用 Diamond 模式 (EIP-2535)
// 将合约拆分为多个 Facet

// 4. 减少不必要的功能
// 删除未使用的函数和变量
```

## 部署问题

### 12. 如何验证合约代码？

**答**：使用 Etherscan 或 Sourcify：

```bash
# 使用 Hardhat
npx hardhat verify --network sepolia <CONTRACT_ADDRESS> <CONSTRUCTOR_ARGS>

# 使用 Foundry
forge verify-contract \
  --chain-id 11155111 \
  --num-of-optimizations 200 \
  --constructor-args $(cast abi-encode "constructor(uint256)" 100) \
  --etherscan-api-key $ETHERSCAN_API_KEY \
  <CONTRACT_ADDRESS> \
  src/MyContract.sol:MyContract

# 使用 Sourcify (去中心化验证)
forge verify-contract --verifier sourcify ...
```

### 13. 测试网 ETH 从哪里获取？

**答**：常用测试网水龙头：

| 测试网 | 水龙头地址 |
|--------|-----------|
| Sepolia | https://sepoliafaucet.com |
| Sepolia | https://www.alchemy.com/faucets/ethereum-sepolia |
| Sepolia | https://faucet.quicknode.com/ethereum/sepolia |
| Polygon Mumbai | https://faucet.polygon.technology |
| Arbitrum Sepolia | https://faucet.quicknode.com/arbitrum/sepolia |

```bash
# 快速获取测试 ETH
# 1. 连接 MetaMask 到测试网
# 2. 复制钱包地址
# 3. 访问水龙头网站
# 4. 粘贴地址，点击领取
```

## 工具问题

### 14. Hardhat 和 Foundry 怎么选择？

**答**：根据需求选择：

```
Hardhat 优势:
├── JavaScript/TypeScript 生态
├── 插件丰富
├── 调试友好
├── 适合前端团队
└── 企业级项目

Foundry 优势:
├── 编译速度快 10-100 倍
├── Solidity 编写测试
├── 内置模糊测试
├── 脚本功能强大
└── 适合纯 Solidity 开发

推荐:
├── 新项目: Foundry (趋势)
├── 前端团队: Hardhat
├── 混合使用: Hardhat + Foundry
└── 学习: 两者都学
```

### 15. viem 和 ethers.js 怎么选择？

**答**：

```
ethers.js:
├── 历史悠久，文档丰富
├── 社区支持好
├── API 友好
└── 适合初学者

viem:
├── 更轻量 (tree-shakable)
├── 更好的 TypeScript 支持
├── 性能更好
├── wagmi 默认使用
└── 适合现代前端项目

推荐:
├── 新项目: viem + wagmi
├── 需要稳定: ethers.js
└── 学习: 从 ethers.js 开始
```

## 更多问题？

如果以上 FAQ 没有解决你的问题：

1. 查看 [Solidity 官方文档](https://docs.soliditylang.org/)
2. 搜索 [Ethereum Stack Exchange](https://ethereum.stackexchange.com/)
3. 访问 [登链社区](https://learnblockchain.cn/)
4. 提交 GitHub Issue

## 下一步

[返回目录](README.md)
