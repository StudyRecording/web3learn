# 5.7 wagmi v2 更新指南

## wagmi v2 重大变化

```
┌─────────────────────────────────────────────────────────────┐
│                   wagmi v2 主要变化                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. TanStack Query 完整支持                                 │
│     ├── 可配置缓存、staleTime 等                            │
│     └── 使用 queryKey 进行 invalidate                       │
│                                                             │
│  2. 配置方式变化                                            │
│     ├── 配置使用 createConfig                               │
│     └── 需要 QueryClientProvider                            │
│                                                             │
│  3. Hooks API 变化                                          │
│     ├── useContractRead → useReadContract                   │
│     ├── useContractWrite → useWriteContract                 │
│     └── useContractEvent → useWatchContractEvent            │
│                                                             │
│  4. query 选项位置变化                                      │
│     └── enabled、select 等移到 query 对象内                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 安装和配置

### 安装依赖

```bash
npm install wagmi viem@2.x @tanstack/react-query
```

### 基础配置

```typescript
// src/wagmi.ts
import { http, createConfig } from 'wagmi';
import { mainnet, sepolia, polygon, arbitrum } from 'wagmi/chains';
import { injected, metaMask, walletConnect, coinbaseWallet } from 'wagmi/connectors';

const projectId = 'YOUR_WALLET_CONNECT_PROJECT_ID';

export const config = createConfig({
  chains: [mainnet, sepolia, polygon, arbitrum],
  connectors: [
    injected(),
    metaMask(),
    coinbaseWallet({
      appName: 'My DApp',
    }),
    walletConnect({ projectId }),
  ],
  transports: {
    [mainnet.id]: http(),
    [sepolia.id]: http(),
    [polygon.id]: http(),
    [arbitrum.id]: http(),
  },
});

// 类型声明
declare module 'wagmi' {
  interface Register {
    config: typeof config;
  }
}
```

### 应用入口配置

```tsx
// src/main.tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { WagmiProvider } from 'wagmi';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { config } from './wagmi';
import App from './App';

import './index.css';

// 创建 QueryClient
const queryClient = new QueryClient();

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <WagmiProvider config={config}>
      <QueryClientProvider client={queryClient}>
        <App />
      </QueryClientProvider>
    </WagmiProvider>
  </React.StrictMode>
);
```

## v1 到 v2 迁移

### API 名称变化

```typescript
// v1 (旧)
import { useContractRead, useContractWrite, useContractEvent } from 'wagmi';

// v2 (新)
import { 
  useReadContract, 
  useWriteContract, 
  useWatchContractEvent 
} from 'wagmi';
```

### query 选项变化

```tsx
// v1 (旧)
const { data } = useContractRead({
  address: '0x...',
  abi: tokenAbi,
  functionName: 'balanceOf',
  args: [address],
  enabled: !!address,           // v1: 直接在外层
  select: (data) => formatUnits(data, 18),  // v1: 直接在外层
  suspense: true,               // v1: 直接在外层
});

// v2 (新)
const { data } = useReadContract({
  address: '0x...',
  abi: tokenAbi,
  functionName: 'balanceOf',
  args: [address],
  query: {                      // v2: 放在 query 对象内
    enabled: !!address,
    select: (data) => formatUnits(data as bigint, 18),
    gcTime: 1000 * 60 * 5,      // 缓存时间 5 分钟
    staleTime: 1000 * 30,       // 30 秒内数据视为新鲜
  },
});
```

## 核心 Hooks 使用

### 账户相关

```tsx
import { useAccount, useBalance, useDisconnect, useSwitchChain } from 'wagmi';

function AccountInfo() {
  const { address, isConnected, chain, connector } = useAccount();
  const { disconnect } = useDisconnect();
  const { switchChain } = useSwitchChain();
  
  const { data: balance, isLoading } = useBalance({
    address,
    query: {
      enabled: !!address,
      refetchInterval: 10000,  // 每 10 秒刷新
    },
  });
  
  if (!isConnected) {
    return <div>请连接钱包</div>;
  }
  
  return (
    <div>
      <p>地址: {address}</p>
      <p>链: {chain?.name}</p>
      <p>余额: {balance?.formatted} {balance?.symbol}</p>
      <p>连接器: {connector?.name}</p>
      
      <button onClick={() => switchChain({ chainId: 1 })}>
        切换到以太坊
      </button>
      <button onClick={() => disconnect()}>
        断开连接
      </button>
    </div>
  );
}
```

### 读取合约

```tsx
import { useReadContract } from 'wagmi';
import { formatUnits } from 'viem';

