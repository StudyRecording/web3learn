# 2.3 EVM 虚拟机

## 什么是 EVM

EVM (Ethereum Virtual Machine) 是以太坊的核心，一个**图灵完备的虚拟机**，用于执行智能合约。

```
┌─────────────────────────────────────────────────────────────┐
│                     EVM 架构                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  输入                        EVM                      输出 │
│  ┌─────────┐            ┌───────────┐            ┌────────┐│
│  │ 交易    │            │           │            │状态变更││
│  │ + 数据  │───────────>│    EVM    │───────────>│ + 事件 ││
│  └─────────┘            │           │            └────────┘│
│                         │  ┌─────┐  │                      │
│                         │  │Stack│  │                      │
│                         │  └─────┘  │                      │
│                         │  ┌─────┐  │                      │
│                         │  │Memory│  │                      │
│                         │  └─────┘  │                      │
│                         │  ┌───────┐│                      │
│                         │  │Storage││                      │
│                         │  └───────┘│                      │
│                         └───────────┘                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## EVM 特点

```
EVM 特点

1. 图灵完备
   └── 可以执行任意计算（受 Gas 限制）

2. 沙箱隔离
   ├── 合约之间相互隔离
   └── 无法直接访问外部资源

3. 确定性
   ├── 相同输入产生相同输出
   └── 所有节点执行结果一致

4. 基于栈
   ├── 操作数存储在栈上
   └── 栈深度最大 1024

5. 256位
   ├── 字长 256 位 (32 字节)
   └── 适合密码学运算
```

## EVM 存储结构

### 1. Stack (栈)

```
栈 (Stack)

特点:
├── 后进先出 (LIFO)
├── 最大深度 1024
├── 每个元素 256 位
└── 用于计算和临时存储

操作:
├── PUSH: 压入值
├── POP: 弹出值
├── DUP: 复制栈顶元素
└── SWAP: 交换元素

示例 (加法):
栈: []
PUSH 5  -> 栈: [5]
PUSH 3  -> 栈: [5, 3]
ADD     -> 栈: [8]  (5 + 3)
```

### 2. Memory (内存)

```
内存 (Memory)

特点:
├── 字节寻址数组
├── 运行时临时存储
├── 交易结束后清除
├── 读写需要 Gas
└── 按需扩展

用途:
├── 函数参数
├── 返回数据
├── 临时变量
└── 外部调用数据

示例:
Memory: [0x00, 0x01, 0x02, ...]
MSTORE 0x00, 0x1234...  // 存储 32 字节到位置 0
MLOAD 0x00              // 从位置 0 读取 32 字节
```

### 3. Storage (存储)

```
存储 (Storage)

特点:
├── 持久化键值存储
├── 合约级别的存储
├── 256 位键 -> 256 位值
├── 读写 Gas 较高
└── 数据永久保存

布局:
┌─────────────────────────────────────────┐
│ Slot 0: 状态变量 1                       │
│ Slot 1: 状态变量 2                       │
│ Slot 2: 状态变量 3                       │
│ ...                                     │
│ Slot 2^256-1: 最大槽位                  │
└─────────────────────────────────────────┘

映射的存储:
keccak256(key . slot) -> value
```

### 存储对比

| 特性 | Stack | Memory | Storage |
|------|-------|--------|---------|
| 持久性 | 无 | 无 | 永久 |
| 容量 | 1024 × 32字节 | 可扩展 | 2^256 槽位 |
| 速度 | 最快 | 中等 | 最慢 |
| Gas | 低 | 中 | 高 |
| 用途 | 计算 | 临时数据 | 状态存储 |

## 字节码

### 编译流程

```
Solidity 到字节码

┌────────────┐    ┌────────────┐    ┌────────────┐
│  Solidity  │───>│  编译器    │───>│   字节码   │
│   源代码   │    │  (solc)    │    │  (Bytecode)│
└────────────┘    └────────────┘    └────────────┘

示例:
// 源代码
contract Add {
    function add(uint a, uint b) public pure returns (uint) {
        return a + b;
    }
}

// 编译后字节码
608060405234801561001057600080fd5b50...
```

### 常见操作码

```
EVM 操作码 (Opcode)

算术运算:
├── ADD: 加法
├── SUB: 减法
├── MUL: 乘法
├── DIV: 除法
├── MOD: 取模
└── EXP: 指数

比较运算:
├── LT: 小于
├── GT: 大于
├── EQ: 等于
└── ISZERO: 是否为零

位运算:
├── AND: 与
├── OR: 或
├── XOR: 异或
└── NOT: 非

栈操作:
├── PUSH1-PUSH32: 压入 1-32 字节
├── POP: 弹出
├── DUP1-DUP16: 复制
└── SWAP1-SWAP16: 交换

内存操作:
├── MLOAD: 从内存读取
├── MSTORE: 写入内存
└── MSTORE8: 写入单字节

存储操作:
├── SLOAD: 从存储读取
└── SSTORE: 写入存储

控制流:
├── JUMP: 无条件跳转
├── JUMPI: 条件跳转
├── PC: 程序计数器
├── JUMPDEST: 跳转目标
└── STOP: 停止执行

区块信息:
├── COINBASE: 区块矿工地址
├── TIMESTAMP: 区块时间戳
├── NUMBER: 区块号
├── GASLIMIT: Gas 上限
└── CHAINID: 链 ID

