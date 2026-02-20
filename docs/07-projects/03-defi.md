# 7.3 项目三：简易 DEX

## 项目概述

创建一个类似 Uniswap V2 的去中心化交易所。

```
┌─────────────────────────────────────────────────────────────┐
│                    DEX 架构                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  工厂合约 (Factory)                                         │
│  ├── 创建交易对                                             │
│  └── 管理交易对地址                                         │
│                                                             │
│  交易对合约 (Pair)                                          │
│  ├── 流动性池 (TokenA + TokenB)                            │
│  ├── AMM 价格计算                                           │
│  └── LP Token 管理                                          │
│                                                             │
│  路由合约 (Router)                                          │
│  ├── 代币兑换                                               │
│  ├── 添加/移除流动性                                        │
│  └── 多跳交易                                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 智能合约

### 交易对合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

contract SimplePair is ERC20 {
    using SafeERC20 for IERC20;
    
    IERC20 public token0;
    IERC20 public token1;
    
    uint256 public reserve0;
    uint256 public reserve1;
    
    uint256 public constant MINIMUM_LIQUIDITY = 1000;
    
    event Mint(address indexed sender, uint256 amount0, uint256 amount1);
    event Burn(address indexed sender, uint256 amount0, uint256 amount1, address to);
    event Swap(
        address indexed sender,
        uint256 amount0In,
        uint256 amount1In,
        uint256 amount0Out,
        uint256 amount1Out,
        address to
    );
    event Sync(uint256 reserve0, uint256 reserve1);
    
    constructor(address _token0, address _token1) ERC20("LP Token", "LP") {
        token0 = IERC20(_token0);
        token1 = IERC20(_token1);
    }
    
    function mint(address to) external returns (uint256 liquidity) {
        uint256 balance0 = token0.balanceOf(address(this));
        uint256 balance1 = token1.balanceOf(address(this));
        
        uint256 amount0 = balance0 - reserve0;
        uint256 amount1 = balance1 - reserve1;
        
        uint256 totalSupply_ = totalSupply();
        
        if (totalSupply_ == 0) {
            liquidity = sqrt(amount0 * amount1) - MINIMUM_LIQUIDITY;
            _mint(address(1), MINIMUM_LIQUIDITY); // 锁定最小流动性
        } else {
            liquidity = min(
                (amount0 * totalSupply_) / reserve0,
                (amount1 * totalSupply_) / reserve1
            );
        }
        
        require(liquidity > 0, "Insufficient liquidity minted");
        _mint(to, liquidity);
        
        _update(balance0, balance1);
        
        emit Mint(msg.sender, amount0, amount1);
    }
    
    function burn(address to) external returns (uint256 amount0, uint256 amount1) {
        uint256 liquidity = balanceOf(address(this));
        
        uint256 totalSupply_ = totalSupply();
        
        amount0 = (liquidity * reserve0) / totalSupply_;
        amount1 = (liquidity * reserve1) / totalSupply_;
        
        require(amount0 > 0 && amount1 > 0, "Insufficient liquidity burned");
        
        _burn(address(this), liquidity);
        
        token0.safeTransfer(to, amount0);
        token1.safeTransfer(to, amount1);
        
        uint256 balance0 = token0.balanceOf(address(this));
        uint256 balance1 = token1.balanceOf(address(this));
        
        _update(balance0, balance1);
        
        emit Burn(msg.sender, amount0, amount1, to);
    }
    
    function swap(
        uint256 amount0Out,
        uint256 amount1Out,
        address to
    ) external {
        require(amount0Out > 0 || amount1Out > 0, "Insufficient output amount");
        require(amount0Out < reserve0 && amount1Out < reserve1, "Insufficient liquidity");
        
        if (amount0Out > 0) token0.safeTransfer(to, amount0Out);
        if (amount1Out > 0) token1.safeTransfer(to, amount1Out);
        
        uint256 balance0 = token0.balanceOf(address(this));
        uint256 balance1 = token1.balanceOf(address(this));
        
        uint256 amount0In = balance0 > reserve0 - amount0Out 
            ? balance0 - (reserve0 - amount0Out) 
            : 0;
        uint256 amount1In = balance1 > reserve1 - amount1Out 
            ? balance1 - (reserve1 - amount1Out) 
            : 0;
        
        require(amount0In > 0 || amount1In > 0, "Insufficient input amount");
        
        // 检查 K 值（恒定乘积）
        uint256 balance0Adjusted = balance0 * 1000 - amount0In * 3;
        uint256 balance1Adjusted = balance1 * 1000 - amount1In * 3;
        require(
            balance0Adjusted * balance1Adjusted >= reserve0 * reserve1 * 1000**2,
            "K"
        );
        
        _update(balance0, balance1);
        
        emit Swap(msg.sender, amount0In, amount1In, amount0Out, amount1Out, to);
    }
    
    function _update(uint256 balance0, uint256 balance1) private {
        reserve0 = balance0;
        reserve1 = balance1;
        emit Sync(reserve0, reserve1);
    }
    
    function sqrt(uint256 y) internal pure returns (uint256 z) {
        if (y > 3) {
            z = y;
            uint256 x = y / 2 + 1;
            while (x < z) {
                z = x;
                x = (y / x + x) / 2;
            }
        } else if (y != 0) {
            z = 1;
        }
    }
    
    function min(uint256 a, uint256 b) internal pure returns (uint256) {
        return a < b ? a : b;
    }
    
    function getReserves() external view returns (uint256, uint256) {
        return (reserve0, reserve1);
    }
}
```

