# 安全实践

> 智能合约安全至关重要，一次漏洞可能导致巨额损失

## 安全原则

```
┌─────────────────────────────────────────────────────────────┐
│                   智能合约安全原则                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 默认拒绝                                                │
│     所有调用都是恶意的，先拒绝再允许                        │
│                                                             │
│  2. 最小权限                                                │
│     只给必要的权限，不多给                                   │
│                                                             │
│  3. 失败安全                                                │
│     失败时回滚，不留部分状态                                │
│                                                             │
│  4. 简单设计                                                │
│     越简单越安全，复杂度带来漏洞                            │
│                                                             │
│  5. 充分测试                                                │
│     单元测试、集成测试、审计                                │
│                                                             │
│  Java 对比:                                                 │
│  Java bug → 服务重启修复                                   │
│  Solidity bug → 永久损失，无法修复                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 重入攻击 (Reentrancy)

### 攻击原理

```
┌─────────────────────────────────────────────────────────────┐
│                    重入攻击流程                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  正常流程:                                                  │
│  用户 ──► withdraw() ──► 扣余额 ──► 转账 ──► 完成          │
│                                                             │
│  攻击流程:                                                  │
│  1. 攻击者调用 withdraw()                                   │
│  2. 合约扣减余额                                            │
│  3. 合约发送 ETH 到攻击者                                   │
│  4. 攻击者的 fallback() 被触发                              │
│  5. fallback() 再次调用 withdraw()                         │
│  6. 余额已扣但未更新，再次提款                              │
│  7. 循环直到 Gas 耗尽或余额清空                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 漏洞代码

```solidity
// ❌ 有漏洞的代码
contract VulnerableBank {
    mapping(address => uint256) public balances;
    
    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }
    
    function withdraw() public {
        uint256 amount = balances[msg.sender];
        
        // 先转账，后更新状态 - 危险！
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
        
        balances[msg.sender] = 0;  // 状态更新太晚
    }
    
    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
}
```

### 攻击合约

```solidity
contract Attacker {
    VulnerableBank public bank;
    
    constructor(address _bank) {
        bank = VulnerableBank(_bank);
    }
    
    function attack() external payable {
        bank.deposit{value: msg.value}();
        bank.withdraw();
    }
    
    receive() external payable {
        if (bank.getBalance() >= 1 ether) {
            bank.withdraw();  // 重入攻击
        }
    }
}
```

### 修复方案

```solidity
// ✅ 方案 1: 检查-生效-交互模式 (CEI)
contract SafeBankCEI {
    mapping(address => uint256) public balances;
    
    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }
    
    function withdraw() public {
        // 1. 检查
        uint256 amount = balances[msg.sender];
        require(amount > 0, "No balance");
        
        // 2. 生效 - 先更新状态
        balances[msg.sender] = 0;
        
        // 3. 交互 - 最后外部调用
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}

// ✅ 方案 2: 重入锁
contract SafeBankMutex {
    mapping(address => uint256) public balances;
    bool private locked;
    
    modifier noReentrancy() {
        require(!locked, "Reentrant call");
        locked = true;
        _;
        locked = false;
    }
    
    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }
    
    function withdraw() public noReentrancy {
        uint256 amount = balances[msg.sender];
        require(amount > 0, "No balance");
        
        balances[msg.sender] = 0;
        
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}

// ✅ 方案 3: OpenZeppelin ReentrancyGuard
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract SafeBankOZ is ReentrancyGuard {
    mapping(address => uint256) public balances;
    
    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }
    
    function withdraw() public nonReentrant {
        uint256 amount = balances[msg.sender];
        require(amount > 0, "No balance");
        
        balances[msg.sender] = 0;
        
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

## 整数溢出/下溢

### 0.8.0 前的问题

```solidity
// ❌ Solidity < 0.8.0 有溢出问题
contract OverflowOld {
    uint256 public count = 0;
    
    function increment() public {
        count += 1;  // 可能溢出回绕
    }
    
    function decrement() public {
        count -= 1;  // 可能下溢到极大值
    }
}