其他:
├── BALANCE: 账户余额
├── CALL: 调用其他合约
├── CREATE: 创建新合约
├── SELFDESTRUCT: 销毁合约
└── REVERT: 回滚
```

### 字节码示例

```
简单的加法函数字节码分析

函数: add(2, 3)

// 调用数据 (calldata)
0x771602f7     // 函数选择器 (add(uint256,uint256))
0000...0002    // 第一个参数 (2)
0000...0003    // 第二个参数 (3)

// 字节码执行
PUSH1 0x04     // 压入 4
CALLDATALOAD   // 读取第一个参数 (2)
PUSH1 0x24     // 压入 36
CALLDATALOAD   // 读取第二个参数 (3)
ADD            // 加法 (2 + 3 = 5)
PUSH1 0x40     // 压入 64
MSTORE         // 存储结果到内存
PUSH1 0x20     // 压入 32
PUSH1 0x40     // 压入 64
RETURN         // 返回结果
```

## 合约调用

### 调用类型

```
合约调用类型

1. CALL
   ├── 最常用的调用方式
   ├── 转账 ETH
   ├── 调用函数
   ├── 独立的执行上下文
   └── 可修改被调用合约的存储

2. DELEGATECALL
   ├── 代理模式的核心
   ├── 使用调用者的存储
   ├── 使用调用者的 msg.sender
   ├── 使用调用者的 msg.value
   └── 用于库合约和代理

3. STATICCALL
   ├── 只读调用
   ├── 不能修改状态
   └── 用于 view/pure 函数

4. CREATE
   └── 创建新合约

5. CREATE2
   ├── 创建新合约
   └── 可预测合约地址
```

### 调用上下文

```
CALL 执行上下文

调用者 (Alice)
     │
     │ CALL
     ▼
合约 A
     │ msg.sender = Alice
     │ msg.value = 1 ETH
     │
     │ CALL
     ▼
合约 B
     │ msg.sender = 合约 A
     │ msg.value = 1 ETH
     │


DELEGATECALL 执行上下文

调用者 (Alice)
     │
     │ CALL
     ▼
代理合约
     │ msg.sender = Alice
     │ msg.value = 1 ETH
     │
     │ DELEGATECALL
     ▼
逻辑合约
     │ msg.sender = Alice (不变!)
     │ msg.value = 1 ETH (不变!)
     │ 使用代理合约的存储
```

### 消息结构

```
msg 对象

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  msg.sender: address                                        │
│  ├── 直接调用者地址                                         │
│  └── DELEGATECALL 时保持不变                                │
│                                                             │
│  msg.value: uint256                                         │
│  ├── 随调用发送的 ETH 数量                                  │
│  └── 以 Wei 为单位                                          │
│                                                             │
│  msg.data: bytes                                            │
│  ├── 完整的调用数据                                         │
│  └── 函数选择器 + 参数                                      │
│                                                             │
│  msg.sig: bytes4                                            │
│  └── 函数选择器 (msg.data 前 4 字节)                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Gas 计算

### Gas 消耗规则

```
Gas 消耗分类

1. 基础费用
   └── 交易基础: 21,000 Gas

2. 计算费用
   ├── 每个操作码有固定 Gas
   └── 示例: ADD = 3, SLOAD = 2100

3. 存储费用
   ├── SSTORE 从零到非零: 20,000
   ├── SSTORE 修改值: 5,000
   └── SSTORE 清零: 退款 15,000

4. 内存扩展费用
   └── 内存大小增加时收费

5. 调用费用
   ├── 基础调用: 700
   ├── 值传递: 9,000
   └── 新账户: 25,000
```

### 优化技巧

```solidity
// ❌ 高 Gas 消耗
contract Bad {
    uint256 public count;
    
    function loop() public {
        for (uint256 i = 0; i < count; i++) {
            // count 每次循环都读取存储
        }
    }
}

// ✓ Gas 优化
contract Good {
    uint256 public count;
    
    function loop() public {
        uint256 _count = count;  // 缓存到内存
        for (uint256 i = 0; i < _count; i++) {
            // 使用内存变量
        }
    }
}

// ❌ 使用存储
contract BadStorage {
    struct User {
        uint256 id;
        string name;
        uint256 score;
    }
    
    mapping(address => User) public users;
}

// ✓ 打包存储
contract GoodStorage {
    struct User {
        uint96 id;      // 12 字节
        uint160 score;  // 20 字节
        // 总共 32 字节，一个槽位
    }
    
    mapping(address => User) public users;
    mapping(address => string) public names;  // 单独存储变长数据
}
```

## EVM 兼容链

```
EVM 兼容链

这些链实现了 EVM，可以运行以太坊智能合约:

Layer 1:
├── BSC (Binance Smart Chain)
├── Avalanche C-Chain
├── Fantom
└── Polygon (侧链)

Layer 2:
├── Arbitrum
├── Optimism
├── Base
├── zkSync
└── StarkNet

优势:
├── 代码复用
├── 工具兼容
├── 用户熟悉
└── 开发效率高
```

## 调试工具

### 1. Tenderly

```
Tenderly 功能:
├── 交易模拟
├── Gas 分析
├── 调试器
└── 错误追踪

使用场景:
├── 分析交易失败原因
├── Gas 优化
├── 安全审计
└── 开发调试
```

### 2. Remix Debugger

```
Remix 内置调试器:
├── 单步执行
├── 查看栈、内存、存储
├── 查看操作码
└── 断点调试
```

## 下一步

[下一节：以太坊网络 →](04-networks.md)
