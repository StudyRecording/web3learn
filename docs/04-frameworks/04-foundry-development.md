# Foundry 开发流程

本章介绍使用 Foundry 进行智能合约开发的完整流程，包括使用 Solidity 编写测试、模糊测试、部署等。

## 编译合约

### 基础编译

```bash
forge build
```

输出：

```
[⠒] Compiling...
[⠊] Compiling 10 files with 0.8.24
[⠔] Solc 0.8.24 finished in 2.34s
```

### 编译选项

```bash
# 强制重新编译
forge build --force

# 详细输出
forge build -vvv

# 指定 solc 版本
forge build --use 0.8.24

# 只检查语法
forge build --check

# 输出特定信息
forge build --extra-output abi,metadata,ir

# 查看合约大小
forge build --sizes
```

### 查看编译产物

```bash
# 查看 ABI
forge inspect SimpleToken abi

# 查看 bytecode
forge inspect SimpleToken bytecode

# 查看部署的 bytecode
forge inspect SimpleToken deployedBytecode

# 查看存储布局
forge inspect SimpleToken storage-layout

# 查看所有可用字段
forge inspect SimpleToken --list
```

## 测试合约

Foundry 的最大特点是使用 Solidity 编写测试！

### 基础测试示例

```solidity
// test/SimpleToken.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Test.sol";
import "../src/SimpleToken.sol";

contract SimpleTokenTest is Test {
    SimpleToken public token;
    
    address public owner = address(0x1);
    address public alice = address(0x2);
    address public bob = address(0x3);
    
    function setUp() public {
        vm.prank(owner);
        token = new SimpleToken("My Token", "MTK", 1_000_000 * 10 ** 18);
    }
    
    // 测试部署
    function test_Deployment() public view {
        assertEq(token.name(), "My Token");
        assertEq(token.symbol(), "MTK");
        assertEq(token.decimals(), 18);
        assertEq(token.totalSupply(), 1_000_000 * 10 ** 18);
        assertEq(token.balanceOf(owner), 1_000_000 * 10 ** 18);
    }
    
    // 测试转账
    function test_Transfer() public {
        vm.prank(owner);
        token.transfer(alice, 100 * 10 ** 18);
        
        assertEq(token.balanceOf(alice), 100 * 10 ** 18);
        assertEq(token.balanceOf(owner), 999_900 * 10 ** 18);
    }
    
    // 测试余额不足
    function test_RevertWhen_TransferWithoutBalance() public {
        vm.prank(alice);
        vm.expectRevert("Insufficient balance");
        token.transfer(bob, 1);
    }
    
    // 测试事件
    function test_TransferEvent() public {
        vm.expectEmit(true, true, false, true);
        emit IERC20.Transfer(owner, alice, 100);
        
        vm.prank(owner);
        token.transfer(alice, 100);
    }
    
    // 测试授权
    function test_Approve() public {
        vm.prank(owner);
        token.approve(alice, 1000);
        
        assertEq(token.allowance(owner, alice), 1000);
    }
    
    // 测试 transferFrom
    function test_TransferFrom() public {
        vm.startPrank(owner);
        token.approve(alice, 500);
        vm.stopPrank();
        
        vm.prank(alice);
        token.transferFrom(owner, bob, 300);
        
        assertEq(token.balanceOf(bob), 300);
        assertEq(token.allowance(owner, alice), 200);
    }
}
```

### forge-std 测试工具

