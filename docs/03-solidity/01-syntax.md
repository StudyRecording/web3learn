# Solidity 语法基础

> 掌握 Solidity 的数据类型、变量、函数和控制流

## 数据类型

### 值类型 (Value Types)

```
┌─────────────────────────────────────────────────────────────┐
│                    Solidity 值类型                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  布尔型                                                      │
│  └── bool (true/false)                                      │
│                                                             │
│  整数型                                                      │
│  ├── int8   ~ int256   (有符号，步长8)                      │
│  └── uint8  ~ uint256  (无符号，步长8)                      │
│                                                             │
│  定点数 (少用)                                               │
│  ├── fixed / ufixed                                         │
│                                                             │
│  地址类型                                                    │
│  ├── address   (20字节，不含 ETH 余额)                      │
│  └── address payable (可接收 ETH)                           │
│                                                             │
│  定长字节数组                                                │
│  └── bytes1 ~ bytes32                                       │
│                                                             │
│  枚举                                                        │
│  └── enum                                                   │
│                                                             │
│  用户定义值类型                                              │
│  └── type C is uint256                                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 整数类型

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract IntegerTypes {
    int8 public signed8 = -10;        // -128 to 127
    uint8 public unsigned8 = 255;     // 0 to 255
    
    int256 public signed256 = -1000;  // 常用
    uint256 public unsigned256 = 1000; // 最常用
    
    // Java 对比:
    // int8  ≈ byte
    // int256 ≈ BigInteger (范围更大)
    // uint256 ≈ 无符号 BigInteger
}
```

### 地址类型

```solidity
contract AddressTypes {
    address public owner;
    address payable public treasury;
    
    constructor() {
        owner = msg.sender;
        treasury = payable(msg.sender);
    }
    
    function getBalance() public view returns (uint256) {
        return owner.balance;
    }
    
    function sendEther() public payable {
        treasury.transfer(msg.value);
    }
}
```

### 引用类型 (Reference Types)

```
┌─────────────────────────────────────────────────────────────┐
│                    数据位置修饰符                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  storage   - 永久存储 (状态变量默认)                         │
│             类似: 数据库                                     │
│             成本: 高                                         │
│                                                             │
│  memory    - 临时内存 (函数参数/局部变量)                    │
│             类似: RAM                                        │
│             成本: 低                                         │
│                                                             │
│  calldata  - 只读调用数据 (external 函数参数)                │
│             类似: 只读 buffer                                │
│             成本: 最低                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```solidity
contract ReferenceTypes {
    // storage - 状态变量
    uint256[] public numbers;
    mapping(address => uint256) public balances;
    
    function example(
        string memory name,           // memory
        uint256[] calldata ids        // calldata (external only)
    ) external pure returns (string memory) {
        uint256[] memory temp = new uint256[](10);  // memory
        return name;
    }
    
    // 数组
    uint256[] public dynamicArray;
    uint256[10] public fixedArray;
    
    // 结构体
    struct User {
        string name;
        uint256 balance;
        bool isActive;
    }
    
    User public user;
    User[] public users;
    
    // 映射
    mapping(address => User) public userMap;
    mapping(address => mapping(uint256 => bool)) public nested;
}
```

### Java 对比

```solidity
// Solidity
struct User {
    string name;
    uint256 balance;
}

mapping(address => User) users;
```

```java
// Java 等价
public class User {
    private String name;
    private BigInteger balance;
}

