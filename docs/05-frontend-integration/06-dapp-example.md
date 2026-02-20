# 5.6 完整 DApp 示例

## 项目概述

我们将创建一个简单的**代币水龙头 DApp**，包含以下功能：

- 查看代币余额
- 领取代币（每24小时一次）
- 查看领取记录

```
┌─────────────────────────────────────────────────────────────┐
│                    DApp 架构                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  前端 (React + TypeScript + wagmi)                          │
│  ├── 连接钱包                                               │
│  ├── 查询余额                                               │
│  ├── 领取代币                                               │
│  └── 显示交易状态                                           │
│                                                             │
│  智能合约 (Solidity)                                        │
│  ├── 代币合约 (ERC-20)                                      │
│  └── 水龙头合约                                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 智能合约

### 代币合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MyToken is ERC20 {
    constructor() ERC20("My Token", "MTK") {
        _mint(msg.sender, 1000000 * 10**decimals());
    }
}
```

### 水龙头合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract Faucet is Ownable {
    IERC20 public token;
    uint256 public claimAmount = 100 * 10**18; // 100 tokens
    uint256 public claimCooldown = 24 hours;
    
    mapping(address => uint256) public lastClaimTime;
    
    event Claimed(address indexed user, uint256 amount, uint256 timestamp);
    
    constructor(address _token) Ownable(msg.sender) {
        token = IERC20(_token);
    }
    
    function claim() external {
        require(
            block.timestamp >= lastClaimTime[msg.sender] + claimCooldown,
            "Please wait for cooldown"
        );
        
        require(
            token.balanceOf(address(this)) >= claimAmount,
            "Faucet is empty"
        );
        
        lastClaimTime[msg.sender] = block.timestamp;
        
        require(
            token.transfer(msg.sender, claimAmount),
            "Transfer failed"
        );
        
        emit Claimed(msg.sender, claimAmount, block.timestamp);
    }
    
    function setClaimAmount(uint256 _amount) external onlyOwner {
        claimAmount = _amount;
    }
    
    function setClaimCooldown(uint256 _cooldown) external onlyOwner {
        claimCooldown = _cooldown;
    }
    
    function withdrawTokens(uint256 amount) external onlyOwner {
        require(token.transfer(msg.sender, amount), "Transfer failed");
    }
    
    function getNextClaimTime(address user) external view returns (uint256) {
        uint256 lastClaim = lastClaimTime[user];
        if (lastClaim == 0) {
            return 0;
        }
        return lastClaim + claimCooldown;
    }
    
    function canClaim(address user) external view returns (bool) {
        return block.timestamp >= lastClaimTime[user] + claimCooldown;
    }
}
```

## 前端项目结构

```
faucet-dapp/
├── src/
│   ├── components/
│   │   ├── ConnectWallet.tsx
│   │   ├── TokenBalance.tsx
│   │   ├── ClaimButton.tsx
│   │   └── TransactionStatus.tsx
│   ├── hooks/
│   │   ├── useTokenBalance.ts
│   │   └── useFaucet.ts
│   ├── contracts/
│   │   ├── addresses.ts
│   │   ├── MyToken.json
│   │   └── Faucet.json
│   ├── App.tsx
│   ├── main.tsx
│   └── wagmi.ts
├── package.json
└── vite.config.ts
```

## 配置文件

### wagmi.ts

```typescript
import { http, createConfig } from 'wagmi';
import { mainnet, sepolia } from 'wagmi/chains';
import { injected, metaMask, walletConnect } from 'wagmi/connectors';

const projectId = 'YOUR_WALLET_CONNECT_PROJECT_ID';

export const config = createConfig({
  chains: [mainnet, sepolia],
  connectors: [
    injected(),
    metaMask(),
    walletConnect({ projectId }),
  ],
  transports: {
    [mainnet.id]: http(),
    [sepolia.id]: http(),
  },
});

declare module 'wagmi' {
  interface Register {
    config: typeof config;
  }
}
```

### 合约地址配置

```typescript
// src/contracts/addresses.ts
import { sepolia } from 'wagmi/chains';

export const addresses = {
  [sepolia.id]: {
    token: '0xTokenAddress...' as `0x${string}`,
    faucet: '0xFaucetAddress...' as `0x${string}`,
  },
} as const;

export function getAddresses(chainId: number) {
  return addresses[chainId as keyof typeof addresses];
}
```

## React 组件

### ConnectWallet.tsx

```tsx
import { useAccount, useConnect, useDisconnect, useBalance } from 'wagmi';

