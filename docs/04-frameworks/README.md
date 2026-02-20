# 第四章：开发框架（Hardhat/Foundry）

## 本章概述

本章将深入介绍以太坊智能合约开发的两大主流框架：**Hardhat** 和 **Foundry**。作为有 Java 和 Rust 背景的开发者，你会发现这两个框架在理念上有很大不同。

## 为什么需要开发框架？

在传统的 Java 开发中，你可能会使用 Maven 或 Gradle 来管理项目依赖、编译代码、运行测试。在 Web3 开发中，开发框架承担着类似的角色，但功能更加丰富：

| 功能 | 传统开发 | Web3 开发 |
|------|----------|-----------|
| 依赖管理 | Maven/Gradle | npm/foundry.toml |
| 编译 | javac | solc |
| 测试 | JUnit | Hardhat/Foundry Test |
| 部署 | 手动/CI/CD | 部署脚本 |
| 本地调试 | 本地运行 | 本地测试网络 |

## Hardhat vs Foundry

### Hardhat

**技术栈**：JavaScript/TypeScript

**优势**：
- 生态系统成熟，插件丰富
- JavaScript/TypeScript 编写测试，前端开发者友好
- 与 Ethers.js、Waffle 等库深度集成
- 调试工具强大（console.log、stack trace）
- 社区活跃，文档完善

**劣势**：
- 测试执行速度相对较慢
- 依赖 Node.js 生态
- TypeScript 类型定义有时不完整

### Foundry

**技术栈**：Rust + Solidity

**优势**：
- 极快的编译和测试速度（比 Hardhat 快 10-100 倍）
- 使用 Solidity 编写测试，无需学习新语言
- 内置模糊测试（Fuzz Testing）
- 用 Rust 编写，性能优异
- 工具链统一（forge、cast、anvil）

**劣势**：
- 相对较新（2022年发布），生态系统不如 Hardhat 成熟
- 部分高级调试功能需要额外配置
- 与现有 JS 工具链集成需要额外工作

### 选择建议

| 场景 | 推荐框架 |
|------|----------|
| 纯前端团队 | Hardhat |
| 有 Rust 背景 | Foundry |
| 追求测试速度 | Foundry |
| 需要丰富的插件 | Hardhat |
| 大型复杂项目 | Foundry + Hardhat 混合使用 |

## 章节目录

1. [Hardhat 环境搭建](./01-hardhat-setup.md)
2. [Hardhat 开发流程](./02-hardhat-development.md)
3. [Foundry 环境搭建](./03-foundry-setup.md)
4. [Foundry 开发流程](./04-foundry-development.md)
5. [测试最佳实践](./05-testing.md)
6. [部署流程](./06-deployment.md)
7. [Hardhat 3 新特性](./07-hardhat3.md) - ESM、Solidity测试、多链支持

## 学习路径建议

作为有 Java 和 Rust 经验的开发者，建议按以下顺序学习：

```
┌─────────────────────────────────────────────────────────────┐
│  第一阶段：Hardhat（熟悉 Web3 开发流程）                      │
│  - JavaScript/TypeScript 环境更接近你熟悉的前端开发          │
│  - 丰富的插件和调试工具帮助你理解智能合约开发                 │
│  - 学习编译、测试、部署的完整流程                            │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  第二阶段：Foundry（发挥 Rust 背景）                         │
│  - 利用你对 Rust 的理解，更好地理解底层原理                  │
│  - 体验极速的开发体验                                        │
│  - 学习 Solidity 测试和模糊测试                              │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│  第三阶段：混合使用                                          │
│  - 用 Foundry 进行快速开发和测试                             │
│  - 用 Hardhat 进行部署和与前端集成                           │
│  - 发挥两个框架各自的优势                                    │
└─────────────────────────────────────────────────────────────┘
```

## 类比理解

如果你熟悉 Java 生态：

| 概念 | Java | Hardhat | Foundry |
|------|------|---------|---------|
| 构建工具 | Maven/Gradle | npm + hardhat | forge |
| 测试框架 | JUnit | Mocha/Chai | Solidity test |
| 包管理 | Maven Central | npm registry | git submodules |
| 本地运行 | Application Server | Hardhat Network | Anvil |

如果你熟悉 Rust 生态：

| 概念 | Rust | Hardhat | Foundry |
|------|------|---------|---------|
| 构建工具 | cargo | npm + hardhat | forge |
| 测试 | cargo test | npm test | forge test |
| 包管理 | crates.io | npm | crates.io/git |
| 本地运行 | cargo run | npx hardhat node | anvil |

## 实践项目

本章将使用一个简单的代币合约作为示例，完整体验两个框架的开发流程：

```solidity
// 简化的 ERC20 代币合约
contract SimpleToken {
    string public name;
    string public symbol;
    uint8 public decimals = 18;
    uint256 public totalSupply;
    
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    constructor(string memory _name, string memory _symbol, uint256 _initialSupply) {
        name = _name;
        symbol = _symbol;
        totalSupply = _initialSupply * 10 ** uint256(decimals);
        balanceOf[msg.sender] = totalSupply;
    }
    
    function transfer(address to, uint256 value) public returns (bool) {
        require(balanceOf[msg.sender] >= value, "Insufficient balance");
        require(to != address(0), "Invalid recipient");
        
        balanceOf[msg.sender] -= value;
        balanceOf[to] += value;
        
        emit Transfer(msg.sender, to, value);
        return true;
    }
    
    function approve(address spender, uint256 value) public returns (bool) {
        allowance[msg.sender][spender] = value;
        emit Approval(msg.sender, spender, value);
        return true;
    }
    
    function transferFrom(address from, address to, uint256 value) public returns (bool) {
        require(value <= balanceOf[from], "Insufficient balance");
        require(value <= allowance[from][msg.sender], "Insufficient allowance");
        require(to != address(0), "Invalid recipient");
        
        balanceOf[from] -= value;
        balanceOf[to] += value;
        allowance[from][msg.sender] -= value;
        
        emit Transfer(from, to, value);
        return true;
    }
}
```

让我们开始学习这两个强大的开发框架吧！