### 工厂合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "./SimplePair.sol";

contract SimpleFactory {
    mapping(address => mapping(address => address)) public getPair;
    address[] public allPairs;
    
    event PairCreated(
        address indexed token0,
        address indexed token1,
        address pair,
        uint256 pairCount
    );
    
    function createPair(
        address tokenA,
        address tokenB
    ) external returns (address pair) {
        require(tokenA != tokenB, "Identical addresses");
        
        (address token0, address token1) = tokenA < tokenB 
            ? (tokenA, tokenB) 
            : (tokenB, tokenA);
        
        require(token0 != address(0), "Zero address");
        require(getPair[token0][token1] == address(0), "Pair exists");
        
        bytes32 salt = keccak256(abi.encodePacked(token0, token1));
        
        pair = address(new SimplePair{salt: salt}(token0, token1));
        
        getPair[token0][token1] = pair;
        getPair[token1][token0] = pair;
        allPairs.push(pair);
        
        emit PairCreated(token0, token1, pair, allPairs.length);
    }
}
```

### 路由合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import "./SimpleFactory.sol";
import "./SimplePair.sol";

contract SimpleRouter {
    using SafeERC20 for IERC20;
    
    SimpleFactory public factory;
    
    constructor(address _factory) {
        factory = SimpleFactory(_factory);
    }
    
    function addLiquidity(
        address tokenA,
        address tokenB,
        uint256 amountADesired,
        uint256 amountBDesired,
        uint256 amountAMin,
        uint256 amountBMin,
        address to,
        uint256 deadline
    ) external returns (uint256 amountA, uint256 amountB, uint256 liquidity) {
        require(block.timestamp <= deadline, "Expired");
        
        (amountA, amountB) = _calculateLiquidityAmounts(
            tokenA, tokenB,
            amountADesired, amountBDesired,
            amountAMin, amountBMin
        );
        
        address pair = factory.getPair(tokenA, tokenB);
        
        IERC20(tokenA).safeTransferFrom(msg.sender, pair, amountA);
        IERC20(tokenB).safeTransferFrom(msg.sender, pair, amountB);
        
        liquidity = SimplePair(pair).mint(to);
    }
    
    function removeLiquidity(
        address tokenA,
        address tokenB,
        uint256 liquidity,
        uint256 amountAMin,
        uint256 amountBMin,
        address to,
        uint256 deadline
    ) external returns (uint256 amountA, uint256 amountB) {
        require(block.timestamp <= deadline, "Expired");
        
        address pair = factory.getPair(tokenA, tokenB);
        
        IERC20(pair).safeTransferFrom(msg.sender, pair, liquidity);
        
        (amountA, amountB) = SimplePair(pair).burn(to);
        
        require(amountA >= amountAMin, "Insufficient A amount");
        require(amountB >= amountBMin, "Insufficient B amount");
    }
    
    function swapExactTokensForTokens(
        uint256 amountIn,
        uint256 amountOutMin,
        address[] calldata path,
        address to,
        uint256 deadline
    ) external returns (uint256[] memory amounts) {
        require(block.timestamp <= deadline, "Expired");
        
        amounts = _getAmountsOut(amountIn, path);
        require(amounts[amounts.length - 1] >= amountOutMin, "Insufficient output");
        
        IERC20(path[0]).safeTransferFrom(
            msg.sender,
            factory.getPair(path[0], path[1]),
            amounts[0]
        );
        
        _swap(amounts, path, to);
    }
    
    function _swap(
        uint256[] memory amounts,
        address[] memory path,
        address to
    ) internal {
        for (uint256 i = 0; i < path.length - 1; i++) {
            (address input, address output) = (path[i], path[i + 1]);
            
            (address token0, ) = input < output 
                ? (input, output) 
                : (output, input);
            
            SimplePair pair = SimplePair(factory.getPair(input, output));
            
            uint256 amountOut = amounts[i + 1];
            
            (uint256 amount0Out, uint256 amount1Out) = input == token0
                ? (uint256(0), amountOut)
                : (amountOut, uint256(0));
            
            address _to = i < path.length - 2 
                ? factory.getPair(output, path[i + 2]) 
                : to;
            
            pair.swap(amount0Out, amount1Out, _to);
        }
    }
    
    function _calculateLiquidityAmounts(
        address tokenA,
        address tokenB,
        uint256 amountADesired,
        uint256 amountBDesired,
        uint256 amountAMin,
        uint256 amountBMin
    ) internal view returns (uint256 amountA, uint256 amountB) {
        address pair = factory.getPair(tokenA, tokenB);
        
        if (pair == address(0)) {
            (amountA, amountB) = (amountADesired, amountBDesired);
        } else {
            (uint256 reserveA, uint256 reserveB) = SimplePair(pair).getReserves();
            
            if (reserveA == 0 && reserveB == 0) {
                (amountA, amountB) = (amountADesired, amountBDesired);
            } else {
                uint256 amountBOptimal = (amountADesired * reserveB) / reserveA;
                
                if (amountBOptimal <= amountBDesired) {
                    require(amountBOptimal >= amountBMin, "Insufficient B");
                    (amountA, amountB) = (amountADesired, amountBOptimal);
                } else {
                    uint256 amountAOptimal = (amountBDesired * reserveA) / reserveB;
                    require(amountAOptimal <= amountADesired);
                    require(amountAOptimal >= amountAMin, "Insufficient A");
                    (amountA, amountB) = (amountAOptimal, amountBDesired);
                }
            }
        }
    }
    
    function _getAmountsOut(
        uint256 amountIn,
        address[] memory path
    ) internal view returns (uint256[] memory amounts) {
        require(path.length >= 2, "Invalid path");
        
        amounts = new uint256[](path.length);
        amounts[0] = amountIn;
        
        for (uint256 i = 0; i < path.length - 1; i++) {
            SimplePair pair = SimplePair(factory.getPair(path[i], path[i + 1]));
            (uint256 reserveIn, uint256 reserveOut) = pair.getReserves();
            
            amounts[i + 1] = (amounts[i] * 997 * reserveOut) / (reserveIn * 1000 + amounts[i] * 997);
        }
    }
    
    function getAmountsOut(
        uint256 amountIn,
        address[] calldata path
    ) external view returns (uint256[] memory) {
        return _getAmountsOut(amountIn, path);
    }
}
```

## 下一步

[下一项目：DAO 治理 →](04-dao.md)