export function ConnectWallet() {
  const { address, isConnected, chain } = useAccount();
  const { connectors, connect, isPending } = useConnect();
  const { disconnect } = useDisconnect();
  const { data: balance } = useBalance({ address });
  
  if (isConnected) {
    return (
      <div className="wallet-info">
        <div className="address">
          {address?.slice(0, 6)}...{address?.slice(-4)}
        </div>
        <div className="balance">
          {balance?.formatted} {balance?.symbol}
        </div>
        <div className="chain">{chain?.name}</div>
        <button onClick={() => disconnect()}>断开连接</button>
      </div>
    );
  }
  
  return (
    <div className="connect-buttons">
      {connectors.map((connector) => (
        <button
          key={connector.uid}
          onClick={() => connect({ connector })}
          disabled={isPending}
        >
          {isPending ? '连接中...' : `连接 ${connector.name}`}
        </button>
      ))}
    </div>
  );
}
```

### TokenBalance.tsx

```tsx
import { useAccount, useReadContract } from 'wagmi';
import { formatUnits } from 'viem';
import { addresses } from '../contracts/addresses';
import { MyTokenAbi } from '../contracts/MyToken';

export function TokenBalance() {
  const { address, chain } = useAccount();
  
  const chainId = chain?.id || 11155111; // 默认 Sepolia
  const contractAddresses = addresses[chainId as keyof typeof addresses];
  
  const { data: balance, isLoading } = useReadContract({
    address: contractAddresses?.token,
    abi: MyTokenAbi,
    functionName: 'balanceOf',
    args: [address],
  });
  
  const { data: symbol } = useReadContract({
    address: contractAddresses?.token,
    abi: MyTokenAbi,
    functionName: 'symbol',
  });
  
  const { data: decimals } = useReadContract({
    address: contractAddresses?.token,
    abi: MyTokenAbi,
    functionName: 'decimals',
  });
  
  if (isLoading) {
    return <div>加载中...</div>;
  }
  
  const formattedBalance = balance && decimals
    ? formatUnits(balance as bigint, decimals as number)
    : '0';
  
  return (
    <div className="token-balance">
      <h3>代币余额</h3>
      <p className="balance">
        {formattedBalance} {symbol}
      </p>
    </div>
  );
}
```

### ClaimButton.tsx

```tsx
import { useState, useEffect } from 'react';
import { useAccount, useReadContract, useWriteContract, useWaitForTransactionReceipt } from 'wagmi';
import { addresses } from '../contracts/addresses';
import { FaucetAbi } from '../contracts/Faucet';

export function ClaimButton() {
  const { address, chain } = useAccount();
  const [cooldownRemaining, setCooldownRemaining] = useState(0);
  
  const chainId = chain?.id || 11155111;
  const contractAddresses = addresses[chainId as keyof typeof addresses];
  
  // 查询是否可以领取
  const { data: canClaim, refetch: refetchCanClaim } = useReadContract({
    address: contractAddresses?.faucet,
    abi: FaucetAbi,
    functionName: 'canClaim',
    args: [address],
  });
  
  // 查询下次可领取时间
  const { data: nextClaimTime } = useReadContract({
    address: contractAddresses?.faucet,
    abi: FaucetAbi,
    functionName: 'getNextClaimTime',
    args: [address],
  });
  
  // 写入合约
  const { data: hash, writeContract, isPending } = useWriteContract();
  
  // 等待交易确认
  const { isLoading: isConfirming, isSuccess } = useWaitForTransactionReceipt({
    hash,
  });
  
  // 更新倒计时
  useEffect(() => {
    if (nextClaimTime) {
      const updateCooldown = () => {
        const now = Math.floor(Date.now() / 1000);
        const remaining = Number(nextClaimTime) - now;
        setCooldownRemaining(remaining > 0 ? remaining : 0);
      };
      
      updateCooldown();
      const interval = setInterval(updateCooldown, 1000);
      
      return () => clearInterval(interval);
    }
  }, [nextClaimTime]);
  
  // 交易成功后刷新
  useEffect(() => {
    if (isSuccess) {
      refetchCanClaim();
    }
  }, [isSuccess, refetchCanClaim]);
  
  // 领取代币
  const handleClaim = () => {
    writeContract({
      address: contractAddresses?.faucet,
      abi: FaucetAbi,
      functionName: 'claim',
    });
  };
  
  // 格式化剩余时间
  const formatTime = (seconds: number) => {
    const hours = Math.floor(seconds / 3600);
    const minutes = Math.floor((seconds % 3600) / 60);
    const secs = seconds % 60;
    return `${hours}时${minutes}分${secs}秒`;
  };
  
  // 状态判断
  const isButtonDisabled = !canClaim || isPending || isConfirming;
  
  let buttonText = '领取代币';
  if (isPending) buttonText = '确认中...';
  else if (isConfirming) buttonText = '交易确认中...';
  else if (!canClaim && cooldownRemaining > 0) {
    buttonText = `等待 ${formatTime(cooldownRemaining)}`;
  }
  
  return (
    <div className="claim-section">
      <button
        onClick={handleClaim}
        disabled={isButtonDisabled}
        className={isSuccess ? 'success' : ''}
      >
        {buttonText}
      </button>
      
      {hash && (
        <div className="tx-hash">
          交易哈希: {hash.slice(0, 10)}...{hash.slice(-8)}
        </div>
      )}
      
      {isSuccess && (
        <div className="success-message">
          领取成功！
        </div>
      )}
    </div>
  );
}
```

### App.tsx

```tsx
import { WagmiProvider } from 'wagmi';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { config } from './wagmi';
import { ConnectWallet } from './components/ConnectWallet';
import { TokenBalance } from './components/TokenBalance';
import { ClaimButton } from './components/ClaimButton';
import './App.css';

