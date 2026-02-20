# 合约结构

> 理解智能合约的组成要素

## 合约组成

```
┌─────────────────────────────────────────────────────────────┐
│                    合约结构概览                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  contract MyContract {                                      │
│      ├── SPDX 许可声明                                      │
│      ├── pragma 版本指定                                    │
│      ├── import 导入                                        │
│      │                                                     │
│      ├── 状态变量 (State Variables)                        │
│      │   └── 持久化存储的数据                              │
│      │                                                     │
│      ├── 事件 (Events)                                     │
│      │   └── 日志记录，前端监听                            │
│      │                                                     │
│      ├── 修饰器 (Modifiers)                                │
│      │   └── 函数执行前/后的通用逻辑                       │
│      │                                                     │
│      ├── 构造函数 (Constructor)                            │
│      │   └── 部署时执行一次                                │
│      │                                                     │
│      ├── 函数 (Functions)                                  │
│      │   └── 业务逻辑                                      │
│      │                                                     │
│      └── 结构体/枚举 (Structs/Enums)                       │
│          └── 自定义类型                                    │
│  }                                                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 基础结构

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract Token {
    string public name;
    string public symbol;
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    constructor(string memory _name, string memory _symbol, uint256 _totalSupply) {
        name = _name;
        symbol = _symbol;
        totalSupply = _totalSupply;
        balanceOf[msg.sender] = _totalSupply;
    }
    
    function transfer(address to, uint256 amount) public returns (bool) {
        require(balanceOf[msg.sender] >= amount, "Insufficient balance");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }
}
```

### Java 对比

```
┌─────────────────────────────────────────────────────────────┐
│                 合约 vs Java 类                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Solidity                   Java                            │
│  ─────────                  ─────────                       │
│  contract Token             class Token                     │
│  状态变量                    成员变量 (数据库映射)           │
│  mapping                    Map<K,V>                        │
│  event                      (无，需日志框架)                │
│  modifier                   AOP / 装饰器                    │
│  constructor                constructor                     │
│  function                   method                          │
│  payable                    (无直接对应)                    │
│                                                             │
│  关键差异:                                                  │
│  1. 状态变量永久存储在链上                                  │
│  2. event 可被前端实时监听                                  │
│  3. modifier 在函数前后执行                                 │
│  4. payable 处理 ETH 转账                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 状态变量

### 变量可见性

```solidity
contract StateVariables {
    uint256 public publicVar = 1;       // 自动生成 getter
    uint256 private privateVar = 2;     // 仅本合约
    uint256 internal internalVar = 3;   // 本合约 + 子合约
    uint256 internal defaultVar = 4;    // 默认 internal
}
```

### 变量声明位置

```solidity
contract VariableLocations {
    uint256 stateVar;           // 状态变量: storage
    
    function example(uint256 param) public pure {
        uint256 localVar;       // 局部变量: 栈或内存
        
        string memory str;      // 显式指定 memory
        uint256[] memory arr;   // 引用类型必须指定位置
    }
}
```

### 初始值

```solidity
contract InitialValues {
    uint256 public defaultUint;     // 0
    int256 public defaultInt;       // 0
    bool public defaultBool;        // false
    address public defaultAddr;     // 0x0000000000000000000000000000000000000000
    bytes32 public defaultBytes;    // 0x000...
    
    enum Status { A, B, C }
    Status public defaultEnum;      // A (第一个)
    
    uint256[] public defaultArray;  // 空数组 []
    mapping(address => uint256) public defaultMap;  // 空映射
    
    function checkInitial() public view returns (bool) {
        return (defaultUint == 0 && !defaultBool && defaultAddr == address(0));
    }
}
```

## 函数

### 函数完整语法

```solidity
function <name>(<parameters>) <visibility> [modifiers] [state] [payable] 
    returns (<return types>) {
    // body
}
```

### 函数重载

```solidity
contract Overloading {
    function add(uint256 a, uint256 b) public pure returns (uint256) {
        return a + b;
    }
    
    function add(uint256 a, uint256 b, uint256 c) public pure returns (uint256) {
        return a + b + c;
    }
    
    // Java 也支持重载，用法相同
}
```

### 接收函数

```solidity
contract ReceiveTypes {
    // 普通函数调用
    function normalCall() public pure returns (string memory) {
        return "Normal function call";
    }
    
    // 接收 ETH 的函数
    function receiveEther() public payable returns (string memory) {
        return "Received via function";
    }
    
    // 部署时调用
    constructor() payable {
        // 可接收初始资金
    }
}
```

## 事件 (Events)

### 事件的作用

```
┌─────────────────────────────────────────────────────────────┐
│                      事件的作用                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 日志记录                                                 │
│     永久存储在区块链日志中                                   │
│     比 storage 更便宜                                        │
│                                                             │
│  2. 前端监听                                                 │
│     DApp 实时响应合约状态变化                                │
│     类似: WebSocket 推送                                     │
│                                                             │
│  3. 索引查询                                                 │
│     indexed 参数可被高效过滤                                 │
│     最多 3 个 indexed 参数                                   │
│                                                             │
│  Java 对比:                                                 │
│  event ≈ log4j/slf4j + WebSocket                          │
│  emit ≈ logger.info() + push                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 事件语法