// 需要使用 SafeMath
library SafeMath {
    function add(uint256 a, uint256 b) internal pure returns (uint256) {
        uint256 c = a + b;
        require(c >= a, "Overflow");
        return c;
    }
    
    function sub(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b <= a, "Underflow");
        return a - b;
    }
}
```

### 0.8.0+ 的改进

```solidity
// ✅ Solidity >= 0.8.0 自动检查溢出
contract OverflowNew {
    uint256 public count = 0;
    
    function increment() public {
        count += 1;  // 自动检查溢出
    }
    
    function decrement() public {
        count -= 1;  // 自动检查下溢，会 revert
    }
    
    // 使用 unchecked 绕过检查 (确定安全时)
    function unsafeIncrement() public {
        unchecked {
            count += 1;  // 不检查溢出
        }
    }
}
```

## 访问控制漏洞

### 常见问题

```solidity
// ❌ 缺少访问控制
contract VulnerableAccess {
    address public owner;
    uint256 public sensitiveData;
    
    // 漏洞: 没有 onlyOwner
    function setOwner(address _newOwner) public {
        owner = _newOwner;
    }
    
    // 漏洞: 没有权限检查
    function setSensitiveData(uint256 _data) public {
        sensitiveData = _data;
    }
}

// ✅ 正确的访问控制
contract SafeAccess {
    address public owner;
    uint256 public sensitiveData;
    
    event OwnerChanged(address oldOwner, address newOwner);
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    constructor() {
        owner = msg.sender;
    }
    
    function setOwner(address _newOwner) public onlyOwner {
        require(_newOwner != address(0), "Zero address");
        emit OwnerChanged(owner, _newOwner);
        owner = _newOwner;
    }
    
    function setSensitiveData(uint256 _data) public onlyOwner {
        sensitiveData = _data;
    }
}
```

### tx.origin 陷阱

```solidity
// ❌ 不要用 tx.origin 做权限检查
contract VulnerableTxOrigin {
    address public owner;
    
    constructor() {
        owner = msg.sender;
    }
    
    function transfer(address _to, uint256 _amount) public {
        require(tx.origin == owner, "Not owner");  // 危险！
        // 转账逻辑
    }
}

// 攻击原理:
// 用户调用恶意合约 -> 恶意合约调用 VulnerableTxOrigin
// tx.origin 仍然是用户，绕过检查

// ✅ 使用 msg.sender
contract SafeTxOrigin {
    address public owner;
    
    constructor() {
        owner = msg.sender;
    }
    
    function transfer(address _to, uint256 _amount) public {
        require(msg.sender == owner, "Not owner");  // 安全
        // 转账逻辑
    }
}
```

## 前端运行 (Front-Running)

### 攻击场景

```
┌─────────────────────────────────────────────────────────────┐
│                   Front-Running 攻击                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  场景: 去中心化交易所                                       │
│                                                             │
│  1. 用户提交购买代币的大额交易                              │
│     │                                                       │
│     ▼                                                       │
│  2. 攻击者看到交易池中的待处理交易                          │
│     │                                                       │
│     ▼                                                       │
│  3. 攻击者提交相同交易，但设置更高 Gas 价格                  │
│     │                                                       │
│     ▼                                                       │
│  4. 矿工优先打包高 Gas 交易                                 │
│     │                                                       │
│     ▼                                                       │
│  5. 攻击者先买入，价格上涨                                  │
│     │                                                       │
│     ▼                                                       │
│  6. 用户交易执行，价格更高                                  │
│     │                                                       │
│     ▼                                                       │
│  7. 攻击者卖出获利                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 防护措施

