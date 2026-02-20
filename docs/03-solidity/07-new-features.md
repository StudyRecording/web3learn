# 3.7 Solidity 新特性

## 瞬态存储 (Transient Storage)

### 什么是瞬态存储

瞬态存储是 Solidity 0.8.24 引入的新数据位置，通过 EIP-1153 实现。

```
┌─────────────────────────────────────────────────────────────┐
│                   存储类型对比                              │
├──────────────┬─────────────┬─────────────┬─────────────────┤
│     特性      │   Storage   │  Transient  │     Memory      │
├──────────────┼─────────────┼─────────────┼─────────────────┤
│ Gas 成本     │ 20,000+     │ 100         │ 低              │
│ 持久性       │ 永久        │ 交易期间    │ 函数调用期间    │
│ 作用域       │ 跨交易      │ 跨函数调用  │ 单函数          │
│ 清理         │ 需手动      │ 自动清除    │ 自动清除        │
└──────────────┴─────────────┴─────────────┴─────────────────┘
```

### 使用方法

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

contract TransientExample {
    // 声明瞬态存储变量 (Solidity 0.8.28+)
    bool transient locked;
    uint256 transient tempValue;
    
    // 使用瞬态存储的重入锁
    modifier nonReentrant() {
        require(!locked, "Reentrancy detected");
        locked = true;
        _;
        locked = false;  // 交易结束后自动清除
    }
    
    // 瞬态存储用于临时计算
    function calculateWithTemp(uint256 a, uint256 b) public returns (uint256) {
        tempValue = a * b;  // 存储中间结果
        // ... 其他操作
        uint256 result = tempValue / 100;
        tempValue = 0;  // 可选：手动清除
        return result;
    }
}
```

### 汇编方式使用 (Solidity 0.8.24+)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

contract TransientAssembly {
    function useTransientStorage() public pure returns (uint256) {
        uint256 result;
        assembly {
            // 存储到瞬态存储
            tstore(0x01, 100)
            
            // 从瞬态存储读取
            result := tload(0x01)
        }
        return result;
    }
}
```

### 瞬态存储 vs 传统存储对比

```solidity
// 传统重入锁 - 使用存储
contract OldReentrancyGuard {
    uint256 private _status = 1;  // 存储：每次写入 20,000+ gas
    
    modifier nonReentrant() {
        require(_status == 1, "Reentrancy");
        _status = 2;
        _;
        _status = 1;
    }
}

// 使用瞬态存储 - 更高效
contract NewReentrancyGuard {
    bool transient _locked;  // 瞬态存储：每次操作仅 100 gas
    
    modifier nonReentrant() {
        require(!_locked, "Reentrancy");
        _locked = true;
        _;
        _locked = false;
    }
}

// Gas 节省：每次调用节省约 40,000+ gas
```

### 适用场景

```
瞬态存储适用场景:

1. 重入锁 ✓
   - 最经典的应用场景
   - Gas 节省显著

2. 单次授权 (Single-use allowances) ✓
   - 临时授权在交易内有效
   - 交易后自动清除

3. 多步计算的中间结果 ✓
   - 存储中间计算结果
   - 避免 memory 扩展成本

4. 跨合约调用的临时数据 ✓
   - 在同一交易内的合约间传递数据

不适合的场景:

✗ 需要持久化的数据
✗ 跨交易的状态
✗ 大型数据结构（仅支持值类型）
```

### 注意事项

```solidity
contract TransientCautions {
    // ❌ 不能在声明时初始化
    // uint256 transient value = 100;  // 错误！
    
    // ✓ 正确：声明后赋值
    uint256 transient value;
    
    function set() public {
        value = 100;
    }
    
    // 仅支持值类型
    uint256 transient num;        // ✓
    address transient addr;       // ✓
    bool transient flag;          // ✓
    
    // ❌ 不支持引用类型
    // uint256[] transient arr;    // 错误！
    // mapping(...) transient m;   // 错误！
}
```

## 其他新特性

### unchecked 循环增量 (Solidity 0.8.22+)

```solidity
// 旧版本需要手动 unchecked
contract OldLoop {
    function sum(uint256[] memory arr) public pure returns (uint256) {
        uint256 total;
        for (uint256 i = 0; i < arr.length;) {
            total += arr[i];
            unchecked { i++; }  // 手动优化
        }
        return total;
    }
}

// 新版本自动优化 (Solidity 0.8.22+)
contract NewLoop {
    function sum(uint256[] memory arr) public pure returns (uint256) {
        uint256 total;
        for (uint256 i = 0; i < arr.length; i++) {
            total += arr[i];
            // 编译器自动使用 unchecked 增量
        }
        return total;
    }
}
```

### require 中的自定义错误 (Solidity 0.8.26+)

```solidity
// 定义自定义错误
error InsufficientBalance(uint256 available, uint256 required);

contract RequireWithCustomError {
    mapping(address => uint256) public balances;
    
    // 旧版本
    function oldWithdraw(uint256 amount) public {
        require(balances[msg.sender] >= amount, "Insufficient balance");
        // ...
    }
    
    // 新版本 - 使用自定义错误
    function newWithdraw(uint256 amount) public {
        require(
            balances[msg.sender] >= amount,
            InsufficientBalance(balances[msg.sender], amount)
        );
        // 更清晰、更省 Gas 的错误信息
    }
}
```

### 即将废弃的特性 (Solidity 0.8.31+)

```solidity
contract DeprecatedFeatures {
    // ⚠️ 即将废弃的 transfer 和 send
    function oldStyle() public {
        // ❌ 将被废弃
        payable(msg.sender).transfer(1 ether);
        
        // ❌ 将被废弃
        bool success = payable(msg.sender).send(1 ether);
        
        // ✓ 推荐使用 call
        (bool success, ) = msg.sender.call{value: 1 ether}("");
        require(success, "Transfer failed");
    }
}
```

## 编译器版本选择建议

```
推荐版本:

新项目:
├── Solidity 0.8.28+ (支持瞬态存储关键字)
├── EVM 版本: cancun 或最新
└── 启用优化器

需要兼容旧链:
├── Solidity 0.8.20+
├── EVM 版本: paris (Merge 后)
└── 注意不支持瞬态存储

版本声明:
pragma solidity ^0.8.28;   // 推荐：允许小版本更新
pragma solidity >=0.8.28;  // 允许任何更新
pragma solidity 0.8.28;    // 锁定版本（审计项目推荐）
```

## 下一步

[返回目录](README.md)