function TokenBalance({ 
  tokenAddress, 
  userAddress 
}: { 
  tokenAddress: `0x${string}`; 
  userAddress: `0x${string}` 
}) {
  const { 
    data: balance, 
    isLoading, 
    error,
    refetch,
    queryKey,
  } = useReadContract({
    address: tokenAddress,
    abi: tokenAbi,
    functionName: 'balanceOf',
    args: [userAddress],
    query: {
      enabled: !!userAddress,
      select: (data) => ({
        raw: data as bigint,
        formatted: formatUnits(data as bigint, 18),
      }),
      gcTime: 60_000,      // 缓存 1 分钟
      staleTime: 30_000,   // 30 秒内数据新鲜
    },
  });
  
  // 手动刷新
  const handleRefresh = () => {
    refetch();
  };
  
  if (isLoading) return <div>加载中...</div>;
  if (error) return <div>错误: {error.message}</div>;
  
  return (
    <div>
      <p>余额: {balance?.formatted}</p>
      <button onClick={handleRefresh}>刷新</button>
    </div>
  );
}
```

### 写入合约

```tsx
import { useWriteContract, useWaitForTransactionReceipt } from 'wagmi';
import { parseUnits } from 'viem';

function TransferToken({ tokenAddress }: { tokenAddress: `0x${string}` }) {
  const [to, setTo] = useState('');
  const [amount, setAmount] = useState('');
  
  const { 
    data: hash, 
    writeContract, 
    isPending,
    error,
  } = useWriteContract();
  
  // 等待交易确认
  const { isLoading: isConfirming, isSuccess } = useWaitForTransactionReceipt({
    hash,
  });
  
  const handleTransfer = () => {
    writeContract({
      address: tokenAddress,
      abi: tokenAbi,
      functionName: 'transfer',
      args: [to as `0x${string}`, parseUnits(amount, 18)],
    });
  };
  
  return (
    <div>
      <input 
        placeholder="接收地址" 
        value={to} 
        onChange={(e) => setTo(e.target.value)} 
      />
      <input 
        placeholder="金额" 
        value={amount} 
        onChange={(e) => setAmount(e.target.value)} 
      />
      
      <button 
        onClick={handleTransfer} 
        disabled={isPending || isConfirming}
      >
        {isPending ? '确认中...' : isConfirming ? '交易确认中...' : '转账'}
      </button>
      
      {hash && <p>交易哈希: {hash}</p>}
      {isSuccess && <p>交易成功!</p>}
      {error && <p>错误: {error.message}</p>}
    </div>
  );
}
```

### 监听事件

```tsx
import { useWatchContractEvent } from 'wagmi';

function TransferEvents({ tokenAddress }: { tokenAddress: `0x${string}` }) {
  const [events, setEvents] = useState<any[]>([]);
  
  useWatchContractEvent({
    address: tokenAddress,
    abi: tokenAbi,
    eventName: 'Transfer',
    onLogs(logs) {
      // 处理新事件
      const newEvents = logs.map((log) => ({
        from: log.args.from,
        to: log.args.to,
        value: log.args.value?.toString(),
        txHash: log.transactionHash,
      }));
      
      setEvents((prev) => [...prev, ...newEvents]);
    },
  });
  
  return (
    <div>
      <h3>转账事件</h3>
      <ul>
        {events.map((event, i) => (
          <li key={i}>
            {event.from?.slice(0, 6)}... → {event.to?.slice(0, 6)}... : {event.value}
          </li>
        ))}
      </ul>
    </div>
  );
}
```

## TanStack Query 高级用法

### 使用 queryKey 进行缓存控制

```tsx
import { useQueryClient } from '@tanstack/react-query';
import { useReadContract } from 'wagmi';

function TokenBalance({ tokenAddress, userAddress }) {
  const queryClient = useQueryClient();
  
  const { data, queryKey } = useReadContract({
    address: tokenAddress,
    abi: tokenAbi,
    functionName: 'balanceOf',
    args: [userAddress],
  });
  
  // 手动使缓存失效，触发重新获取
  const invalidate = () => {
    queryClient.invalidateQueries({ queryKey });
  };
  
  // 设置新数据（乐观更新）
  const setCachedData = (newBalance: bigint) => {
    queryClient.setQueryData(queryKey, newBalance);
  };
  
  return (
    <div>
      <p>余额: {data?.toString()}</p>
      <button onClick={invalidate}>刷新余额</button>
    </div>
  );
}
```

### 并行查询

```tsx
import { useReadContracts } from 'wagmi';

function TokenInfo({ tokenAddress }: { tokenAddress: `0x${string}` }) {
  const { data, isLoading } = useReadContracts({
    contracts: [
      {
        address: tokenAddress,
        abi: tokenAbi,
        functionName: 'name',
      },
      {
        address: tokenAddress,
        abi: tokenAbi,
        functionName: 'symbol',
      },
      {
        address: tokenAddress,
        abi: tokenAbi,
        functionName: 'decimals',
      },
      {
        address: tokenAddress,
        abi: tokenAbi,
        functionName: 'totalSupply',
      },
    ],
    query: {
      select: (results) => ({
        name: results[0].result,
        symbol: results[1].result,
        decimals: results[2].result,
        totalSupply: results[3].result,
      }),
    },
  });
  
  if (isLoading) return <div>加载中...</div>;
  
  return (
    <div>
      <p>名称: {data?.name}</p>
      <p>符号: {data?.symbol}</p>
      <p>精度: {data?.decimals}</p>
      <p>总供应: {data?.totalSupply?.toString()}</p>
    </div>
  );
}
```

### 模拟交易

```tsx
import { useSimulateContract, useWriteContract } from 'wagmi';

