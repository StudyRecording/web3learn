# 测试最佳实践

本章介绍智能合约测试的最佳实践，包括单元测试、模糊测试、不变量测试等。

## 测试的重要性

智能合约一旦部署就无法修改，bug 可能导致巨额损失。与 Java 不同：

| 特性 | Java | Solidity |
|------|------|----------|
| 代码更新 | 可以热更新/重新部署 | 不可变（除非使用代理） |
| 漏洞影响 | 数据泄露 | 资金丢失 |
| 测试重点 | 功能正确性 | 功能 + 安全 + Gas |

## 测试金字塔

```
                    ┌─────────┐
                    │  E2E    │  ← 少量集成测试
                   ╱└─────────┘╲
                  ╱  ┌─────────┐ ╲
                 ╱   │  模糊   │  ╲ ← 中量模糊测试
                ╱    └─────────┘   ╲
               ╱     ┌─────────┐    ╲
              ╱      │  单元   │     ╲ ← 大量单元测试
             ╱       └─────────┘      ╲
            ╱        ┌─────────┐       ╲
           ╱         │  快速   │        ╲
          ╱          └─────────┘         ╲
```

## 单元测试

### 测试结构

```solidity
// test/SimpleToken.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "../src/SimpleToken.sol";

contract SimpleTokenTest is Test {
    // ===== 状态变量 =====
    SimpleToken token;
    address owner;
    address alice;
    address bob;
    
    // ===== 事件（用于测试） =====
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    // ===== Setup =====
    function setUp() public {
        owner = address(this);
        alice = makeAddr("alice");
        bob = makeAddr("bob");
        
        token = new SimpleToken("Test Token", "TST", 1_000_000);
    }
    
    // ===== 部署测试 =====
    
    function test_Deployment() public view {
        // 基本属性
        assertEq(token.name(), "Test Token", "Name should match");
        assertEq(token.symbol(), "TST", "Symbol should match");
        assertEq(token.decimals(), 18, "Decimals should be 18");
        
        // 供应量
        assertEq(token.totalSupply(), 1_000_000 * 10 ** 18);
        
        // 初始分配
        assertEq(token.balanceOf(owner), 1_000_000 * 10 ** 18);
    }
    
    // ===== 功能测试 =====
    
    function test_Transfer() public {
        uint256 amount = 100 * 10 ** 18;
        
        token.transfer(alice, amount);
        
        assertEq(token.balanceOf(alice), amount);
        assertEq(token.balanceOf(owner), 1_000_000 * 10 ** 18 - amount);
    }
    
    function test_Approve() public {
        uint256 amount = 500 * 10 ** 18;
        
        token.approve(alice, amount);
        
        assertEq(token.allowance(owner, alice), amount);
    }
    
    function test_TransferFrom() public {
        uint256 amount = 100 * 10 ** 18;
        
        // 授权
        token.approve(alice, amount);
        
        // 从 alice 地址调用 transferFrom
        vm.prank(alice);
        token.transferFrom(owner, bob, amount);
        
        // 验证
        assertEq(token.balanceOf(bob), amount);
        assertEq(token.allowance(owner, alice), 0);
    }
    
    // ===== 边界测试 =====
    
    function test_TransferZeroAmount() public {
        uint256 balanceBefore = token.balanceOf(alice);
        
        token.transfer(alice, 0);
        
        assertEq(token.balanceOf(alice), balanceBefore);
    }
    
    function test_TransferFullBalance() public {
        uint256 balance = token.balanceOf(owner);
        
        token.transfer(alice, balance);
        
        assertEq(token.balanceOf(owner), 0);
        assertEq(token.balanceOf(alice), balance);
    }
    
    // ===== 错误测试 =====
    
    function test_RevertWhen_TransferToZeroAddress() public {
        vm.expectRevert("Invalid recipient");
        token.transfer(address(0), 100);
    }
    
    function test_RevertWhen_TransferExceedsBalance() public {
        vm.prank(alice);
        vm.expectRevert("Insufficient balance");
        token.transfer(bob, 1);
    }
    
    function test_RevertWhen_TransferFromExceedsAllowance() public {
        token.approve(alice, 100);
        
        vm.prank(alice);
        vm.expectRevert("Insufficient allowance");
        token.transferFrom(owner, bob, 101);
    }
    
    // ===== 事件测试 =====
    
    function test_TransferEmitsEvent() public {
        uint256 amount = 100 * 10 ** 18;
        
        vm.expectEmit(true, true, false, true);
        emit Transfer(owner, alice, amount);
        
        token.transfer(alice, amount);
    }
    
    function test_ApproveEmitsEvent() public {
        uint256 amount = 100 * 10 ** 18;
        
        vm.expectEmit(true, true, false, true);
        emit Approval(owner, alice, amount);
        
        token.approve(alice, amount);
    }
}
```