const queryClient = new QueryClient();

function App() {
  return (
    <WagmiProvider config={config}>
      <QueryClientProvider client={queryClient}>
        <div className="app">
          <header>
            <h1>代币水龙头</h1>
            <ConnectWallet />
          </header>
          
          <main>
            <TokenBalance />
            <ClaimButton />
          </main>
          
          <footer>
            <p>每24小时可领取一次，每次100 MTK</p>
          </footer>
        </div>
      </QueryClientProvider>
    </WagmiProvider>
  );
}

export default App;
```

## 样式文件

```css
/* App.css */
.app {
  max-width: 600px;
  margin: 0 auto;
  padding: 20px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}

header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 40px;
  padding-bottom: 20px;
  border-bottom: 1px solid #eee;
}

.wallet-info {
  display: flex;
  align-items: center;
  gap: 10px;
}

.address {
  font-family: monospace;
  background: #f0f0f0;
  padding: 5px 10px;
  border-radius: 4px;
}

.connect-buttons {
  display: flex;
  gap: 10px;
}

button {
  background: #4a90d9;
  color: white;
  border: none;
  padding: 10px 20px;
  border-radius: 8px;
  cursor: pointer;
  font-size: 14px;
}

button:hover:not(:disabled) {
  background: #3a7bc8;
}

button:disabled {
  background: #ccc;
  cursor: not-allowed;
}

.token-balance {
  background: #f9f9f9;
  padding: 20px;
  border-radius: 12px;
  margin-bottom: 20px;
}

.token-balance h3 {
  margin: 0 0 10px 0;
  color: #666;
}

.token-balance .balance {
  font-size: 32px;
  font-weight: bold;
  margin: 0;
}

.claim-section {
  text-align: center;
}

.claim-section button {
  width: 100%;
  padding: 15px;
  font-size: 18px;
}

.claim-section button.success {
  background: #4caf50;
}

.tx-hash {
  margin-top: 10px;
  font-family: monospace;
  font-size: 12px;
  color: #666;
}

.success-message {
  margin-top: 10px;
  color: #4caf50;
  font-weight: bold;
}

footer {
  margin-top: 40px;
  text-align: center;
  color: #999;
}
```

## 运行项目

### 安装依赖

```bash
npm create vite@latest faucet-dapp -- --template react-ts
cd faucet-dapp

npm install wagmi viem@2.x @tanstack/react-query
```

### 开发命令

```bash
# 启动开发服务器
npm run dev

# 构建生产版本
npm run build

# 预览生产版本
npm run preview
```

## 部署

### 前端部署 (Vercel)

```bash
# 安装 Vercel CLI
npm i -g vercel

# 部署
vercel
```

### 环境变量

```bash
# .env.production
VITE_WALLET_CONNECT_PROJECT_ID=your_project_id
```

## 扩展建议

1. **添加网络切换** - 提示用户切换到正确的网络
2. **添加交易历史** - 显示用户的领取记录
3. **添加代币转账** - 允许用户转账代币
4. **添加移动端适配** - 优化移动端体验
5. **添加错误边界** - 更好的错误处理

## 恭喜！

你已经完成了一个完整的 Web3 DApp！接下来可以：

1. 学习更多 Solidity 高级特性
2. 探索 DeFi 协议开发
3. 尝试 NFT 项目
4. 学习 Layer 2 开发

[返回目录](README.md)