function SafeTransfer({ tokenAddress, to, amount }) {
  // 先模拟交易，检查是否会成功
  const { data: simulation, error: simulateError } = useSimulateContract({
    address: tokenAddress,
    abi: tokenAbi,
    functionName: 'transfer',
    args: [to, amount],
    query: {
      enabled: !!to && !!amount,
    },
  });
  
  const { writeContract, isPending } = useWriteContract();
  
  const handleTransfer = () => {
    if (simulation?.request) {
      writeContract(simulation.request);
    }
  };
  
  return (
    <div>
      {simulateError && <p className="error">交易将失败: {simulateError.message}</p>}
      <button 
        onClick={handleTransfer} 
        disabled={isPending || simulateError}
      >
        {isPending ? '处理中...' : '安全转账'}
      </button>
    </div>
  );
}
```

## 钱包连接

### 使用 Connectors

```tsx
import { useConnect, useAccount } from 'wagmi';

function ConnectWallet() {
  const { connectors, connect, isPending, error } = useConnect();
  const { isConnected } = useAccount();
  
  if (isConnected) {
    return <AccountInfo />;
  }
  
  return (
    <div>
      <h3>连接钱包</h3>
      {connectors.map((connector) => (
        <button
          key={connector.uid}
          onClick={() => connect({ connector })}
          disabled={isPending}
        >
          {connector.name}
          {isPending && ' (连接中...)'}
        </button>
      ))}
      {error && <p className="error">{error.message}</p>}
    </div>
  );
}
```

### 签名消息

```tsx
import { useSignMessage, useVerifyMessage } from 'wagmi';

function SignMessage() {
  const [message, setMessage] = useState('Hello Web3!');
  
  const { signMessage, data: signature, isPending } = useSignMessage();
  
  const { data: isValid } = useVerifyMessage({
    message,
    signature,
  });
  
  return (
    <div>
      <input 
        value={message} 
        onChange={(e) => setMessage(e.target.value)} 
      />
      <button 
        onClick={() => signMessage({ message })}
        disabled={isPending}
      >
        {isPending ? '签名中...' : '签名'}
      </button>
      
      {signature && (
        <div>
          <p>签名: {signature}</p>
          <p>验证: {isValid ? '有效' : '无效'}</p>
        </div>
      )}
    </div>
  );
}
```

## 开发者工具

### TanStack Query DevTools

```tsx
// src/main.tsx
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <WagmiProvider config={config}>
      <QueryClientProvider client={queryClient}>
        <App />
        <ReactQueryDevtools initialIsOpen={false} />
      </QueryClientProvider>
    </WagmiProvider>
  </React.StrictMode>
);
```

### wagmi DevTools

```tsx
import { WagmiDevTools } from '@wagmi/cli/plugins/react';

// 在 App 中使用
function App() {
  return (
    <>
      {/* 你的应用 */}
      <WagmiDevTools />
    </>
  );
}
```

## 完整示例

```tsx
// src/App.tsx
import { useState } from 'react';
import { useAccount, useBalance, useReadContract, useWriteContract } from 'wagmi';
import { parseUnits, formatUnits } from 'viem';

const TOKEN_ADDRESS = '0x...' as `0x${string}`;
const TOKEN_ABI = [...] as const;

function App() {
  const { address, isConnected } = useAccount();
  const [amount, setAmount] = useState('');
  
  // 读取余额
  const { data: balance } = useReadContract({
    address: TOKEN_ADDRESS,
    abi: TOKEN_ABI,
    functionName: 'balanceOf',
    args: [address],
    query: {
      enabled: !!address,
      select: (data) => formatUnits(data as bigint, 18),
    },
  });
  
  // 读取 ETH 余额
  const { data: ethBalance } = useBalance({
    address,
    query: { enabled: !!address },
  });
  
  // 写入合约
  const { writeContract, isPending } = useWriteContract();
  
  const handleApprove = () => {
    writeContract({
      address: TOKEN_ADDRESS,
      abi: TOKEN_ABI,
      functionName: 'approve',
      args: [address!, parseUnits(amount, 18)],
    });
  };
  
  if (!isConnected) {
    return <ConnectButton />;
  }
  
  return (
    <div className="app">
      <h1>我的 DApp</h1>
      
      <div className="balance-section">
        <p>ETH: {ethBalance?.formatted}</p>
        <p>Token: {balance}</p>
      </div>
      
      <div className="action-section">
        <input
          placeholder="金额"
          value={amount}
          onChange={(e) => setAmount(e.target.value)}
        />
        <button onClick={handleApprove} disabled={isPending}>
          {isPending ? '处理中...' : '授权'}
        </button>
      </div>
    </div>
  );
}

export default App;
```

## 下一步

[返回目录](README.md)