### 测试分组

```solidity
contract GroupedTest is Test {
    SimpleToken token;
    
    function setUp() public {
        token = new SimpleToken("Test", "TST", 1_000_000);
    }
    
    // 使用 modifier 分组测试
    modifier withFundedAccount(address account, uint256 amount) {
        token.transfer(account, amount);
        _;
    }
    
    function test_Transfer_WhenFunded() 
        public 
        withFundedAccount(alice, 1000) 
    {
        vm.prank(alice);
        token.transfer(bob, 500);
        
        assertEq(token.balanceOf(alice), 500);
        assertEq(token.balanceOf(bob), 500);
    }
    
    // 使用内部函数组织测试
    function _deployAndFund(address to, uint256 amount) internal returns (SimpleToken) {
        SimpleToken t = new SimpleToken("Test", "TST", amount);
        t.transfer(to, amount);
        return t;
    }
}
```

### 自定义错误测试

```solidity
// 自定义错误定义
error InsufficientBalance(uint256 balance, uint256 needed);
error InsufficientAllowance(uint256 allowance, uint256 needed);

contract CustomErrorTest is Test {
    function test_RevertWith_CustomError() public {
        uint256 needed = 100;
        
        vm.expectRevert(
            abi.encodeWithSelector(
                InsufficientBalance.selector,
                0,  // 当前余额
                needed
            )
        );
        
        vm.prank(alice);
        token.transfer(bob, needed);
    }
}
```

## 模糊测试（Fuzz Testing）

### 基础模糊测试

```solidity
contract FuzzTest is Test {
    MathLibrary math;
    
    function setUp() public {
        math = new MathLibrary();
    }
    
    // 自动生成随机输入
    function testFuzz_Add(uint256 a, uint256 b) public pure {
        uint256 result = math.add(a, b);
        
        if (b <= type(uint256).max - a) {
            assertEq(result, a + b);
        }
    }
    
    // 约束输入范围
    function testFuzz_Multiply(uint256 a, uint256 b) public pure {
        vm.assume(b == 0 || a <= type(uint256).max / b);
        
        uint256 result = math.mul(a, b);
        assertEq(result, a * b);
    }
    
    // 使用 bound 约束
    function testFuzz_BoundedValue(uint256 x) public pure {
        // 将 x 约束到 [1, 1000]
        uint256 bounded = bound(x, 1, 1000);
        
        assertGe(bounded, 1);
        assertLe(bounded, 1000);
    }
}
```

### 模糊测试策略