```solidity
contract EventExample {
    event Transfer(
        address indexed from,
        address indexed to,
        uint256 value
    );
    
    event Approval(
        address indexed owner,
        address indexed spender,
        uint256 value
    );
    
    event Deposit(
        address indexed account,
        uint256 amount,
        uint256 timestamp
    );
    
    mapping(address => uint256) public balances;
    
    function deposit() public payable {
        balances[msg.sender] += msg.value;
        
        emit Deposit(msg.sender, msg.value, block.timestamp);
    }
    
    function transfer(address to, uint256 amount) public {
        require(balances[msg.sender] >= amount, "Insufficient");
        
        balances[msg.sender] -= amount;
        balances[to] += amount;
        
        emit Transfer(msg.sender, to, amount);
    }
}
```

### indexed 的作用

```solidity
contract IndexedExample {
    event MultiParam(
        address indexed a,    // 可过滤
        address indexed b,    // 可过滤
        address indexed c,    // 可过滤
        uint256 d,            // 不可过滤
        string e              // 不可过滤
    );
    
    // 前端查询示例 (ethers.js):
    // filter = contract.filters.MultiParam(userAddress, null, null)
    // events = await contract.queryFilter(filter)
}
```

## 修饰器 (Modifiers)

### 修饰器原理

```solidity
contract ModifierBasics {
    address public owner;
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;  // 继续执行函数体
    }
    
    constructor() {
        owner = msg.sender;
    }
    
    function sensitiveAction() public onlyOwner {
        // 只有 owner 能调用
    }
}
```

### 执行流程

```
┌─────────────────────────────────────────────────────────────┐
│                   修饰器执行流程                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  function sensitiveAction() public onlyOwner {              │
│      // 函数体                                              │
│  }                                                         │
│                                                             │
│  调用时执行顺序:                                            │
│                                                             │
│  1. 进入 modifier onlyOwner                                 │
│     │                                                      │
│     ▼                                                      │
│  2. require(msg.sender == owner)                           │
│     │                                                      │
│     ├── 失败 ──► revert("Not owner")                       │
│     │                                                      │
│     └── 通过 ──► 继续                                      │
│                 │                                          │
│                 ▼                                          │
│  3. _;  (跳转到函数体)                                     │
│     │                                                      │
│     ▼                                                      │
│  4. 执行函数体                                              │
│     │                                                      │
│     ▼                                                      │
│  5. 返回 (可继续执行 modifier _; 后的代码)                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 修饰器进阶

```solidity
contract AdvancedModifiers {
    address public owner;
    uint256 public pauseTime;
    bool public paused;
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    modifier whenNotPaused() {
        require(!paused, "Contract paused");
        _;
    }
    
    modifier whenPaused() {
        require(paused, "Contract not paused");
        _;
    }
    
    modifier costs(uint256 amount) {
        require(msg.value >= amount, "Insufficient payment");
        _;
    }
    
    modifier timedPause(uint256 duration) {
        require(block.timestamp >= pauseTime + duration, "Still paused");
        _;
    }
    
    modifier lock() {
        // 重入锁模式
        require(!locked, "Reentrant call");
        locked = true;
        _;
        locked = false;
    }
    
    bool private locked;
    
    constructor() {
        owner = msg.sender;
    }
    
    function pause() public onlyOwner {
        paused = true;
        pauseTime = block.timestamp;
    }
    
    function unpause() public onlyOwner whenPaused {
        paused = false;
    }
    
    function normalOperation() public whenNotPaused {
        // 正常操作
    }
    
    function premiumFeature() public payable costs(1 ether) whenNotPaused {
        // 需要 1 ETH
    }
    
    function safeWithdraw() public lock {
        // 防重入
    }
}
```

### 多修饰器

```solidity
contract MultipleModifiers {
    address public owner;
    bool public paused;
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    modifier whenNotPaused() {
        require(!paused, "Paused");
        _;
    }
    
    modifier validateAddress(address addr) {
        require(addr != address(0), "Invalid address");
        _;
    }
    
    // 修饰器按顺序执行
    function transferOwnership(address newOwner) 
        public 
        onlyOwner 
        whenNotPaused 
        validateAddress(newOwner) 
    {
        owner = newOwner;
    }
}
```

## 构造函数

```solidity
contract ConstructorExample {
    address public owner;
    uint256 public createdAt;
    string public name;
    
    constructor(string memory _name) {
        owner = msg.sender;
        createdAt = block.timestamp;
        name = _name;
    }
    
    // 只在部署时调用一次
    // 不能被外部调用
    // 可以为 payable
}
```

### Java 对比

```solidity
// Solidity
constructor(string memory _name) {
    owner = msg.sender;
    name = _name;
}
```

```java
// Java
public class Contract {
    private Address owner;
    private String name;
    