public class Contract {
    private Map<Address, User> users = new HashMap<>();
}
```

## 变量

### 变量类型

```solidity
contract Variables {
    // 1. 状态变量 (storage)
    uint256 public stateVar = 100;
    
    // 2. 局部变量 (memory/stack)
    function example() public pure returns (uint256) {
        uint256 localVar = 200;
        return localVar;
    }
    
    // 3. 全局变量 (内置)
    function globalVars() public view returns (
        address sender,
        uint256 value,
        uint256 timestamp,
        uint256 blockNum
    ) {
        sender = msg.sender;      // 调用者地址
        value = msg.value;        // 发送的 ETH
        timestamp = block.timestamp;  // 区块时间戳
        blockNum = block.number;      // 区块高度
    }
}
```

### 常量与不可变

```solidity
contract Constants {
    // constant - 编译时确定
    uint256 public constant DECIMALS = 18;
    address public constant ZERO = address(0);
    
    // immutable - 部署时确定
    address public immutable OWNER;
    uint256 public immutable CREATED_AT;
    
    constructor() {
        OWNER = msg.sender;
        CREATED_AT = block.timestamp;
    }
}
```

### 全局变量速查

```
┌─────────────────────────────────────────────────────────────┐
│                     常用全局变量                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  msg.sender      - 调用者地址                               │
│  msg.value       - 发送的 ETH (wei)                         │
│  msg.data        - 完整调用数据                             │
│  msg.sig         - 函数选择器 (前4字节)                     │
│                                                             │
│  block.timestamp - 当前区块时间戳                           │
│  block.number    - 当前区块号                               │
│  block.coinbase  - 矿工地址                                 │
│  block.difficulty- 区块难度                                 │
│  block.gaslimit  - 区块 Gas 限制                            │
│                                                             │
│  tx.origin       - 交易原始发起者                           │
│  tx.gasprice     - Gas 价格                                 │
│                                                             │
│  blockhash(n)    - 指定区块的哈希                           │
│  gasleft()       - 剩余 Gas                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 函数

### 函数声明

```solidity
contract FunctionBasics {
    uint256 private counter;
    
    function increment() public {
        counter += 1;
    }
    
    // 带参数和返回值
    function add(uint256 a, uint256 b) public pure returns (uint256) {
        return a + b;
    }
    
    // 命名返回值
    function subtract(uint256 a, uint256 b) public pure returns (uint256 result) {
        result = a - b;
    }
    
    // 多返回值
    function divide(uint256 a, uint256 b) public pure returns (uint256 quotient, uint256 remainder) {
        quotient = a / b;
        remainder = a % b;
    }
}
```

### 可见性修饰符

