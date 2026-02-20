# wagmi + viem 现代开发方式

wagmi 是专为 React 设计的以太坊 Hooks 库，配合 viem（TypeScript 优先的以太坊接口）提供最佳的开发体验。

## 为什么选择 wagmi + viem

```
┌─────────────────────────────────────────────────────────────┐
│              wagmi + viem vs ethers.js                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ethers.js                    wagmi + viem                  │
│  ─────────                    ─────────────                 │
│  命令式 API                   声明式 Hooks                   │
│  手动管理状态                 自动缓存/重试                  │
│  需要自己封装                 开箱即用                       │
│  较大的包体积                 更轻量（Tree-shaking）         │
│  运行时类型                   编译时类型检查                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 安装

```bash
bun add wagmi viem @tanstack/react-query @wagmi/connectors
```

## 1. 项目配置

```typescript
// src/wagmi/config.ts
import { http, createConfig } from 'wagmi'
import { mainnet, sepolia, localhost } from 'wagmi/chains'
import { injected, walletConnect, coinbaseWallet } from '@wagmi/connectors'

// WalletConnect 项目 ID（从 https://cloud.walletconnect.com 获取）
const projectId = 'YOUR_PROJECT_ID'

export const config = createConfig({
  chains: [mainnet, sepolia, localhost],
  connectors: [
    injected(), // MetaMask 等浏览器钱包
    walletConnect({ projectId }),
    coinbaseWallet({ appName: 'My DApp' }),
  ],
  transports: {
    [mainnet.id]: http('https://eth.llamarpc.com'),
    [sepolia.id]: http('https://rpc.sepolia.org'),
    [localhost.id]: http('http://127.0.0.1:8545'),
  },
})

// 类型声明
declare module 'wagmi' {
  export interface Register {
    config: typeof config
  }
}
```

```typescript
// src/App.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { WagmiProvider } from 'wagmi'
import { config } from './wagmi/config'

const queryClient = new QueryClient()

function App() {
  return (
    <WagmiProvider config={config}>
      <QueryClientProvider client={queryClient}>
        <YourDApp />
      </QueryClientProvider>
    </WagmiProvider>
  )
}
```

## 2. 链配置

```typescript
// src/wagmi/chains.ts
import { defineChain } from 'viem'

// 自定义链
export const myCustomChain = defineChain({
  id: 12345,
  name: 'My Custom Chain',
  nativeCurrency: {
    decimals: 18,
    name: 'Ether',
    symbol: 'ETH',
  },
  rpcUrls: {
    default: { http: ['https://rpc.mychain.org'] },
    public: { http: ['https://rpc.mychain.org'] },
  },
  blockExplorers: {
    default: { name: 'Explorer', url: 'https://explorer.mychain.org' },
  },
  contracts: {
    multicall3: {
      address: '0xca11bde05977b3631167028862bE2a173976CA11',
      blockCreated: 11_907_934,
    },
  },
})

// 常用测试链
export const SUPPORTED_CHAINS = {
  mainnet: {
    id: 1,
    name: 'Ethereum',
    rpcUrl: 'https://eth.llamarpc.com',
    explorer: 'https://etherscan.io',
  },
  sepolia: {
    id: 11155111,
    name: 'Sepolia',
    rpcUrl: 'https://rpc.sepolia.org',
    explorer: 'https://sepolia.etherscan.io',
  },
  polygon: {
    id: 137,
    name: 'Polygon',
    rpcUrl: 'https://polygon-rpc.com',
    explorer: 'https://polygonscan.com',
  },
  arbitrum: {
    id: 42161,
    name: 'Arbitrum One',
    rpcUrl: 'https://arb1.arbitrum.io/rpc',
    explorer: 'https://arbiscan.io',
  },
} as const
```

## 3. 钱包连接 Hooks

```typescript
// src/hooks/useWallet.ts
import { useAccount, useConnect, useDisconnect, useBalance } from 'wagmi'