    public Contract(String name) {
        this.owner = getCurrentCaller();  // 需要额外获取
        this.name = name;
    }
}
```

## 结构体

```solidity
contract StructExample {
    struct User {
        string name;
        address wallet;
        uint256 balance;
        uint256 joinedAt;
        bool isActive;
    }
    
    struct Order {
        uint256 id;
        address buyer;
        uint256 amount;
        Status status;
    }
    
    enum Status { Pending, Shipped, Delivered, Cancelled }
    
    User public admin;
    mapping(address => User) public users;
    Order[] public orders;
    
    constructor() {
        admin = User({
            name: "Admin",
            wallet: msg.sender,
            balance: 0,
            joinedAt: block.timestamp,
            isActive: true
        });
    }
    
    function register(string memory _name) public {
        require(users[msg.sender].wallet == address(0), "Already registered");
        
        users[msg.sender] = User({
            name: _name,
            wallet: msg.sender,
            balance: 0,
            joinedAt: block.timestamp,
            isActive: true
        });
    }
    
    function createOrder(uint256 _amount) public {
        orders.push(Order({
            id: orders.length,
            buyer: msg.sender,
            amount: _amount,
            status: Status.Pending
        }));
    }
    
    function updateBalance(address _user, uint256 _amount) public {
        User storage user = users[_user];
        user.balance += _amount;
    }
}
```

## 错误处理

### 三种方式

```
┌─────────────────────────────────────────────────────────────┐
│                    错误处理方式                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  require(condition, "message")                              │
│  ├── 用于: 输入验证、权限检查                               │
│  ├── 失败时: 退还剩余 Gas                                   │
│  └── 最常用                                                 │
│                                                             │
│  revert("message")                                          │
│  ├── 用于: 复杂条件判断后回退                               │
│  ├── 失败时: 退还剩余 Gas                                   │
│  └── 比 require 更灵活                                      │
│                                                             │
│  assert(condition)                                          │
│  ├── 用于: 内部不变量检查                                   │
│  ├── 失败时: 消耗所有 Gas                                   │
│  └── 极少使用，用于测试                                     │
│                                                             │
│  Java 对比:                                                 │
│  require ≈ Preconditions.checkArgument()                   │
│  revert  ≈ throw new RuntimeException()                    │
│  assert  ≈ assert (Java assert)                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 使用示例

```solidity
contract ErrorHandling {
    uint256 public counter;
    address public owner;
    
    constructor() {
        owner = msg.sender;
    }
    
    function increment(uint256 amount) public {
        require(amount > 0, "Amount must be positive");
        require(amount < 100, "Amount too large");
        
        counter += amount;
    }
    
    function complexValidation(uint256 a, uint256 b) public pure returns (uint256) {
        if (a == 0) {
            revert("a cannot be zero");
        }
        
        if (b == 0) {
            revert("b cannot be zero");
        }
        
        return a / b;
    }
    
    function checkInvariant() public view {
        assert(counter >= 0);  // 永远为真，用于测试
    }
}
```

### 自定义错误 (0.8.4+)

```solidity
contract CustomErrors {
    error InsufficientBalance(uint256 available, uint256 required);
    error Unauthorized();
    error InvalidAmount();
    
    mapping(address => uint256) public balances;
    
    function withdraw(uint256 amount) public {
        uint256 balance = balances[msg.sender];
        
        if (amount == 0) {
            revert InvalidAmount();
        }
        
        if (balance < amount) {
            revert InsufficientBalance(balance, amount);
        }
        
        balances[msg.sender] -= amount;
    }
    
    function adminAction() public {
        if (msg.sender != address(0x1234)) {  // 假设的 admin
            revert Unauthorized();
        }
        // admin 逻辑
    }
}
```

## 最佳实践

### 代码组织

```solidity
contract WellOrganized {
    // 1. 类型定义
    enum Status { Active, Paused }
    struct Info { string name; uint256 value; }
    
    // 2. 状态变量
    address public owner;
    Status public status;
    uint256 public constant VERSION = 1;
    
    // 3. 事件
    event StatusChanged(Status newStatus);
    event OwnerChanged(address newOwner);
    
    // 4. 修饰器
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    // 5. 构造函数
    constructor() {
        owner = msg.sender;
        status = Status.Active;
    }
    
    // 6. 外部函数
    function setOwner(address _newOwner) external onlyOwner {
        owner = _newOwner;
        emit OwnerChanged(_newOwner);
    }
    
    // 7. 公共函数
    function pause() public onlyOwner {
        status = Status.Paused;
        emit StatusChanged(status);
    }
    
    // 8. 内部函数
    function _internalHelper() internal pure returns (uint256) {
        return 0;
    }
    
    // 9. 私有函数
    function _privateHelper() private pure returns (uint256) {
        return 1;
    }
}
```

[下一节：继承与组合 →](03-inheritance.md)