```solidity
contract FuzzStrategyTest is Test {
    Vault vault;
    
    function setUp() public {
        vault = new Vault();
    }
    
    // 策略 1: 约束无效输入
    function testFuzz_Deposit(uint256 amount) public {
        vm.assume(amount > 0);
        vm.assume(amount <= 1_000_000 * 10 ** 18);  // 最大值
        
        vault.deposit{value: amount}();
        assertEq(vault.balanceOf(address(this)), amount);
    }
    
    // 策略 2: 测试不变量
    function testFuzz_TotalValueConserved(
        uint256 depositAmount,
        uint256 withdrawAmount
    ) public {
        depositAmount = bound(depositAmount, 1, 100 ether);
        
        vault.deposit{value: depositAmount}();
        
        withdrawAmount = bound(withdrawAmount, 0, vault.balanceOf(address(this)));
        vault.withdraw(withdrawAmount);
        
        uint256 remaining = vault.balanceOf(address(this));
        uint256 contractBalance = address(vault).balance;
        
        // 不变量：合约余额 >= 账户余额
        assertGe(contractBalance, remaining);
    }
    
    // 策略 3: 测试状态转换
    function testFuzz_StateTransitions(uint8 action, uint256 amount) public {
        amount = bound(amount, 1, 100 ether);
        
        if (action % 2 == 0) {
            // 存款
            vault.deposit{value: amount}();
            assertGe(vault.balanceOf(address(this)), 0);
        } else {
            // 先存款再取款
            vault.deposit{value: amount}();
            vault.withdraw(amount / 2);
        }
    }
    
    // 策略 4: 测试多个参数
    function testFuzz_MultipleUsers(
        address user1,
        address user2,
        uint256 amount1,
        uint256 amount2
    ) public {
        vm.assume(user1 != user2);
        vm.assume(user1 != address(0));
        vm.assume(user2 != address(0));
        
        amount1 = bound(amount1, 1, 100 ether);
        amount2 = bound(amount2, 1, 100 ether);
        
        vm.deal(user1, amount1);
        vm.deal(user2, amount2);
        
        vm.prank(user1);
        vault.deposit{value: amount1}();
        
        vm.prank(user2);
        vault.deposit{value: amount2}();
        
        assertEq(vault.totalSupply(), amount1 + amount2);
    }
}
```

### 模糊测试配置

```toml
# foundry.toml
[profile.default.fuzz]
runs = 256            # 默认运行次数
max_test_rejects = 65536  # 最大拒绝次数
seed = '0x...'        # 固定随机种子（可复现）

[profile.ci.fuzz]
runs = 10000          # CI 中运行更多次
```

## 不变量测试（Invariant Testing）

不变量测试持续调用随机函数，确保系统核心属性始终成立。

### 基础不变量测试

```solidity
// test/invariant/VaultInvariantTest.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "../../src/Vault.sol";

contract VaultHandler is Test {
    Vault public vault;
    
    constructor(Vault _vault) {
        vault = _vault;
    }
    
    // 可被不变量测试调用的函数
    function deposit(uint256 amount) public {
        amount = bound(amount, 0, address(this).balance);
        if (amount > 0) {
            vault.deposit{value: amount}();
        }
    }
    
    function withdraw(uint256 amount) public {
        amount = bound(amount, 0, vault.balanceOf(address(this)));
        if (amount > 0) {
            vault.withdraw(amount);
        }
    }
    
    receive() external payable {}
}

contract VaultInvariantTest is Test {
    Vault vault;
    VaultHandler handler;
    
    function setUp() public {
        vault = new Vault();
        handler = new VaultHandler(vault);
        
        // 给 handler 资金
        vm.deal(address(handler), 100 ether);
    }
    
    // 指定要测试的合约
    function targetContracts() public view returns (address[] memory) {
        address[] memory targets = new address[](1);
        targets[0] = address(handler);
        return targets;
    }
    
    // 指定要调用的函数
    function targetSelectors() public view returns (bytes4[] memory) {
        bytes4[] memory selectors = new bytes4[](2);
        selectors[0] = handler.deposit.selector;
        selectors[1] = handler.withdraw.selector;
        return selectors;
    }
    
    // 不变量：合约余额 >= 总供应量
    function invariant_TotalSupplyLessThanBalance() public view {
        assertLe(vault.totalSupply(), address(vault).balance);
    }
    
    // 不变量：每个用户的份额余额 >= 0
    function invariant_BalancesNonNegative() public view {
        assertGe(vault.balanceOf(address(handler)), 0);
    }
}
```

### 复杂不变量测试