```solidity
// 承诺-揭示方案
contract CommitReveal {
    struct Bid {
        bytes32 commitment;
        uint256 value;
        bool revealed;
    }
    
    mapping(address => Bid) public bids;
    
    uint256 public commitEndTime;
    uint256 public revealEndTime;
    
    event Committed(address bidder);
    event Revealed(address bidder, uint256 value);
    
    constructor(uint256 commitDuration, uint256 revealDuration) {
        commitEndTime = block.timestamp + commitDuration;
        revealEndTime = commitEndTime + revealDuration;
    }
    
    // 第一步: 提交承诺
    function commit(bytes32 _commitment) public {
        require(block.timestamp < commitEndTime, "Commit period ended");
        require(bids[msg.sender].commitment == bytes32(0), "Already committed");
        
        bids[msg.sender].commitment = _commitment;
        emit Committed(msg.sender);
    }
    
    // 第二步: 揭示真实值
    function reveal(uint256 _value, bytes32 _secret) public {
        require(block.timestamp >= commitEndTime, "Commit period not ended");
        require(block.timestamp < revealEndTime, "Reveal period ended");
        
        Bid storage bid = bids[msg.sender];
        require(!bid.revealed, "Already revealed");
        
        // 验证承诺
        bytes32 commitment = keccak256(abi.encodePacked(_value, _secret));
        require(commitment == bid.commitment, "Invalid commitment");
        
        bid.value = _value;
        bid.revealed = true;
        
        emit Revealed(msg.sender, _value);
    }
    
    // 辅助函数: 生成承诺
    function generateCommitment(uint256 _value, bytes32 _secret) 
        public pure returns (bytes32) 
    {
        return keccak256(abi.encodePacked(_value, _secret));
    }
}
```

## 拒绝服务 (DoS)

### 常见 DoS 模式

```solidity
// ❌ DoS 漏洞 1: 循环无限制
contract VulnerableLoop {
    address[] public users;
    
    function distributeRewards() public {
        for (uint256 i = 0; i < users.length; i++) {
            // 如果用户太多，会超出 Gas 限制
            payable(users[i]).transfer(1 ether);
        }
    }
}

// ✅ 修复: 分页处理
contract SafeLoop {
    address[] public users;
    
    function distributeRewards(uint256 start, uint256 count) public {
        uint256 end = start + count;
        if (end > users.length) {
            end = users.length;
        }
        
        for (uint256 i = start; i < end; i++) {
            payable(users[i]).transfer(1 ether);
        }
    }
}

// ❌ DoS 漏洞 2: 外部调用失败
contract VulnerableExternalCall {
    mapping(address => uint256) public balances;
    
    function withdrawAll() public {
        // 如果某个用户转账失败，整个函数回滚
        for (address user in getUsers()) {
            payable(user).transfer(balances[user]);
        }
    }
}

// ✅ 修复: 推拉模式
contract SafePullPayment {
    mapping(address => uint256) public balances;
    
    function withdraw() public {
        uint256 amount = balances[msg.sender];
        require(amount > 0, "No balance");
        
        balances[msg.sender] = 0;
        
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

## 整数除法精度

```solidity
// ❌ 精度丢失
contract PrecisionLoss {
    function badDivision(uint256 a, uint256 b) public pure returns (uint256) {
        return (a / b) * b;  // 可能不等于 a
    }
    
    // 例: 7 / 3 = 2, 2 * 3 = 6 ≠ 7
}

// ✅ 正确处理
contract PrecisionSafe {
    uint256 constant DECIMALS = 18;
    
    function safeDivision(uint256 a, uint256 b) public pure returns (uint256) {
        return (a * 10 ** DECIMALS) / b;  // 保持精度
    }
    
    function calculatePercentage(uint256 amount, uint256 percentage) 
        public pure returns (uint256) 
    {
        // percentage 是 100 为基准，如 50 = 50%
        return (amount * percentage) / 100;
    }
}
```

## 未检查的低级调用

```solidity
// ❌ 未检查返回值
contract VulnerableCall {
    function callWithoutCheck(address target, bytes memory data) public {
        target.call(data);  // 返回值被忽略
    }
    
    function delegateWithoutCheck(address target, bytes memory data) public {
        target.delegatecall(data);  // 返回值被忽略
    }
}

