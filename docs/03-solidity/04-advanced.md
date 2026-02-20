# 高级特性

> 掌握 Solidity 的高级功能和特性

## ETH 收发

### payable 修饰符

```solidity
contract PayableExample {
    address public owner;
    
    constructor() {
        owner = msg.sender;
    }
    
    // 普通函数不能接收 ETH
    function normalFunction() public pure returns (string memory) {
        return "Cannot receive ETH";
    }
    
    // payable 函数可以接收 ETH
    function deposit() public payable {
        // msg.value 包含发送的 ETH 数量
        // 自动添加到合约余额
    }
    
    // payable 构造函数
    constructor() payable {
        // 部署时可以发送 ETH
    }
    
    // payable 地址
    function sendTo(address payable recipient) public payable {
        recipient.transfer(msg.value);
    }
}
```

### 发送 ETH 的三种方式

```
┌─────────────────────────────────────────────────────────────┐
│                  ETH 发送方式对比                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  transfer(amount)                                           │
│  ├── Gas 限制: 2300                                         │
│  ├── 失败时: 自动 revert                                    │
│  └── 用途: 简单转账                                         │
│                                                             │
│  send(amount)                                               │
│  ├── Gas 限制: 2300                                         │
│  ├── 失败时: 返回 false                                     │
│  └── 用途: 需要处理失败情况                                 │
│                                                             │
│  call{value: amount}("")                                    │
│  ├── Gas 限制: 可自定义 (转发所有 Gas)                      │
│  ├── 失败时: 返回 (bool, bytes)                             │
│  ├── 用途: 最灵活，推荐使用                                 │
│  └── 注意: 可能导致重入攻击                                 │
│                                                             │
│  Java 对比:                                                 │
│  transfer ≈ 数据库事务自动回滚                              │
│  send    ≈ 需要检查返回值                                   │
│  call    ≈ 远程调用                                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```solidity
contract SendEther {
    address public owner;
    
    constructor() {
        owner = msg.sender;
    }
    
    receive() external payable {}  // 接收 ETH
    
    // 方式 1: transfer
    function sendViaTransfer(address payable _to) public payable {
        _to.transfer(msg.value);  // 失败自动 revert
    }
    
    // 方式 2: send
    function sendViaSend(address payable _to) public payable {
        bool success = _to.send(msg.value);
        require(success, "Send failed");
    }
    
    // 方式 3: call (推荐)
    function sendViaCall(address payable _to) public payable {
        (bool success, ) = _to.call{value: msg.value}("");
        require(success, "Call failed");
    }
    
    // 带数据的调用
    function callWithFunction(address _contract, uint256 amount) public payable {
        (bool success, bytes memory data) = _contract.call{value: msg.value}(
            abi.encodeWithSignature("deposit(uint256)", amount)
        );
        require(success, "Call failed");
    }
}
```

## receive 和 fallback

### 执行流程

```
┌─────────────────────────────────────────────────────────────┐
│                receive/fallback 执行流程                     │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│            收到 ETH 调用                                    │
│                 │                                           │
│                 ▼                                           │
│     ┌───────────────────────┐                              │
│     │ 是否有调用数据?        │                              │
│     └───────────┬───────────┘                              │
│           │           │                                    │
│          否           是                                    │
│           │           │                                    │
│           ▼           ▼                                    │
│     ┌──────────┐  ┌──────────────┐                         │
│     │ receive  │  │ 有函数匹配?   │                         │
│     │ (纯ETH)  │  └──────┬───────┘                         │
│     └──────────┘    是  │    否                             │
│                    │    │                                   │
│                    ▼    ▼                                   │
│              执行函数  fallback                              │
│                                                             │
│  注意:                                                      │
│  - receive() 接收纯 ETH 转账                                │
│  - fallback() 处理未知函数调用                              │
│  - 两者都必须是 external payable                            │
│  - 最多 2300 Gas (transfer/send)                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 代码示例

```solidity
contract ReceiveFallback {
    event Received(address sender, uint256 amount);
    event FallbackCalled(address sender, uint256 amount, bytes data);
    
    uint256 public receivedCount;
    uint256 public fallbackCount;
    
    // 接收纯 ETH 转账
    receive() external payable {
        receivedCount++;
        emit Received(msg.sender, msg.value);
    }
    
    // 处理未知函数调用
    fallback() external payable {
        fallbackCount++;
        emit FallbackCalled(msg.sender, msg.value, msg.data);
    }
    
    // 正常函数
    function getBalance() public view returns (uint256) {
        return address(this).balance;
    }
    
    function deposit() public payable {
        // 正常存款
    }
}

// 测试:
// 1. transfer(1 ether) -> 触发 receive()
// 2. call{value: 1 ether}("unknownFunc()") -> 触发 fallback()
// 3. deposit{value: 1 ether}() -> 正常调用 deposit()
```

