# 第三章：Solidity 智能合约开发

> 掌握以太坊智能合约编程语言

## 本章目标

- 掌握 Solidity 语法基础
- 理解合约结构与组织方式
- 学会继承与组合模式
- 掌握高级特性与安全实践
- 熟悉主流代币标准

## 目录

1. [语法基础](01-syntax.md) - 数据类型、变量、函数、控制流
2. [合约结构](02-contract-structure.md) - 状态变量、函数、事件、修饰器
3. [继承与组合](03-inheritance.md) - is关键字、抽象合约、接口
4. [高级特性](04-advanced.md) - payable、fallback、receive、错误处理
5. [安全实践](05-security.md) - 常见漏洞、防护措施、最佳实践
6. [代币标准](06-standards.md) - ERC-20、ERC-721、ERC-1155
7. [新特性](07-new-features.md) - 瞬态存储、unchecked循环、废弃特性

## 学习时间

预计 5-7 天

## 前置知识

- 第一章：Web3 基础概念
- 第二章：以太坊基础
- 任意编程语言基础（Java/Rust 最佳）

## 核心要点

### Solidity 是什么

Solidity 是以太坊上最流行的**静态类型**智能合约编程语言，语法类似 JavaScript，但类型系统更严谨。

```
┌─────────────────────────────────────────────────────────────┐
│                   Solidity 开发流程                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  编写合约 (.sol)                                            │
│       │                                                     │
│       ▼                                                     │
│  编译 (solc/hardhat/foundry)                                │
│       │                                                     │
│       ├──► ABI (接口描述)                                   │
│       └──► Bytecode (部署字节码)                            │
│                                                             │
│  部署到网络                                                  │
│       │                                                     │
│       ▼                                                     │
│  合约地址 (不可变)                                          │
│                                                             │
│  交互调用                                                    │
│       │                                                     │
│       └──► 通过 ABI 调用合约方法                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Solidity vs Java 对比

| 特性 | Java | Solidity |
|------|------|----------|
| 类型系统 | 静态类型 | 静态类型 |
| 内存管理 | GC 自动回收 | Storage/Memory/Calldata |
| 继承 | 单继承 + 接口 | 多继承 |
| 访问修饰符 | public/private/protected | public/private/internal/external |
| 错误处理 | try-catch | require/revert/assert |
| 事件 | 无原生支持 | event + emit |
| 货币处理 | BigDecimal | uint256 + ether 关键字 |
| 部署 | Jar 包 | 字节码上链 |

### 开发环境搭建

```bash
# 推荐使用 Foundry (Rust 编写，速度极快)
curl -L https://foundry.paradigm.xyz | bash
foundryup

# 或使用 Hardhat (Node.js 生态)
npm init -y
npm install --save-dev hardhat
npx hardhat init
```

### 第一个合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract HelloWorld {
    string public message;
    
    constructor(string memory _message) {
        message = _message;
    }
    
    function setMessage(string memory _newMessage) public {
        message = _newMessage;
    }
}
```

对比 Java：

```java
// Java 等价代码
public class HelloWorld {
    private String message;
    
    public HelloWorld(String message) {
        this.message = message;
    }
    
    public String getMessage() {
        return message;
    }
    
    public void setMessage(String newMessage) {
        this.message = newMessage;
    }
}
```

### 关键差异速览

```
┌─────────────────────────────────────────────────────────────┐
│              Java vs Solidity 核心差异                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 运行环境                                                 │
│     Java:   JVM (中心化服务器)                              │
│     Solidity: EVM (去中心化区块链)                          │
│                                                             │
│  2. 状态持久化                                               │
│     Java:   数据库 (可修改删除)                             │
│     Solidity: 链上存储 (永久保存)                           │
│                                                             │
│  3. 代码更新                                                 │
│     Java:   随时重新部署                                    │
│     Solidity: 不可变 (需代理模式)                           │
│                                                             │
│  4. 执行成本                                                 │
│     Java:   服务器成本                                      │
│     Solidity: Gas 费用 (每次操作都要付费)                   │
│                                                             │
│  5. 安全模型                                                 │
│     Java:   用户输入验证                                    │
│     Solidity: 所有调用都是恶意的                            │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 开始学习

[下一节：语法基础 →](01-syntax.md)
