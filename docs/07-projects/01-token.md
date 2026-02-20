# 7.1 项目一：ERC-20 代币

## 项目概述

创建一个完整的 ERC-20 代币项目，包含合约、测试和前端。

```
┌─────────────────────────────────────────────────────────────┐
│                    项目结构                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  my-token/                                                  │
│  ├── contracts/                                             │
│  │   └── MyToken.sol                                       │
│  ├── test/                                                  │
│  │   └── MyToken.t.sol                                     │
│  ├── script/                                                │
│  │   └── Deploy.s.sol                                      │
│  └── frontend/                                              │
│      └── (React DApp)                                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 智能合约

### 基础 ERC-20 合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Permit.sol";

contract MyToken is ERC20, Ownable, ERC20Burnable, ERC20Permit {
    uint8 private constant _decimals = 18;
    uint256 public constant MAX_SUPPLY = 1_000_000_000 * 10**_decimals; // 10亿
    
    event Minted(address indexed to, uint256 amount);
    event Burned(address indexed from, uint256 amount);
    
    constructor(
        string memory name,
        string memory symbol,
        uint256 initialSupply
    ) ERC20(name, symbol) Ownable(msg.sender) ERC20Permit(name) {
        require(initialSupply <= MAX_SUPPLY, "Exceeds max supply");
        _mint(msg.sender, initialSupply);
    }
    
    function decimals() public pure override returns (uint8) {
        return _decimals;
    }
    
    function mint(address to, uint256 amount) public onlyOwner {
        require(totalSupply() + amount <= MAX_SUPPLY, "Exceeds max supply");
        _mint(to, amount);
        emit Minted(to, amount);
    }
    
    function burn(uint256 amount) public override {
        super.burn(amount);
        emit Burned(msg.sender, amount);
    }
    
    function burnFrom(address account, uint256 amount) public override {
        super.burnFrom(account, amount);
        emit Burned(account, amount);
    }
}
```

### 功能增强版

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Pausable.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Permit.sol";

contract AdvancedToken is ERC20, Ownable, ERC20Burnable, ERC20Pausable, ERC20Permit {
    uint8 private constant _decimals = 18;
    uint256 public constant MAX_SUPPLY = 1_000_000_000 * 10**_decimals;
    
    mapping(address => bool) public blacklisted;
    mapping(address => bool) public operators;
    
    event Blacklisted(address indexed account, bool status);
    event OperatorSet(address indexed operator, bool status);
    
    modifier notBlacklisted(address account) {
        require(!blacklisted[account], "Account is blacklisted");
        _;
    }
    
    modifier onlyOperator() {
        require(operators[msg.sender] || msg.sender == owner(), "Not operator");
        _;
    }
    
    constructor(
        string memory name,
        string memory symbol,
        uint256 initialSupply
    ) ERC20(name, symbol) Ownable(msg.sender) ERC20Permit(name) {
        _mint(msg.sender, initialSupply);
    }
    
    function decimals() public pure override returns (uint8) {
        return _decimals;
    }
    
    function mint(address to, uint256 amount) public onlyOwner {
        require(totalSupply() + amount <= MAX_SUPPLY, "Exceeds max supply");
        _mint(to, amount);
    }
    
    function pause() public onlyOwner {
        _pause();
    }
    
    function unpause() public onlyOwner {
        _unpause();
    }
    
    function setBlacklist(address account, bool status) public onlyOperator {
        blacklisted[account] = status;
        emit Blacklisted(account, status);
    }
    
    function setOperator(address operator, bool status) public onlyOwner {
        operators[operator] = status;
        emit OperatorSet(operator, status);
    }
    
    function _update(
        address from,
        address to,
        uint256 amount
    ) internal override(ERC20, ERC20Pausable) {
        require(!blacklisted[from], "Sender blacklisted");
        require(!blacklisted[to], "Recipient blacklisted");
        super._update(from, to, amount);
    }
}
```

## 测试文件

```solidity
// test/MyToken.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/MyToken.sol";