// 连接状态
export function useWalletStatus() {
  const { address, isConnected, isConnecting, chain } = useAccount()
  const { connectors, connect, isPending, error } = useConnect()
  const { disconnect } = useDisconnect()
  const { data: balance } = useBalance({ address })

  return {
    address,
    isConnected,
    isConnecting,
    chain,
    balance: balance?.formatted,
    connectors,
    connect,
    disconnect,
    isPending,
    error,
  }
}

// 连接钱包组件
import { Connector } from '@wagmi/connectors'

export function ConnectWallet() {
  const { connectors, connect, isPending, error } = useConnect()
  const { isConnected, address } = useAccount()
  const { disconnect } = useDisconnect()

  if (isConnected && address) {
    return (
      <div>
        <span>{address.slice(0, 6)}...{address.slice(-4)}</span>
        <button onClick={() => disconnect()}>Disconnect</button>
      </div>
    )
  }

  return (
    <div>
      {connectors.map((connector) => (
        <button
          key={connector.uid}
          onClick={() => connect({ connector })}
          disabled={isPending}
        >
          {connector.name}
        </button>
      ))}
      {error && <p>{error.message}</p>}
    </div>
  )
}
```

## 4. 网络切换

```typescript
// src/hooks/useNetwork.ts
import { useSwitchChain, useChainId, useAccount } from 'wagmi'
import { mainnet, sepolia } from 'wagmi/chains'

export function useNetwork() {
  const chainId = useChainId()
  const { chain } = useAccount()
  const { switchChain, isPending, error } = useSwitchChain()

  const switchToMainnet = () => switchChain({ chainId: mainnet.id })
  const switchToSepolia = () => switchChain({ chainId: sepolia.id })

  return {
    chainId,
    chain,
    switchChain,
    switchToMainnet,
    switchToSepolia,
    isPending,
    error,
    isSupported: chain?.id === mainnet.id || chain?.id === sepolia.id,
  }
}

// 网络选择组件
export function NetworkSelector() {
  const { chain } = useAccount()
  const { switchChain, isPending } = useSwitchChain()

  const networks = [
    { id: 1, name: 'Ethereum' },
    { id: 11155111, name: 'Sepolia' },
  ]

  return (
    <select
      value={chain?.id}
      onChange={(e) => switchChain({ chainId: Number(e.target.value) })}
      disabled={isPending}
    >
      {networks.map((network) => (
        <option key={network.id} value={network.id}>
          {network.name}
        </option>
      ))}
    </select>
  )
}
```

## 5. 读取合约数据

```typescript
// src/hooks/useContractRead.ts
import { useReadContract, useReadContracts } from 'wagmi'
import { type Address, type Abi } from 'viem'

// 读取单个合约方法
export function useTokenInfo(tokenAddress: Address) {
  const abi = [
    {
      name: 'name',
      type: 'function',
      stateMutability: 'view',
      inputs: [],
      outputs: [{ type: 'string' }],
    },
    {
      name: 'symbol',
      type: 'function',
      stateMutability: 'view',
      inputs: [],
      outputs: [{ type: 'string' }],
    },
    {
      name: 'decimals',
      type: 'function',
      stateMutability: 'view',
      inputs: [],
      outputs: [{ type: 'uint8' }],
    },
    {
      name: 'totalSupply',
      type: 'function',
      stateMutability: 'view',
      inputs: [],
      outputs: [{ type: 'uint256' }],
    },
  ] as const

  const { data: name } = useReadContract({
    address: tokenAddress,
    abi,
    functionName: 'name',
  })

  const { data: symbol } = useReadContract({
    address: tokenAddress,
    abi,
    functionName: 'symbol',
  })

  const { data: decimals } = useReadContract({
    address: tokenAddress,
    abi,
    functionName: 'decimals',
  })

  const { data: totalSupply } = useReadContract({
    address: tokenAddress,
    abi,
    functionName: 'totalSupply',
  })

  return {
    name,
    symbol,
    decimals,
    totalSupply,
  }
}