### 代理合约模式

```solidity
contract Proxy {
    address public implementation;
    
    constructor(address _implementation) {
        implementation = _implementation;
    }
    
    // 转发所有未知调用到实现合约
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
    
    receive() external payable {
        // 接收 ETH
    }
}
```

## 错误处理进阶

### require vs revert vs assert

```solidity
contract ErrorHandlingAdvanced {
    uint256 public counter;
    address public owner;
    
    constructor() {
        owner = msg.sender;
    }
    
    // require - 输入验证、条件检查
    function increment(uint256 amount) public {
        require(amount > 0, "Amount must be positive");
        require(amount <= 100, "Amount too large");
        require(msg.sender == owner, "Only owner");
        
        counter += amount;
    }
    
    // revert - 复杂条件
    function complexCheck(uint256 a, uint256 b) public pure returns (uint256) {
        if (a == 0) {
            revert("a cannot be zero");
        }
        
        if (b == 0) {
            revert("b cannot be zero");
        }
        
        if (a > b) {
            revert("a must be <= b");
        }
        
        return b - a;
    }
    
    // assert - 不变量检查 (测试用)
    function withAssert() public view {
        assert(counter >= 0);  // 永远为真
        assert(owner != address(0));
    }
}
```

### 自定义错误

```solidity
contract CustomErrors {
    error InsufficientBalance(uint256 available, uint256 required);
    error Unauthorized(address caller);
    error InvalidAmount();
    error TransferFailed();
    
    mapping(address => uint256) public balances;
    address public owner;
    
    constructor() {
        owner = msg.sender;
    }
    
    function withdraw(uint256 amount) public {
        if (amount == 0) {
            revert InvalidAmount();
        }
        
        uint256 balance = balances[msg.sender];
        
        if (balance < amount) {
            revert InsufficientBalance(balance, amount);
        }
        
        balances[msg.sender] -= amount;
        
        (bool success, ) = payable(msg.sender).call{value: amount}("");
        if (!success) {
            revert TransferFailed();
        }
    }
    
    function adminWithdraw(address to, uint256 amount) public {
        if (msg.sender != owner) {
            revert Unauthorized(msg.sender);
        }
        
        (bool success, ) = payable(to).call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

### try/catch (外部调用)

```solidity
interface IExternalContract {
    function riskyOperation(uint256 value) external returns (uint256);
}

contract TryCatch {
    event OperationSuccess(uint256 result);
    event OperationFailed(string reason);
    event OperationFailedBytes(bytes data);
    
    IExternalContract public externalContract;
    
    constructor(address _contract) {
        externalContract = IExternalContract(_contract);
    }
    
    function safeOperation(uint256 value) public returns (uint256) {
        try externalContract.riskyOperation(value) returns (uint256 result) {
            emit OperationSuccess(result);
            return result;
        } catch Error(string memory reason) {
            emit OperationFailed(reason);
            revert(reason);
        } catch (bytes memory data) {
            emit OperationFailedBytes(data);
            revert("Unknown error");
        }
    }
    
    // try/catch 只能用于外部调用
    // 内部调用不能使用 try/catch
}
```

## 库 (Library)

### 库的特点

```
┌─────────────────────────────────────────────────────────────┐
│                    Library 特点                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 不能有状态变量                                          │
│  2. 不能继承或被继承                                        │
│  3. 不能接收 ETH                                            │
│  4. 不能被销毁                                              │
│  5. 函数默认 internal                                       │
│                                                             │
│  使用方式:                                                  │
│  ├── 直接调用: Library.function()                          │
│  └── using for: using Library for Type                     │
│                                                             │
│  Java 对比:                                                 │
│  Library ≈ 工具类 (static methods only)                    │
│  using for ≈ 扩展函数 (Kotlin)                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 库的定义