contract MyTokenTest is Test {
    MyToken public token;
    address public owner;
    address public user1;
    address public user2;
    
    string constant NAME = "My Token";
    string constant SYMBOL = "MTK";
    uint256 constant INITIAL_SUPPLY = 1_000_000 * 10**18;
    
    function setUp() public {
        owner = address(this);
        user1 = address(0x1);
        user2 = address(0x2);
        
        token = new MyToken(NAME, SYMBOL, INITIAL_SUPPLY);
    }
    
    function test_Deployment() public view {
        assertEq(token.name(), NAME);
        assertEq(token.symbol(), SYMBOL);
        assertEq(token.decimals(), 18);
        assertEq(token.totalSupply(), INITIAL_SUPPLY);
        assertEq(token.balanceOf(owner), INITIAL_SUPPLY);
        assertEq(token.owner(), owner);
    }
    
    function test_Mint() public {
        uint256 mintAmount = 1000 * 10**18;
        token.mint(user1, mintAmount);
        
        assertEq(token.balanceOf(user1), mintAmount);
        assertEq(token.totalSupply(), INITIAL_SUPPLY + mintAmount);
    }
    
    function test_MintExceedsMaxSupply() public {
        uint256 maxMint = token.MAX_SUPPLY() - INITIAL_SUPPLY;
        token.mint(user1, maxMint);
        
        vm.expectRevert("Exceeds max supply");
        token.mint(user1, 1);
    }
    
    function test_Transfer() public {
        uint256 transferAmount = 1000 * 10**18;
        
        token.transfer(user1, transferAmount);
        
        assertEq(token.balanceOf(user1), transferAmount);
        assertEq(token.balanceOf(owner), INITIAL_SUPPLY - transferAmount);
    }
    
    function test_Approve() public {
        uint256 approveAmount = 1000 * 10**18;
        
        token.approve(user1, approveAmount);
        
        assertEq(token.allowance(owner, user1), approveAmount);
    }
    
    function test_TransferFrom() public {
        uint256 amount = 1000 * 10**18;
        
        token.approve(user1, amount);
        
        vm.prank(user1);
        token.transferFrom(owner, user2, amount);
        
        assertEq(token.balanceOf(user2), amount);
        assertEq(token.allowance(owner, user1), 0);
    }
    
    function test_Burn() public {
        uint256 burnAmount = 1000 * 10**18;
        
        token.burn(burnAmount);
        
        assertEq(token.totalSupply(), INITIAL_SUPPLY - burnAmount);
    }
    
    function test_BurnFrom() public {
        uint256 amount = 1000 * 10**18;
        
        token.transfer(user1, amount);
        
        vm.prank(user1);
        token.approve(owner, amount);
        
        token.burnFrom(user1, amount);
        
        assertEq(token.balanceOf(user1), 0);
    }
    
    function test_MintOnlyOwner() public {
        vm.prank(user1);
        vm.expectRevert();
        token.mint(user1, 1000);
    }
    
    function testFuzz_Transfer(uint256 amount) public {
        vm.assume(amount <= INITIAL_SUPPLY);
        
        token.transfer(user1, amount);
        
        assertEq(token.balanceOf(user1), amount);
    }
}
```

## 部署脚本

```solidity
// script/Deploy.s.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Script.sol";
import "../src/MyToken.sol";

contract DeployScript is Script {
    function run() external returns (MyToken) {
        uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
        
        vm.startBroadcast(deployerPrivateKey);
        
        MyToken token = new MyToken(
            "My Token",
            "MTK",
            1_000_000 * 10**18
        );
        
        vm.stopBroadcast();
        
        return token;
    }
}
```

## 验证合约

```bash
# 使用 Foundry 验证
forge verify-contract \
  --chain-id 11155111 \
  --num-of-optimizations 200 \
  --constructor-args $(cast abi-encode "constructor(string,string,uint256)" "My Token" "MTK" 1000000000000000000000000) \
  --etherscan-api-key $ETHERSCAN_API_KEY \
  <CONTRACT_ADDRESS> \
  src/MyToken.sol:MyToken
```

## 前端集成

### 合约 ABI

```typescript
// src/contracts/MyToken.ts
export const MyTokenABI = [
  {
    inputs: [
      { name: 'name', type: 'string' },
      { name: 'symbol', type: 'string' },
      { name: 'initialSupply', type: 'uint256' },
    ],
    stateMutability: 'nonpayable',
    type: 'constructor',
  },
  {
    inputs: [{ name: 'owner', type: 'address' }],
    name: 'balanceOf',
    outputs: [{ name: '', type: 'uint256' }],
    stateMutability: 'view',
    type: 'function',
  },
  {
    inputs: [
      { name: 'to', type: 'address' },
      { name: 'amount', type: 'uint256' },
    ],
    name: 'transfer',
    outputs: [{ name: '', type: 'bool' }],
    stateMutability: 'nonpayable',
    type: 'function',
  },
  {
    inputs: [
      { name: 'spender', type: 'address' },
      { name: 'amount', type: 'uint256' },
    ],
    name: 'approve',
    outputs: [{ name: '', type: 'bool' }],
    stateMutability: 'nonpayable',
    type: 'function',
  },
  {
    inputs: [
      { name: 'spender', type: 'address' },
      { name: 'amount', type: 'uint256' },
    ],
    name: 'mint',
    outputs: [],
    stateMutability: 'nonpayable',
    type: 'function',
  },
] as const;
```

### React 组件

```tsx
import { useAccount, useReadContract, useWriteContract } from 'wagmi';
import { parseUnits, formatUnits } from 'viem';
import { MyTokenABI } from '../contracts/MyToken';

const TOKEN_ADDRESS = '0x...' as `0x${string}`;

export function TokenDashboard() {
  const { address } = useAccount();
  const { writeContract } = useWriteContract();
  
  const { data: balance } = useReadContract({
    address: TOKEN_ADDRESS,
    abi: MyTokenABI,
    functionName: 'balanceOf',
    args: [address],
  });
  
  const { data: totalSupply } = useReadContract({
    address: TOKEN_ADDRESS,
    abi: MyTokenABI,
    functionName: 'totalSupply',
  });
  
  const { data: name } = useReadContract({
    address: TOKEN_ADDRESS,
    abi: MyTokenABI,
    functionName: 'name',
  });
  
  const { data: symbol } = useReadContract({
    address: TOKEN_ADDRESS,
    abi: MyTokenABI,
    functionName: 'symbol',
  });
  
  const handleTransfer = (to: string, amount: string) => {
    writeContract({
      address: TOKEN_ADDRESS,
      abi: MyTokenABI,
      functionName: 'transfer',
      args: [to, parseUnits(amount, 18)],
    });
  };
  
  return (
    <div>
      <h1>{name} ({symbol})</h1>
      <p>总供应: {totalSupply ? formatUnits(totalSupply as bigint, 18) : 0}</p>
      <p>我的余额: {balance ? formatUnits(balance as bigint, 18) : 0}</p>
    </div>
  );
}
```

## 下一步

[下一项目：NFT 市场 →](02-nft.md)