// 批量读取（使用 multicall）
export function useBatchRead(
  tokenAddress: Address,
  userAddress: Address
) {
  const abi = [
    {
      name: 'balanceOf',
      type: 'function',
      stateMutability: 'view',
      inputs: [{ name: 'account', type: 'address' }],
      outputs: [{ type: 'uint256' }],
    },
    {
      name: 'allowance',
      type: 'function',
      stateMutability: 'view',
      inputs: [
        { name: 'owner', type: 'address' },
        { name: 'spender', type: 'address' },
      ],
      outputs: [{ type: 'uint256' }],
    },
  ] as const

  const { data, isLoading, error } = useReadContracts({
    contracts: [
      {
        address: tokenAddress,
        abi,
        functionName: 'balanceOf',
        args: [userAddress],
      },
      {
        address: tokenAddress,
        abi,
        functionName: 'allowance',
        args: [userAddress, tokenAddress],
      },
    ],
  })

  return {
    balance: data?.[0]?.result,
    allowance: data?.[1]?.result,
    isLoading,
    error,
  }
}

// 条件读取
export function useConditionalRead(
  contractAddress: Address | undefined,
  abi: Abi,
  functionName: string,
  args: unknown[],
  enabled: boolean = true
) {
  return useReadContract({
    address: contractAddress,
    abi,
    functionName,
    args,
    query: {
      enabled,
    },
  })
}

// 带轮询的读取
export function usePollingRead(
  contractAddress: Address,
  abi: Abi,
  functionName: string,
  args: unknown[],
  interval: number = 5000
) {
  return useReadContract({
    address: contractAddress,
    abi,
    functionName,
    args,
    query: {
      refetchInterval: interval,
    },
  })
}
```

## 6. 写入合约

```typescript
// src/hooks/useContractWrite.ts
import { 
  useWriteContract, 
  useWaitForTransactionReceipt,
  useSimulateContract 
} from 'wagmi'
import { type Address, type Abi } from 'viem'

// 基础写入
export function useTokenTransfer(tokenAddress: Address) {
  const abi = [
    {
      name: 'transfer',
      type: 'function',
      stateMutability: 'nonpayable',
      inputs: [
        { name: 'to', type: 'address' },
        { name: 'amount', type: 'uint256' },
      ],
      outputs: [{ type: 'bool' }],
    },
  ] as const

  const { writeContract, data: hash, isPending, error } = useWriteContract()

  const transfer = (to: Address, amount: bigint) => {
    writeContract({
      address: tokenAddress,
      abi,
      functionName: 'transfer',
      args: [to, amount],
    })
  }

  return {
    transfer,
    hash,
    isPending,
    error,
  }
}

// 带模拟的写入（先模拟再发送）
export function useSafeWrite(
  contractAddress: Address,
  abi: Abi,
  functionName: string,
  args: unknown[]
) {
  const { data: simulation, error: simulateError } = useSimulateContract({
    address: contractAddress,
    abi,
    functionName,
    args,
  })

  const { writeContract, data: hash, isPending } = useWriteContract()

  const { isLoading: isConfirming, isSuccess } = useWaitForTransactionReceipt({
    hash,
  })

  const write = () => {
    if (!simulation?.request) return
    writeContract(simulation.request)
  }

  return {
    write,
    hash,
    isPending,
    isConfirming,
    isSuccess,
    simulateError,
    canWrite: !!simulation?.request && !simulateError,
  }
}

// 完整的写入流程
export function useTokenApprove(
  tokenAddress: Address,
  spender: Address,
  amount: bigint
) {
  const abi = [
    {
      name: 'approve',
      type: 'function',
      stateMutability: 'nonpayable',
      inputs: [
        { name: 'spender', type: 'address' },
        { name: 'amount', type: 'uint256' },
      ],
      outputs: [{ type: 'bool' }],
    },
  ] as const

  const { writeContract, data: hash, isPending, error, reset } = useWriteContract()

  const { isLoading: isConfirming, isSuccess, data: receipt } = 
    useWaitForTransactionReceipt({ hash })

  const approve = () => {
    writeContract({
      address: tokenAddress,
      abi,
      functionName: 'approve',
      args: [spender, amount],
    })
  }

  return {
    approve,
    hash,
    isPending,
    isConfirming,
    isSuccess,
    receipt,
    error,
    reset,
  }
}
```

## 7. 签名功能

```typescript
// src/hooks/useSignature.ts
import { useSignMessage, useSignTypedData, useVerifyMessage } from 'wagmi'