```solidity
import "forge-std/Test.sol";

contract TestUtils is Test {
    // ===== 断言 =====
    
    function test_Assertions() public pure {
        // 相等
        assertEq(1, 1);
        assertEq(address(0), address(0));
        assertEq(bytes32(0), bytes32(0));
        assertEq(keccak256("a"), keccak256("a"));
        
        // 布尔
        assertTrue(true);
        assertFalse(false);
        
        // 比较
        assertGt(2, 1);
        assertGe(2, 2);
        assertLt(1, 2);
        assertLe(1, 1);
        
        // 接近相等
        assertApproxEqAbs(100, 101, 2);  // 差值 <= 2
        assertApproxEqRel(100, 101, 0.02e18);  // 相对误差 <= 2%
    }
    
    // ===== 消息断言 =====
    
    function test_AssertWithMessage() public pure {
        assertEq(1, 1, "Values should be equal");
    }
    
    // ===== Revert 测试 =====
    
    function test_ExpectRevert() public {
        // 期望下个调用 revert
        vm.expectRevert();
        someFunctionThatReverts();
        
        // 期望特定错误消息
        vm.expectRevert("Insufficient balance");
        transferWithoutBalance();
        
        // 期望自定义错误
        vm.expectRevert(CustomError.selector);
        functionWithCustomError();
        
        // 期望带参数的自定义错误
        vm.expectRevert(abi.encodeWithSelector(
            CustomError.selector,
            100
        ));
    }
    
    // ===== 事件测试 =====
    
    function test_ExpectEmit() public {
        // 参数: checkTopic1, checkTopic2, checkTopic3, checkData
        vm.expectEmit(true, true, false, true);
        
        // 期望的事件
        emit Transfer(msg.sender, address(1), 100);
        
        // 执行应该触发事件的函数
        token.transfer(address(1), 100);
    }
    
    // ===== 控制区块 =====
    
    function test_BlockManipulation() public {
        // 获取当前区块号
        uint256 blockNum = block.number;
        
        // 前进区块
        vm.roll(blockNum + 100);
        assertEq(block.number, blockNum + 100);
        
        // 获取当前时间戳
        uint256 timestamp = block.timestamp;
        
        // 前进时间
        vm.warp(timestamp + 1 days);
        assertEq(block.timestamp, timestamp + 1 days);
        
        // 同时前进区块和时间
        skip(3600);  // 前进 1 小时
        rewind(1800);  // 后退 30 分钟
    }
    
    // ===== 账户管理 =====
    
    function test_AccountManagement() public {
        // 设置 msg.sender
        vm.prank(alice);
        // 下一次调用的 msg.sender 是 alice
        
        // 持续设置 msg.sender
        vm.startPrank(alice);
        // 多次调用的 msg.sender 都是 alice
        vm.stopPrank();
        
        // 设置余额
        deal(alice, 100 ether);
        assertEq(alice.balance, 100 ether);
        
        // 设置 ERC20 余额
        deal(address(token), alice, 1000 * 10 ** 18);
        
        // 创建新账户
        address newAccount = makeAddr("newAccount");
        
        // 使用私钥签名
        uint256 privateKey = 0xA11CE;
        address signer = vm.addr(privateKey);
    }
    
    // ===== 存储操作 =====
    
    function test_StorageManipulation() public {
        // 读取存储槽
        bytes32 value = vm.load(address(token), 0);
        
        // 写入存储槽
        vm.store(address(token), 0, bytes32(uint256(100)));
    }
    
    // ===== 记录和断言 =====
    
    function test_Recording() public {
        // 记录存储变化
        vm.record();
        // ... 执行一些操作 ...
        (bytes32[] memory reads, bytes32[] memory writes) = vm.accesses(
            address(token)
        );
        
        // 记录日志
        emit log("Simple message");
        emit log_named_uint("Balance", 100);
        emit log_named_address("Owner", owner);
        emit log_named_bytes32("Hash", keccak256("data"));
    }
}
```

### Cheatcodes 完整参考

Foundry 提供的 `vm` 对象（Cheatcodes）：

```solidity
interface Vm {
    // ===== 消息和调用 =====
    
    // 设置下一次调用的 msg.sender
    function prank(address) external;
    // 设置 msg.sender，持续到 stopPrank
    function startPrank(address) external;
    function stopPrank() external;
    
    // ===== 账户 =====
    
    // 设置账户余额
    function deal(address, uint256) external;
    // 生成随机地址
    function addr(uint256 privateKey) external pure returns (address);
    
    // ===== 区块和时间 =====
    
    // 设置区块号
    function roll(uint256) external;
    // 设置时间戳
    function warp(uint256) external;
    // 快进时间
    function skip(uint256) external;
    function rewind(uint256) external;
    
    // ===== 存储和状态 =====
    
    // 读取存储槽
    function load(address, bytes32) external returns (bytes32);
    // 写入存储槽
    function store(address, bytes32, bytes32) external;
    // 清除所有存储
    function clearMockedCalls() external;
    
    // ===== 日志和事件 =====
    
    function expectEmit(bool, bool, bool, bool) external;
    function expectEmit(address) external;
    
    // ===== 错误和 Revert =====
    
    function expectRevert() external;
    function expectRevert(bytes calldata) external;
    function expectRevert(bytes4) external;
    
    // ===== 调用 =====
    
    // Mock 调用
    function mockCall(address, bytes calldata, bytes calldata) external;
    function mockCallRevert(address, bytes calldata, bytes calldata) external;
    
    // ===== 快照 =====
    
    function snapshot() external returns (uint256);
    function revertTo(uint256) external returns (bool);
    
    // ===== 其他 =====
    
    // 设置 gas 价格
    function fee(uint256) external;
    // 设置 chain ID
    function chainId(uint256) external;
    // 设置 coinbase
    function coinbase(address) external;
    // 设置难度
    function difficulty(uint256) external;
    // 录制存储访问
    function record() external;
    function accesses(address) external returns (bytes32[] memory, bytes32[] memory);
}
```