```
┌─────────────────────────────────────────────────────────────┐
│                    函数可见性                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  public    - 内部 + 外部都可调用                            │
│              (状态变量自动生成 getter)                      │
│                                                             │
│  external  - 仅外部调用 (更省 Gas)                          │
│              (参数使用 calldata)                            │
│                                                             │
│  internal  - 仅本合约 + 子合约可调用                        │
│              (状态变量默认)                                 │
│                                                             │
│  private   - 仅本合约可调用                                 │
│              (子合约也不可访问)                             │
│                                                             │
│  Java 对比:                                                 │
│  public     ≈ public                                        │
│  external   ≈ (无直接对应，类似 remote API)                 │
│  internal   ≈ protected                                     │
│  private    ≈ private                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```solidity
contract Visibility {
    uint256 public publicVar = 1;       // 自动生成 getter
    uint256 private privateVar = 2;
    uint256 internal internalVar = 3;
    
    function publicFunc() public pure returns (string memory) {
        return "public";
    }
    
    function externalFunc() external pure returns (string memory) {
        return "external";
    }
    
    function internalFunc() internal pure returns (string memory) {
        return "internal";
    }
    
    function privateFunc() private pure returns (string memory) {
        return "private";
    }
}
```

### 状态修饰符

```solidity
contract StateModifiers {
    uint256 public counter;
    
    // view - 只读，不修改状态
    function getCounter() public view returns (uint256) {
        return counter;
    }
    
    // pure - 不读取也不修改状态
    function add(uint256 a, uint256 b) public pure returns (uint256) {
        return a + b;
    }
    
    // 无修饰符 - 可以读和写
    function increment() public {
        counter += 1;
    }
    
    // payable - 可接收 ETH
    function deposit() public payable {
        // msg.value 可用
    }
}
```

### 自定义修饰器

```solidity
contract Modifiers {
    address public owner;
    uint256 public counter;
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    modifier costs(uint256 price) {
        require(msg.value >= price, "Insufficient ETH");
        _;
    }
    
    constructor() {
        owner = msg.sender;
    }
    
    function increment() public onlyOwner {
        counter += 1;
    }
    
    function buy() public payable costs(1 ether) {
        // 已支付至少 1 ETH
    }
}
```

## 控制流

### 条件语句

```solidity
contract ControlFlow {
    function conditional(uint256 x) public pure returns (string memory) {
        if (x < 10) {
            return "small";
        } else if (x < 100) {
            return "medium";
        } else {
            return "large";
        }
    }
    
    function ternary(uint256 x) public pure returns (string memory) {
        return x > 0 ? "positive" : "zero or negative";
    }
}
```

### 循环

```solidity
contract Loops {
    function sum(uint256 n) public pure returns (uint256) {
        uint256 total = 0;
        
        for (uint256 i = 1; i <= n; i++) {
            total += i;
        }
        
        return total;
    }
    
    function whileLoop(uint256 n) public pure returns (uint256) {
        uint256 sum = 0;
        uint256 i = 1;
        
        while (i <= n) {
            sum += i;
            i++;
        }
        
        return sum;
    }
    
    function doWhile(uint256 n) public pure returns (uint256) {
        uint256 sum = 0;
        uint256 i = 1;
        
        do {
            sum += i;
            i++;
        } while (i <= n);
        
        return sum;
    }
    
    // 遍历数组
    function sumArray(uint256[] memory arr) public pure returns (uint256) {
        uint256 total = 0;
        
        for (uint256 i = 0; i < arr.length; i++) {
            total += arr[i];
        }
        
        return total;
    }
}
```

### 注意事项

```
┌─────────────────────────────────────────────────────────────┐
│                    循环注意事项                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Gas 限制                                                 │
│     循环次数受 Gas 限制                                      │
│     大数组遍历可能超出区块 Gas 限制                          │
│                                                             │
│  2. 无限循环                                                 │
│     确保循环有终止条件                                       │
│     无限循环会消耗所有 Gas                                   │
│                                                             │
│  3. 最佳实践                                                 │
│     - 限制循环次数                                           │
│     - 使用映射替代数组查找                                   │
│     - 批量操作分页处理                                       │
│                                                             │
│  Java 对比:                                                 │
│  Java 循环可以无限运行                                      │
│  Solidity 循环受 Gas 限制必须快速完成                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```solidity
// 错误示例 - 可能 OOG (Out of Gas)
function badSum(uint256[] memory arr) public pure returns (uint256) {
    uint256 total = 0;
    for (uint256 i = 0; i < arr.length; i++) {
        total += arr[i];
    }
    return total;
}

// 正确示例 - 分页处理
function sumPage(
    uint256[] memory arr,
    uint256 start,
    uint256 limit
) public pure returns (uint256) {
    uint256 total = 0;
    uint256 end = start + limit;
    
    if (end > arr.length) {
        end = arr.length;
    }
    
    for (uint256 i = start; i < end; i++) {
        total += arr[i];
    }
    
    return total;
}
```

## 类型转换

```solidity
contract TypeConversion {
    function explicit() public pure returns (uint256) {
        int256 negative = -10;
        uint256 positive = uint256(negative);  // 小心: 结果很大
        
        uint8 small = 255;
        uint256 large = uint256(small);        // 安全
        
        return large;
    }
    
    function addressConversion() public pure returns (address) {
        uint160 addrUint = 123456789;
        address addr = address(addrUint);
        return addr;
    }
    
    function bytesConversion() public pure returns (bytes memory) {
        string memory text = "Hello";
        bytes memory data = bytes(text);
        return data;
    }
}
```

## 枚举

```solidity
contract Enums {
    enum Status { Pending, Active, Completed, Cancelled }
    
    Status public status;
    
    constructor() {
        status = Status.Pending;
    }
    
    function activate() public {
        status = Status.Active;
    }
    
    function complete() public {
        require(status == Status.Active, "Not active");
        status = Status.Completed;
    }
    
    function getStatus() public view returns (Status) {
        return status;
    }
    
    function reset() public {
        delete status;  // 重置为第一个值 (Pending)
    }
}
```

## 实践练习

1. 创建一个简单计数器合约，包含增减和重置功能
2. 实现一个支持存款取款的银行合约
3. 创建一个使用枚举管理订单状态的合约

[下一节：合约结构 →](02-contract-structure.md)