```solidity
// test/invariant/DeFiInvariantTest.t.sol
contract DeFiHandler is Test {
    LendingPool pool;
    Token token;
    
    uint256 public totalDeposited;
    uint256 public totalBorrowed;
    
    constructor(LendingPool _pool, Token _token) {
        pool = _pool;
        token = _token;
    }
    
    function deposit(uint256 amount) public {
        amount = bound(amount, 0, token.balanceOf(address(this)));
        if (amount > 0) {
            token.approve(address(pool), amount);
            pool.deposit(amount);
            totalDeposited += amount;
        }
    }
    
    function borrow(uint256 amount) public {
        uint256 maxBorrow = pool.getMaxBorrow(address(this));
        amount = bound(amount, 0, maxBorrow);
        if (amount > 0) {
            pool.borrow(amount);
            totalBorrowed += amount;
        }
    }
    
    function repay(uint256 amount) public {
        uint256 debt = pool.getDebt(address(this));
        amount = bound(amount, 0, debt);
        if (amount > 0) {
            token.approve(address(pool), amount);
            pool.repay(amount);
            totalBorrowed -= amount;
        }
    }
    
    function withdraw(uint256 amount) public {
        amount = bound(amount, 0, pool.getDeposit(address(this)));
        if (amount > 0) {
            pool.withdraw(amount);
            totalDeposited -= amount;
        }
    }
}

contract DeFiInvariantTest is Test {
    LendingPool pool;
    Token token;
    DeFiHandler handler;
    
    function setUp() public {
        token = new Token("Test", "TST", 1_000_000 * 10 ** 18);
        pool = new LendingPool(address(token));
        handler = new DeFiHandler(pool, token);
        
        token.transfer(address(handler), 100_000 * 10 ** 18);
    }
    
    function targetContracts() public view returns (address[] memory) {
        address[] memory targets = new address[](1);
        targets[0] = address(handler);
        return targets;
    }
    
    // 不变量：总借款 <= 总存款 * 抵押率
    function invariant_Solvency() public view {
        uint256 totalDeposited = handler.totalDeposited();
        uint256 totalBorrowed = handler.totalBorrowed();
        
        if (totalDeposited > 0) {
            uint256 maxBorrow = totalDeposited * 70 / 100;  // 70% LTV
            assertLe(totalBorrowed, maxBorrow);
        }
    }
    
    // 不变量：池子有足够流动性
    function invariant_Liquidity() public view {
        uint256 poolBalance = token.balanceOf(address(pool));
        assertGe(poolBalance, 0);
    }
}
```

## Hardhat 测试最佳实践

### 使用 Waffle 风格断言

```javascript
// test/SimpleToken.js
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("SimpleToken", function () {
  let token, owner, addr1, addr2;

  beforeEach(async function () {
    [owner, addr1, addr2] = await ethers.getSigners();
    const Token = await ethers.getContractFactory("SimpleToken");
    token = await Token.deploy("Test", "TST", 1000000);
    await token.waitForDeployment();
  });

  describe("Transfers", function () {
    it("Should transfer tokens correctly", async function () {
      await token.transfer(addr1.address, 100);
      expect(await token.balanceOf(addr1.address)).to.equal(100);
    });

    it("Should emit Transfer event", async function () {
      await expect(token.transfer(addr1.address, 100))
        .to.emit(token, "Transfer")
        .withArgs(owner.address, addr1.address, 100);
    });

    it("Should revert on insufficient balance", async function () {
      await expect(
        token.connect(addr1).transfer(addr2.address, 100)
      ).to.be.revertedWith("Insufficient balance");
    });

    it("Should revert on transfer to zero address", async function () {
      await expect(
        token.transfer(ethers.ZeroAddress, 100)
      ).to.be.revertedWith("Invalid recipient");
    });
  });

  describe("Approvals", function () {
    it("Should approve spending", async function () {
      await token.approve(addr1.address, 1000);
      expect(
        await token.allowance(owner.address, addr1.address)
      ).to.equal(1000);
    });

    it("Should emit Approval event", async function () {
      await expect(token.approve(addr1.address, 1000))
        .to.emit(token, "Approval")
        .withArgs(owner.address, addr1.address, 1000);
    });
  });
});
```