// 签名消息
export function useSignLoginMessage() {
  const { signMessage, data: signature, isPending, error } = useSignMessage()

  const createLoginMessage = (address: Address, nonce: string) => {
    return `Welcome to My DApp!\n\nSign this message to verify your wallet.\n\nWallet: ${address}\nNonce: ${nonce}`
  }

  const signLogin = (message: string) => {
    signMessage({ message })
  }

  return {
    createLoginMessage,
    signLogin,
    signature,
    isPending,
    error,
  }
}

// EIP-712 类型化签名
export function useTypedDataSign() {
  const { signTypedData, data: signature, isPending } = useSignTypedData()

  const domain = {
    name: 'My DApp',
    version: '1',
    chainId: 1,
    verifyingContract: '0x...' as Address,
  }

  const types = {
    Permit: [
      { name: 'owner', type: 'address' },
      { name: 'spender', type: 'address' },
      { name: 'value', type: 'uint256' },
      { name: 'nonce', type: 'uint256' },
      { name: 'deadline', type: 'uint256' },
    ],
  }

  const signPermit = (owner: Address, spender: Address, value: bigint, nonce: bigint, deadline: bigint) => {
    signTypedData({
      domain,
      types,
      primaryType: 'Permit',
      message: {
        owner,
        spender,
        value,
        nonce,
        deadline,
      },
    })
  }

  return {
    signPermit,
    signature,
    isPending,
  }
}

// 验证签名
export function useVerifySignature() {
  const { verifyMessage, data: isValid, isPending } = useVerifyMessage()

  const verify = (message: string, signature: string, address: Address) => {
    verifyMessage({
      message,
      signature,
      address,
    })
  }

  return {
    verify,
    isValid,
    isPending,
  }
}
```

## 8. 合约 ABI 管理

```typescript
// src/contracts/abis.ts
import { parseAbi } from 'viem'

// 使用 parseAbi 获得类型安全的 ABI
export const erc20Abi = parseAbi([
  'function name() view returns (string)',
  'function symbol() view returns (string)',
  'function decimals() view returns (uint8)',
  'function totalSupply() view returns (uint256)',
  'function balanceOf(address owner) view returns (uint256)',
  'function allowance(address owner, address spender) view returns (uint256)',
  'function transfer(address to, uint256 amount) returns (bool)',
  'function approve(address spender, uint256 amount) returns (bool)',
  'function transferFrom(address from, address to, uint256 amount) returns (bool)',
  'event Transfer(address indexed from, address indexed to, uint256 value)',
  'event Approval(address indexed owner, address indexed spender, uint256 value)',
])

export const erc721Abi = parseAbi([
  'function name() view returns (string)',
  'function symbol() view returns (string)',
  'function tokenURI(uint256 tokenId) view returns (string)',
  'function balanceOf(address owner) view returns (uint256)',
  'function ownerOf(uint256 tokenId) view returns (address)',
  'function approve(address to, uint256 tokenId)',
  'function getApproved(uint256 tokenId) view returns (address)',
  'function setApprovalForAll(address operator, bool approved)',
  'function isApprovedForAll(address owner, address operator) view returns (bool)',
  'function transferFrom(address from, address to, uint256 tokenId)',
  'function safeTransferFrom(address from, address to, uint256 tokenId)',
  'event Transfer(address indexed from, address indexed to, uint256 indexed tokenId)',
  'event Approval(address indexed owner, address indexed approved, uint256 indexed tokenId)',
])

// 合约地址配置
export const CONTRACT_ADDRESSES = {
  mainnet: {
    WETH: '0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2' as Address,
    USDC: '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48' as Address,
  },
  sepolia: {
    WETH: '0x...' as Address,
  },
} as const