// ✅ 正确检查
contract SafeCall {
    function safeCall(address target, bytes memory data) public returns (bool, bytes memory) {
        (bool success, bytes memory result) = target.call(data);
        require(success, "Call failed");
        return (success, result);
    }
    
    function safeDelegateCall(address target, bytes memory data) public returns (bytes memory) {
        (bool success, bytes memory result) = target.delegatecall(data);
        require(success, "Delegatecall failed");
        return result;
    }
}
```

## 安全检查清单

```
┌─────────────────────────────────────────────────────────────┐
│                   安全检查清单                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [ ] 重入攻击                                               │
│      ├── 使用 CEI 模式                                      │
│      ├── 使用 ReentrancyGuard                               │
│      └── 外部调用放在最后                                   │
│                                                             │
│  [ ] 访问控制                                               │
│      ├── 所有关键函数加权限检查                             │
│      ├── 不使用 tx.origin                                   │
│      └── 使用 onlyOwner/onlyRole                            │
│                                                             │
│  [ ] 整数安全                                               │
│      ├── 使用 Solidity 0.8.0+                               │
│      ├── 谨慎使用 unchecked                                 │
│      └── 注意除法精度                                       │
│                                                             │
│  [ ] 外部调用                                               │
│      ├── 检查返回值                                         │
│      ├── 限制 Gas                                           │
│      └── 处理失败情况                                       │
│                                                             │
│  [ ] DoS 防护                                               │
│      ├── 限制循环次数                                       │
│      ├── 使用推拉模式                                       │
│      └── 分页处理                                           │
│                                                             │
│  [ ] 前端运行                                               │
│      ├── 使用承诺-揭示                                      │
│      ├── 添加时间锁                                         │
│      └── 考虑滑点保护                                       │
│                                                             │
│  [ ] 其他                                                   │
│      ├── 验证所有输入                                       │
│      ├── 使用最新编译器                                     │
│      └── 进行代码审计                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## OpenZeppelin 安全库

```solidity
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/security/Pausable.sol";
import "@openzeppelin/contracts/utils/Address.sol";

contract SecureContract is ReentrancyGuard, Ownable, Pausable {
    using Address for address;
    
    mapping(address => uint256) public balances;
    
    event Deposit(address indexed user, uint256 amount);
    event Withdrawal(address indexed user, uint256 amount);
    
    function deposit() public payable whenNotPaused {
        require(msg.value > 0, "Zero amount");
        balances[msg.sender] += msg.value;
        emit Deposit(msg.sender, msg.value);
    }
    
    function withdraw(uint256 amount) 
        public 
        nonReentrant 
        whenNotPaused 
    {
        require(amount > 0, "Zero amount");
        require(balances[msg.sender] >= amount, "Insufficient balance");
        
        balances[msg.sender] -= amount;
        
        // 使用 OpenZeppelin 的 Address 库
        payable(msg.sender).sendValue(amount);
        
        emit Withdrawal(msg.sender, amount);
    }
    
    function pause() public onlyOwner {
        _pause();
    }
    
    function unpause() public onlyOwner {
        _unpause();
    }
    
    function emergencyWithdraw() public onlyOwner {
        uint256 balance = address(this).balance;
        (bool success, ) = owner().call{value: balance}("");
        require(success, "Transfer failed");
    }
}
```

## 历史著名漏洞案例

```
┌─────────────────────────────────────────────────────────────┐
│                   历史著名漏洞                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  The DAO (2016) - 重入攻击                                  │
│  ├── 损失: 360万 ETH (~$50M)                               │
│  ├── 原因: 重入漏洞                                         │
│  └── 结果: 以太坊硬分叉                                    │
│                                                             │
│  Parity Multisig (2017) - 权限漏洞                          │
│  ├── 损失: 15万 ETH 冻结                                   │
│  ├── 原因: 未初始化的合约被接管                             │
│  └── 结果: 资金永久锁定                                    │
│                                                             │
│  BEC Token (2018) - 整数溢出                                │
│  ├── 损失: 市值归零                                         │
│  ├── 原因: 批量转账溢出                                     │
│  └── 结果: 代币价值崩盘                                    │
│                                                             │
│  Uniswap V1 (2019) - 重入攻击                               │
│  ├── 损失: ~$300K                                          │
│  ├── 原因: ETH-ERC20 代币对重入                             │
│  └── 结果: V2 修复                                         │
│                                                             │
│  学习教训:                                                  │
│  1. 所有合约都可能有漏洞                                    │
│  2. 必须进行安全审计                                        │
│  3. 使用经过验证的库                                        │
│  4. 设置紧急暂停机制                                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

[下一节：代币标准 →](06-standards.md)