### 测试时间相关功能

```javascript
const { time } = require("@nomicfoundation/hardhat-network-helpers");

describe("Time-locked functionality", function () {
  it("Should allow withdrawal after lock period", async function () {
    const ONE_WEEK = 7 * 24 * 60 * 60;
    
    const lock = await Lock.deploy(ONE_WEEK);
    
    // 快进一周
    await time.increase(ONE_WEEK);
    
    await expect(lock.withdraw()).to.not.be.reverted;
  });

  it("Should not allow withdrawal before lock period", async function () {
    const lock = await Lock.deploy(7 * 24 * 60 * 60);
    
    await expect(lock.withdraw()).to.be.reverted;
  });
});
```

### 测试 Reentrancy

```javascript
describe("Reentrancy protection", function () {
  it("Should prevent reentrancy attacks", async function () {
    const bank = await Bank.deploy();
    const attacker = await Attacker.deploy(bank.address);
    
    // 存入资金
    await bank.deposit({ value: ethers.parseEther("10") });
    
    // 尝试攻击
    await expect(
      attacker.attack({ value: ethers.parseEther("1") })
    ).to.be.reverted;
  });
});
```

## 测试覆盖率

### Foundry 覆盖率

```bash
# 生成覆盖率报告
forge coverage

# 输出到文件
forge coverage --report lcov

# 使用 lcov 可视化
genhtml lcov.info --output-directory coverage
```

### Hardhat 覆盖率

```bash
# 安装
bun add --dev solidity-coverage

# 运行
bunx hardhat coverage
```

## 测试模板

### 完整测试模板

```solidity
// test/ContractTemplate.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "../src/Contract.sol";

contract ContractTest is Test {
    Contract target;
    
    // 测试账户
    address owner;
    address alice;
    address bob;
    
    // 事件
    event EventName(address indexed from, uint256 amount);
    
    // 错误
    error CustomError(uint256 code);
    
    function setUp() public {
        owner = address(this);
        alice = makeAddr("alice");
        bob = makeAddr("bob");
        
        target = new Contract();
    }
    
    // ========== 部署测试 ==========
    
    function test_Deployment() public view {
        // 验证初始状态
    }
    
    // ========== 功能测试 ==========
    
    function test_FunctionName() public {
        // 测试正常流程
    }
    
    function testFuzz_FunctionName(uint256 input) public {
        // 模糊测试
        vm.assume(input > 0);
        
        // 测试逻辑
    }
    
    // ========== 边界测试 ==========
    
    function test_EdgeCase_Zero() public {
        // 测试零值
    }
    
    function test_EdgeCase_Max() public {
        // 测试最大值
    }
    
    // ========== 错误测试 ==========
    
    function test_RevertWhen_Condition() public {
        vm.expectRevert("Error message");
        target.functionThatReverts();
    }
    
    function test_RevertWith_CustomError() public {
        vm.expectRevert(
            abi.encodeWithSelector(CustomError.selector, 1)
        );
        target.functionWithCustomError();
    }
    
    // ========== 事件测试 ==========
    
    function test_Emits_EventName() public {
        vm.expectEmit(true, true, false, true);
        emit EventName(alice, 100);
        
        target.functionThatEmitsEvent();
    }
    
    // ========== 辅助函数 ==========
    
    function _setupState() internal {
        // 设置测试状态
    }
}
```

## 测试清单

部署前的测试清单：

- [ ] 所有公共函数都有测试
- [ ] 所有错误路径都有测试
- [ ] 边界值（0、最大值）有测试
- [ ] 模糊测试覆盖核心功能
- [ ] 不变量测试覆盖关键属性
- [ ] 事件正确触发
- [ ] 权限控制正确
- [ ] Reentrancy 防护
- [ ] 整数溢出检查
- [ ] 测试覆盖率 > 90%

## 下一步

测试是开发的核心，继续学习 [部署流程](./06-deployment.md)，了解如何安全地将合约部署到生产环境。