```solidity
library Math {
    function max(uint256 a, uint256 b) internal pure returns (uint256) {
        return a >= b ? a : b;
    }
    
    function min(uint256 a, uint256 b) internal pure returns (uint256) {
        return a <= b ? a : b;
    }
    
    function sqrt(uint256 x) internal pure returns (uint256) {
        if (x == 0) return 0;
        
        uint256 z = (x + 1) / 2;
        uint256 y = x;
        
        while (z < y) {
            y = z;
            z = (x / z + z) / 2;
        }
        
        return y;
    }
}

library Address {
    function isContract(address account) internal view returns (bool) {
        uint256 size;
        assembly {
            size := extcodesize(account)
        }
        return size > 0;
    }
    
    function sendValue(address payable recipient, uint256 amount) internal {
        require(address(this).balance >= amount, "Insufficient balance");
        
        (bool success, ) = recipient.call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

### using for 用法

```solidity
contract UsingLibrary {
    using Math for uint256;
    using Address for address;
    
    uint256 public value;
    
    function example() public view returns (uint256) {
        uint256 a = 100;
        uint256 b = 200;
        
        // 使用 Math 库
        uint256 maxValue = a.max(b);  // 等同于 Math.max(a, b)
        uint256 minValue = a.min(b);  // 等同于 Math.min(a, b)
        
        // 链式调用
        uint256 result = a.max(b).sqrt();  // Math.sqrt(Math.max(a, b))
        
        return result;
    }
    
    function checkAddress(address addr) public view returns (bool) {
        return addr.isContract();  // Address.isContract(addr)
    }
}
```

### SafeMath 示例 (0.8.0 前必需)

```solidity
library SafeMath {
    function add(uint256 a, uint256 b) internal pure returns (uint256) {
        uint256 c = a + b;
        require(c >= a, "Addition overflow");
        return c;
    }
    
    function sub(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b <= a, "Subtraction underflow");
        return a - b;
    }
    
    function mul(uint256 a, uint256 b) internal pure returns (uint256) {
        if (a == 0) return 0;
        
        uint256 c = a * b;
        require(c / a == b, "Multiplication overflow");
        return c;
    }
    
    function div(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b > 0, "Division by zero");
        return a / b;
    }
    
    function mod(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b != 0, "Modulo by zero");
        return a % b;
    }
}