### 运行测试

```bash
# 运行所有测试
forge test

# 详细输出
forge test -v
forge test -vv
forge test -vvv
forge test -vvvv  # 最详细

# 运行特定测试
forge test --match-test testTransfer
forge test --match-contract SimpleTokenTest
forge test --match-path test/SimpleToken.t.sol

# 使用正则表达式
forge test --match-test "test.*Transfer"

# 显示 gas 报告
forge test --gas-report

# 运行模糊测试
forge test --fuzz

# 并行测试
forge test --threads 4

# 指定测试目录
forge test --root test

# 只显示失败的测试
forge test --fail-only

# 生成测试覆盖率报告
forge coverage
```

输出示例：

```
Running 15 tests for test/SimpleToken.t.sol:SimpleTokenTest
[PASS] test_Approve() (gas: 35487)
[PASS] test_Deployment() (gas: 20834)
[PASS] test_Transfer() (gas: 51023)
[PASS] test_TransferEvent() (gas: 61021)
[FAIL. Reason: Revert] test_TransferWithoutBalance() (gas: 12034)
Test result: FAILED. 4 passed; 1 failed; 0 skipped; finished in 1.23s
```

## 模糊测试（Fuzz Testing）

Foundry 内置强大的模糊测试支持，自动生成随机输入测试边界情况。

### 基础模糊测试

```solidity
contract FuzzTest is Test {
    SimpleToken token;
    
    function setUp() public {
        token = new SimpleToken("Test", "TST", 1_000_000 * 10 ** 18);
    }
    
    // 任何带参数的测试函数都会自动进行模糊测试
    function testFuzz_Transfer(uint256 amount) public {
        vm.assume(amount > 0 && amount <= token.balanceOf(address(this)));
        
        address to = address(0x1);
        uint256 balanceBefore = token.balanceOf(to);
        
        token.transfer(to, amount);
        
        assertEq(token.balanceOf(to), balanceBefore + amount);
    }
    
    // 多个参数
    function testFuzz_TransferMultiple(
        address to,
        uint256 amount
    ) public {
        vm.assume(to != address(0));
        vm.assume(amount <= token.balanceOf(address(this)));
        
        token.transfer(to, amount);
        
        assertEq(token.balanceOf(to), amount);
    }
    
    // 复杂参数
    struct TransferParams {
        address to;
        uint256 amount;
    }
    
    function testFuzz_ComplexInput(TransferParams memory params) public {
        vm.assume(params.to != address(0));
        vm.assume(params.amount <= token.balanceOf(address(this)));
        
        token.transfer(params.to, params.amount);
    }
}
```

### 约束输入

```solidity
contract FuzzConstraintsTest is Test {
    // 使用 vm.assume 过滤输入
    function testFuzz_WithConstraints(uint256 x) public {
        // 确保满足条件，否则重新生成输入
        vm.assume(x > 0);
        vm.assume(x < 1000);
        
        assertGt(x, 0);
        assertLt(x, 1000);
    }
    
    // 边界值测试
    function testFuzz_Bound(uint256 x) public pure {
        // 将 x 约束到 [10, 100] 范围
        uint256 bounded = bound(x, 10, 100);
        
        assertGe(bounded, 10);
        assertLe(bounded, 100);
    }
    
    // 地址约束
    function testFuzz_AddressConstraints(address addr) public pure {
        vm.assume(addr != address(0));
        vm.assume(addr.code.length == 0);  // 确保是 EOA
    }
    
    // 复杂约束
    function testFuzz_ComplexConstraints(
        uint256 amount,
        address recipient
    ) public view {
        // 约束组合
        vm.assume(amount > 100);
        vm.assume(amount < type(uint256).max / 2);
        vm.assume(recipient != address(0));
        vm.assume(recipient != msg.sender);
        
        // 测试逻辑
    }
}
```