// 获取合约地址
export function getContractAddress(
  chainId: number,
  contractName: keyof typeof CONTRACT_ADDRESSES.mainnet
): Address | undefined {
  const chain = chainId === 1 ? 'mainnet' : chainId === 11155111 ? 'sepolia' : null
  if (!chain) return undefined
  return CONTRACT_ADDRESSES[chain]?.[contractName]
}
```

## 9. 事件监听

```typescript
// src/hooks/useContractEvents.ts
import { useWatchContractEvent, useContractEvents } from 'wagmi'
import { parseAbiItem } from 'viem'

// 实时监听事件
export function useWatchTransfers(tokenAddress: Address) {
  const [transfers, setTransfers] = useState<Array<{
    from: Address
    to: Address
    value: bigint
    hash: string
  }>>([])

  useWatchContractEvent({
    address: tokenAddress,
    abi: parseAbi(['event Transfer(address indexed from, address indexed to, uint256 value)']),
    eventName: 'Transfer',
    onLogs(logs) {
      const newTransfers = logs.map((log) => ({
        from: log.args.from,
        to: log.args.to,
        value: log.args.value,
        hash: log.transactionHash,
      }))
      setTransfers((prev) => [...newTransfers, ...prev])
    },
  })

  return transfers
}

// 查询历史事件
export function useHistoryEvents(
  contractAddress: Address,
  eventName: string,
  fromBlock?: bigint
) {
  const { data, isLoading, error } = useContractEvents({
    address: contractAddress,
    abi: parseAbi(['event Transfer(address indexed from, address indexed to, uint256 value)']),
    eventName,
    fromBlock,
  })

  return {
    events: data,
    isLoading,
    error,
  }
}
```

## 10. 完整的 DApp Hook

```typescript
// src/hooks/useDApp.ts
import { useAccount, useBalance, useReadContract, useWriteContract, useWaitForTransactionReceipt } from 'wagmi'
import { erc20Abi } from '../contracts/abis'
import type { Address } from 'viem'

export interface UseDAppOptions {
  tokenAddress: Address
  spenderAddress?: Address
}

export function useDApp({ tokenAddress, spenderAddress }: UseDAppOptions) {
  const { address, isConnected, chain } = useAccount()
  const { data: ethBalance } = useBalance({ address })

  const { data: tokenBalance } = useReadContract({
    address: tokenAddress,
    abi: erc20Abi,
    functionName: 'balanceOf',
    args: address ? [address] : undefined,
    query: { enabled: !!address },
  })

  const { data: allowance } = useReadContract({
    address: tokenAddress,
    abi: erc20Abi,
    functionName: 'allowance',
    args: address && spenderAddress ? [address, spenderAddress] : undefined,
    query: { enabled: !!address && !!spenderAddress },
  })

  const { writeContract, data: hash, isPending, error } = useWriteContract()

  const { isLoading: isConfirming, isSuccess } = useWaitForTransactionReceipt({ hash })

  const approve = (amount: bigint) => {
    if (!spenderAddress) return
    writeContract({
      address: tokenAddress,
      abi: erc20Abi,
      functionName: 'approve',
      args: [spenderAddress, amount],
    })
  }

  const transfer = (to: Address, amount: bigint) => {
    writeContract({
      address: tokenAddress,
      abi: erc20Abi,
      functionName: 'transfer',
      args: [to, amount],
    })
  }

  return {
    address,
    isConnected,
    chain,
    ethBalance,
    tokenBalance,
    allowance,
    approve,
    transfer,
    hash,
    isPending,
    isConfirming,
    isSuccess,
    error,
  }
}
```

## 最佳实践

1. **使用 parseAbi**：获得完整的类型提示
2. **条件查询**：利用 `query.enabled` 避免无效请求
3. **批量读取**：使用 `useReadContracts` 配合 multicall
4. **错误处理**：每个 Hook 都有 `error` 属性
5. **乐观更新**：结合 `queryClient.setQueryData` 实现