// Solidity 0.8.0+ 内置溢出检查，不再需要 SafeMath
```

## 编码与哈希

### 编码函数

```solidity
contract Encoding {
    function encode(uint256 a, address b) public pure returns (bytes memory) {
        return abi.encode(a, b);
    }
    
    function encodePacked(uint256 a, string memory b) public pure returns (bytes memory) {
        return abi.encodePacked(a, b);  // 紧凑编码
    }
    
    function encodeWithSelector(uint256 a) public pure returns (bytes memory) {
        return abi.encodeWithSelector(this.example.selector, a);
    }
    
    function encodeWithSignature(uint256 a) public pure returns (bytes memory) {
        return abi.encodeWithSignature("example(uint256)", a);
    }
    
    function decode(bytes memory data) public pure returns (uint256, address) {
        return abi.decode(data, (uint256, address));
    }
    
    function example(uint256) public pure {}
}
```

### 哈希函数

```solidity
contract Hashing {
    function keccak256Hash(string memory text) public pure returns (bytes32) {
        return keccak256(abi.encodePacked(text));
    }
    
    function sha256Hash(string memory text) public pure returns (bytes32) {
        return sha256(abi.encodePacked(text));
    }
    
    function hashTwoValues(uint256 a, uint256 b) public pure returns (bytes32) {
        return keccak256(abi.encodePacked(a, b));
    }
    
    function createCommitment(string memory secret) public pure returns (bytes32) {
        return keccak256(abi.encodePacked(msg.sender, secret));
    }
}
```

## 数学运算

```solidity
contract MathOperations {
    function basicMath(uint256 a, uint256 b) public pure returns (
        uint256 sum,
        uint256 diff,
        uint256 product,
        uint256 quotient,
        uint256 remainder
    ) {
        sum = a + b;
        
        // 安全减法 (0.8.0+ 自动检查)
        diff = a > b ? a - b : 0;
        
        product = a * b;
        quotient = a / b;
        remainder = a % b;
    }
    
    function power(uint256 base, uint256 exponent) public pure returns (uint256) {
        uint256 result = 1;
        for (uint256 i = 0; i < exponent; i++) {
            result *= base;
        }
        return result;
    }
    
    function absolute(int256 value) public pure returns (uint256) {
        return value >= 0 ? uint256(value) : uint256(-value);
    }
    
    // 精度处理 (类似 BigDecimal)
    uint256 constant DECIMALS = 18;
    
    function multiplyWithDecimals(uint256 a, uint256 b) public pure returns (uint256) {
        return (a * b) / (10 ** DECIMALS);
    }
    
    function divideWithDecimals(uint256 a, uint256 b) public pure returns (uint256) {
        return (a * (10 ** DECIMALS)) / b;
    }
}
```

## 随机数

```solidity
contract Randomness {
    // 警告: 链上随机数可被预测，不安全！
    // 仅用于演示，生产环境请使用 Chainlink VRF
    
    function unsafeRandom() public view returns (uint256) {
        return uint256(keccak256(abi.encodePacked(
            block.timestamp,
            block.prevrandao,
            msg.sender
        )));
    }
    
    function unsafeRandomRange(uint256 min, uint256 max) public view returns (uint256) {
        return unsafeRandom() % (max - min + 1) + min;
    }
}
```

## Gas 优化技巧

```solidity
contract GasOptimization {
    uint256 public a;
    uint256 public b;
    uint256 public c;
    
    uint256 private locked = 0;
    
    // ❌ 多次存储读取
    function badRead() public view returns (uint256) {
        return a + b + c;
    }
    
    // ✅ 内存缓存
    function goodRead() public view returns (uint256) {
        uint256 _a = a;
        uint256 _b = b;
        uint256 _c = c;
        return _a + _b + _c;
    }
    
    // ❌ 循环中存储操作
    function badLoop(uint256[] memory arr) public {
        for (uint256 i = 0; i < arr.length; i++) {
            a += arr[i];
        }
    }
    
    // ✅ 内存计算后一次存储
    function goodLoop(uint256[] memory arr) public {
        uint256 sum = 0;
        for (uint256 i = 0; i < arr.length; i++) {
            sum += arr[i];
        }
        a += sum;
    }
    
    // ❌ 短路评估
    function badCondition(uint256 x) public view returns (bool) {
        return expensiveOperation(x) && x > 0;
    }
    
    // ✅ 便宜条件先评估
    function goodCondition(uint256 x) public view returns (bool) {
        return x > 0 && expensiveOperation(x);
    }
    
    function expensiveOperation(uint256) internal pure returns (bool) {
        return true;
    }
    
    // ✅ 使用 unchecked (确定不会溢出时)
    function uncheckedLoop() public {
        unchecked {
            for (uint256 i = 0; i < 100; i++) {
                // 0.8.0+ 省略溢出检查
            }
        }
    }
    
    // ✅ 使用 calldata (external 函数)
    function useCalldata(uint256[] calldata arr) external pure returns (uint256) {
        return arr[0];
    }
    
    // ❌ 使用 memory
    function useMemory(uint256[] memory arr) public pure returns (uint256) {
        return arr[0];
    }
}
```

## 实战示例

### 简单拍卖合约

```solidity
contract SimpleAuction {
    address public beneficiary;
    uint256 public auctionEndTime;
    
    address public highestBidder;
    uint256 public highestBid;
    
    mapping(address => uint256) public pendingReturns;
    bool public ended;
    
    event HighestBidIncreased(address bidder, uint256 amount);
    event AuctionEnded(address winner, uint256 amount);
    
    constructor(uint256 biddingTime, address _beneficiary) {
        beneficiary = _beneficiary;
        auctionEndTime = block.timestamp + biddingTime;
    }
    
    function bid() public payable {
        require(block.timestamp < auctionEndTime, "Auction ended");
        require(msg.value > highestBid, "Bid not high enough");
        
        if (highestBid != 0) {
            pendingReturns[highestBidder] += highestBid;
        }
        
        highestBidder = msg.sender;
        highestBid = msg.value;
        
        emit HighestBidIncreased(msg.sender, msg.value);
    }
    
    function withdraw() public returns (bool) {
        uint256 amount = pendingReturns[msg.sender];
        
        if (amount > 0) {
            pendingReturns[msg.sender] = 0;
            
            (bool success, ) = payable(msg.sender).call{value: amount}("");
            require(success, "Transfer failed");
        }
        
        return true;
    }
    
    function auctionEnd() public {
        require(block.timestamp >= auctionEndTime, "Auction not yet ended");
        require(!ended, "Auction already ended");
        
        ended = true;
        emit AuctionEnded(highestBidder, highestBid);
        
        (bool success, ) = payable(beneficiary).call{value: highestBid}("");
        require(success, "Transfer failed");
    }
}
```

[下一节：安全实践 →](05-security.md)