### 模糊测试配置

```toml
# foundry.toml
[profile.default]
fuzz = { runs = 256, max_test_rejects = 65536 }

[profile.ci]
fuzz = { runs = 10000 }
```

```solidity
// 在测试文件中配置
contract ConfiguredFuzzTest is Test {
    // 为单个测试配置
    /// forge-config: default.fuzz.runs = 1000
    function testFuzz_CustomConfig(uint256 x) public pure {
        // ...
    }
    
    // 配置多个参数
    /// forge-config: default.fuzz.runs = 500
    /// forge-config: ci.fuzz.runs = 5000
    function testFuzz_MultiConfig(uint256 x) public pure {
        // ...
    }
}
```

### 模糊测试最佳实践

```solidity
contract FuzzBestPracticesTest is Test {
    SafeMath math;
    
    function setUp() public {
        math = new SafeMath();
    }
    
    // ✅ 好：测试数学不变量
    function testFuzz_AddCommutative(uint256 a, uint256 b) public pure {
        assertEq(a + b, b + a, "Addition should be commutative");
    }
    
    // ✅ 好：测试边界条件
    function testFuzz_MulNoOverflow(uint256 a, uint256 b) public {
        vm.assume(b == 0 || a <= type(uint256).max / b);
        
        uint256 result = math.mul(a, b);
        assertEq(result, a * b);
    }
    
    // ✅ 好：测试状态变化
    function testFuzz_BalanceConsistency(
        address from,
        address to,
        uint256 amount
    ) public {
        vm.assume(from != to);
        vm.assume(amount <= token.balanceOf(from));
        
        uint256 totalBefore = token.balanceOf(from) + token.balanceOf(to);
        
        vm.prank(from);
        token.transfer(to, amount);
        
        uint256 totalAfter = token.balanceOf(from) + token.balanceOf(to);
        assertEq(totalBefore, totalAfter, "Total should be conserved");
    }
}
```

## 部署合约

### 部署脚本

```solidity
// script/DeploySimpleToken.s.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";
import "../src/SimpleToken.sol";

contract DeploySimpleToken is Script {
    function run() external returns (SimpleToken) {
        // 从环境变量获取参数
        string memory name = vm.envString("TOKEN_NAME");
        string memory symbol = vm.envString("TOKEN_SYMBOL");
        uint256 initialSupply = vm.envUint("INITIAL_SUPPLY");
        
        // 开始广播交易
        vm.startBroadcast();
        
        SimpleToken token = new SimpleToken(name, symbol, initialSupply);
        
        vm.stopBroadcast();
        
        return token;
    }
}
```

### 运行部署

```bash
# 本地部署（使用 Anvil）
anvil &
forge script script/DeploySimpleToken.s.sol --rpc-url localhost --broadcast

# 部署到测试网
forge script script/DeploySimpleToken.s.sol \
    --rpc-url $SEPOLIA_RPC_URL \
    --private-key $PRIVATE_KEY \
    --broadcast \
    --verify \
    --etherscan-api-key $ETHERSCAN_API_KEY

# 模拟部署（不实际发送交易）
forge script script/DeploySimpleToken.s.sol \
    --rpc-url $SEPOLIA_RPC_URL \
    --private-key $PRIVATE_KEY

# 使用账户文件部署
forge script script/DeploySimpleToken.s.sol \
    --rpc-url $SEPOLIA_RPC_URL \
    --account my-account \
    --password my-password \
    --broadcast
```

### 复杂部署脚本

```solidity
// script/DeployDeFi.s.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import "forge-std/Script.sol";
import "../src/Token.sol";
import "../src/Exchange.sol";
import "../src/Staking.sol";

contract DeployDeFi is Script {
    function run() external {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        address deployer = vm.addr(deployerPrivateKey);
        
        vm.startBroadcast(deployerPrivateKey);
        
        // 1. 部署代币
        Token token = new Token("DeFi Token", "DFT", 1_000_000 * 10 ** 18);
        console.log("Token deployed at:", address(token));
        
        // 2. 部署交易所
        Exchange exchange = new Exchange(address(token));
        console.log("Exchange deployed at:", address(exchange));
        
        // 3. 部署质押合约
        Staking staking = new Staking(address(token));
        console.log("Staking deployed at:", address(staking));
        
        // 4. 转移代币到合约
        token.transfer(address(exchange), 100_000 * 10 ** 18);
        token.transfer(address(staking), 100_000 * 10 ** 18);
        
        // 5. 设置权限
        token.setMinter(address(staking), true);
        
        vm.stopBroadcast();
        
        // 输出部署摘要
        console.log("\n=== Deployment Summary ===");
        console.log("Deployer:", deployer);
        console.log("Token:", address(token));
        console.log("Exchange:", address(exchange));
        console.log("Staking:", address(staking));
    }
}
```

### 使用广播记录

Foundry 会在 `broadcast/` 目录保存部署记录：

```
broadcast/
└── DeploySimpleToken.s.sol/
    └── 11155111/  # 链 ID
        ├── run-1700000000.json  # 部署记录
        └── run-latest.json      # 最新记录
```

## Cast 命令行工具

Cast 用于与区块链交互：

```bash
# 查询余额
cast balance 0x... --rpc-url $RPC_URL
cast balance 0x... --ether  # 以 ETH 显示

# 读取合约
cast call 0x... "name()" --rpc-url $RPC_URL
cast call 0x... "balanceOf(address)" 0x... --rpc-url $RPC_URL

# 发送交易
cast send 0x... "transfer(address,uint256)" 0x... 100 \
    --rpc-url $RPC_URL \
    --private-key $PRIVATE_KEY

# 估算 gas
cast estimate 0x... "transfer(address,uint256)" 0x... 100 \
    --rpc-url $RPC_URL

# 编码 calldata
cast calldata "transfer(address,uint256)" 0x... 100

# 解码事件
cast decode-event --sig "Transfer(address,address,uint256)" \
    0x...

# ABI 编码/解码
cast abi-encode "f(uint256)" 100
cast --calldata-decode "f(uint256)" 0x...

# 转换格式
cast --to-hex 100
cast --to-dec 0x64
cast --to-wei 1 ether
cast --from-wei 1000000000000000000

# 计算 Keccak256
cast keccak "hello"
cast sig "transfer(address,uint256)"  # 函数选择器

# 获取区块信息
cast block latest --rpc-url $RPC_URL
cast block-number --rpc-url $RPC_URL

# 获取交易
cast tx <TX_HASH> --rpc-url $RPC_URL
cast receipt <TX_HASH> --rpc-url $RPC_URL

# 模拟交易
cast run <TX_HASH> --rpc-url $RPC_URL

# 获取 gas 价格
cast gas-price --rpc-url $RPC_URL

# 获取 nonce
cast nonce 0x... --rpc-url $RPC_URL
```

## Gas 优化

### Gas 快照

```bash
# 生成 gas 快照
forge snapshot

# 比较快照
forge snapshot --diff .gas-snapshot

# 导出为表格
forge snapshot --json > gas.json
```

`.gas-snapshot` 文件：

```
testFuzz_Transfer(uint256) 51023
test_Deployment() 20834
test_Transfer() 51023
```

### Gas 优化技巧

```solidity
contract GasOptimization is Test {
    // 使用 constant 和 immutable
    uint256 public constant DECIMALS = 18;
    address public immutable owner;
    
    // 缓存数组长度
    function sum(uint256[] memory arr) public pure returns (uint256) {
        uint256 total = 0;
        uint256 length = arr.length;  // 缓存长度
        
        for (uint256 i = 0; i < length;) {
            total += arr[i];
            unchecked { ++i; }  // unchecked 递增
        }
        
        return total;
    }
    
    // 使用 uint256 而不是更小的整数
    // 使用 unchecked 进行不会溢出的运算
    function increment(uint256 x) public pure returns (uint256) {
        unchecked {
            return x + 1;
        }
    }
    
    // 使用 calldata 而不是 memory（外部函数）
    function processData(bytes calldata data) external pure returns (bytes32) {
        return keccak256(data);
    }
}
```

## 最佳实践

1. **测试命名**：`test_FunctionName` 和 `testFuzz_FunctionName`
2. **使用 setUp**：每个测试前重置状态
3. **使用断言消息**：`assertEq(a, b, "Message")`
4. **测试边界情况**：0、最大值、空数组等
5. **使用模糊测试**：自动发现边界问题
6. **分离测试文件**：每个合约一个测试文件
7. **使用 console.log**：调试时打印信息
8. **配置 CI**：增加模糊测试运行次数

## 下一步

Foundry 开发流程已经掌握，继续学习 [测试最佳实践](./05-testing.md)，深入了解单元测试和模糊测试的技巧。
