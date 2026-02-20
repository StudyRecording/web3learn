# 合约交互

本章详细介绍 DApp 与智能合约的完整交互流程，包括读取、写入和事件监听。

## 交互流程

```
┌─────────────────────────────────────────────────────────────┐
│                    合约交互流程                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  读取操作（Read）                                            │
│  ──────────────                                             │
│  Provider → Contract.method() → 返回数据                    │
│  （无需签名，无 Gas 费用）                                    │
│                                                             │
│  写入操作（Write）                                           │
│  ───────────────                                            │
│  Signer → 估算 Gas → 构建交易 → 签名 → 广播 → 等待确认      │
│  （需要签名，消耗 Gas）                                       │
│                                                             │
│  事件监听（Event）                                           │
│  ───────────────                                            │
│  Contract.on('Event', callback) → 实时回调                  │
│  Contract.queryFilter() → 查询历史事件                       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 1. 合约 ABI 管理

```typescript
// src/contracts/abis/erc20.ts
import { parseAbi } from 'viem'

export const erc20Abi = parseAbi([
  'function name() view returns (string)',
  'function symbol() view returns (string)',
  'function decimals() view returns (uint8)',
  'function totalSupply() view returns (uint256)',
  'function balanceOf(address account) view returns (uint256)',
  'function allowance(address owner, address spender) view returns (uint256)',
  'function approve(address spender, uint256 amount) returns (bool)',
  'function transfer(address to, uint256 amount) returns (bool)',
  'function transferFrom(address from, address to, uint256 amount) returns (bool)',
  
  'event Transfer(address indexed from, address indexed to, uint256 value)',
  'event Approval(address indexed owner, address indexed spender, uint256 value)',
])

export type ERC20Abi = typeof erc20Abi
```

```typescript
// src/contracts/abis/index.ts
export * from './erc20'
export * from './erc721'
export * from './erc1155'

// 合约地址配置
import type { Address } from 'viem'

export const ADDRESSES = {
  1: {
    WETH: '0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2' as Address,
    USDC: '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48' as Address,
    USDT: '0xdAC17F958D2ee523a2206206994597C13D831ec7' as Address,
  },
  11155111: {
    WETH: '0x7b79995e5f793A07Bc00c21412e50Ecae098E7f9' as Address,
  },
} as const

export function getAddress(
  chainId: number,
  symbol: keyof typeof ADDRESSES[1]
): Address | undefined {
  return ADDRESSES[chainId as keyof typeof ADDRESSES]?.[symbol]
}
```

## 2. 读取合约数据

### 基础读取

```typescript
// src/hooks/useERC20Read.ts
import { useReadContract, useReadContracts } from 'wagmi'
import { erc20Abi } from '../contracts/abis'
import type { Address } from 'viem'

export function useTokenInfo(tokenAddress: Address) {
  const { data: name } = useReadContract({
    address: tokenAddress,
    abi: erc20Abi,
    functionName: 'name',
  })

  const { data: symbol } = useReadContract({
    address: tokenAddress,
    abi: erc20Abi,
    functionName: 'symbol',
  })

  const { data: decimals } = useReadContract({
    address: tokenAddress,
    abi: erc20Abi,
    functionName: 'decimals',
  })

  const { data: totalSupply } = useReadContract({
    address: tokenAddress,
    abi: erc20Abi,
    functionName: 'totalSupply',
  })

  return {
    name,
    symbol,
    decimals,
    totalSupply,
  }
}

export function useTokenBalance(
  tokenAddress: Address,
  ownerAddress: Address | undefined
) {
  const { data, isLoading, refetch } = useReadContract({
    address: tokenAddress,
    abi: erc20Abi,
    functionName: 'balanceOf',
    args: ownerAddress ? [ownerAddress] : undefined,
    query: {
      enabled: !!ownerAddress,
    },
  })

  return {
    balance: data,
    isLoading,
    refetch,
  }
}
```

### 批量读取（Multicall）

```typescript
// src/hooks/useMulticall.ts
import { useReadContracts } from 'wagmi'
import { erc20Abi } from '../contracts/abis'
import type { Address } from 'viem'

export function useTokenBalances(
  tokenAddresses: Address[],
  ownerAddress: Address | undefined
) {
  const { data, isLoading, error } = useReadContracts({
    contracts: tokenAddresses.map((address) => ({
      address,
      abi: erc20Abi,
      functionName: 'balanceOf' as const,
      args: ownerAddress ? [ownerAddress] : undefined,
    })),
    query: {
      enabled: !!ownerAddress && tokenAddresses.length > 0,
    },
  })

  return {
    balances: data?.map((d, i) => ({
      tokenAddress: tokenAddresses[i],
      balance: d.result,
    })),
    isLoading,
    error,
  }
}

export function useAllTokenInfo(tokenAddresses: Address[]) {
  const { data, isLoading } = useReadContracts({
    contracts: tokenAddresses.flatMap((address) => [
      {
        address,
        abi: erc20Abi,
        functionName: 'name' as const,
      },
      {
        address,
        abi: erc20Abi,
        functionName: 'symbol' as const,
      },
      {
        address,
        abi: erc20Abi,
        functionName: 'decimals' as const,
      },
    ]),
  })

  if (!data) return { tokens: [], isLoading }

  const tokens = tokenAddresses.map((address, i) => ({
    address,
    name: data[i * 3]?.result,
    symbol: data[i * 3 + 1]?.result,
    decimals: data[i * 3 + 2]?.result,
  }))

  return { tokens, isLoading }
}
```

### 条件读取与刷新

```typescript
// src/hooks/useConditionalRead.ts
import { useReadContract } from 'wagmi'
import type { Abi, Address } from 'viem'

interface UseConditionalReadOptions {
  address: Address
  abi: Abi
  functionName: string
  args: unknown[]
  enabled?: boolean
  refetchInterval?: number
  staleTime?: number
}

export function useConditionalRead({
  address,
  abi,
  functionName,
  args,
  enabled = true,
  refetchInterval,
  staleTime = 30_000,
}: UseConditionalReadOptions) {
  return useReadContract({
    address,
    abi,
    functionName,
    args,
    query: {
      enabled,
      refetchInterval,
      staleTime,
    },
  })
}

// 带轮询的价格读取
export function useTokenPrice(tokenAddress: Address) {
  return useReadContract({
    address: tokenAddress,
    abi: [
      {
        name: 'getPrice',
        type: 'function',
        stateMutability: 'view',
        inputs: [],
        outputs: [{ type: 'uint256' }],
      },
    ] as const,
    functionName: 'getPrice',
    query: {
      refetchInterval: 10_000,
    },
  })
}
```

## 3. 写入合约

### 基础写入

```typescript
// src/hooks/useERC20Write.ts
import { useWriteContract, useWaitForTransactionReceipt } from 'wagmi'
import { erc20Abi } from '../contracts/abis'
import type { Address } from 'viem'

export function useTransfer() {
  const { writeContract, data: hash, isPending, error, reset } = useWriteContract()

  const { isLoading: isConfirming, isSuccess, data: receipt } = 
    useWaitForTransactionReceipt({ hash })

  const transfer = (
    tokenAddress: Address,
    to: Address,
    amount: bigint
  ) => {
    writeContract({
      address: tokenAddress,
      abi: erc20Abi,
      functionName: 'transfer',
      args: [to, amount],
    })
  }

  return {
    transfer,
    hash,
    isPending,
    isConfirming,
    isSuccess,
    receipt,
    error,
    reset,
  }
}

export function useApprove() {
  const { writeContract, data: hash, isPending, error } = useWriteContract()

  const { isLoading: isConfirming, isSuccess } = 
    useWaitForTransactionReceipt({ hash })

  const approve = (
    tokenAddress: Address,
    spender: Address,
    amount: bigint
  ) => {
    writeContract({
      address: tokenAddress,
      abi: erc20Abi,
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
    error,
  }
}
```

### 带模拟的写入

```typescript
// src/hooks/useSafeWrite.ts
import { useSimulateContract, useWriteContract, useWaitForTransactionReceipt } from 'wagmi'
import type { Abi, Address } from 'viem'

interface UseSafeWriteOptions {
  address: Address
  abi: Abi
  functionName: string
  args: unknown[]
  value?: bigint
  enabled?: boolean
}

export function useSafeWrite({
  address,
  abi,
  functionName,
  args,
  value,
  enabled = true,
}: UseSafeWriteOptions) {
  const { data: simulation, error: simulateError, isFetching: isSimulating } = 
    useSimulateContract({
      address,
      abi,
      functionName,
      args,
      value,
      query: { enabled },
    })

  const { writeContract, data: hash, isPending, error: writeError } = useWriteContract()

  const { isLoading: isConfirming, isSuccess, data: receipt } = 
    useWaitForTransactionReceipt({ hash })

  const write = () => {
    if (!simulation?.request) {
      throw new Error('Simulation not ready')
    }
    writeContract(simulation.request)
  }

  const canWrite = enabled && !!simulation?.request && !simulateError

  return {
    write,
    hash,
    canWrite,
    isSimulating,
    isPending,
    isConfirming,
    isSuccess,
    receipt,
    error: simulateError || writeError,
  }
}
```

### 复杂交易流程

```typescript
// src/hooks/useSwap.ts
import { useWriteContract, useWaitForTransactionReceipt, useReadContract, useAccount } from 'wagmi'
import { erc20Abi } from '../contracts/abis'
import type { Address } from 'viem'

const ROUTER_ABI = [
  {
    name: 'swapExactTokensForTokens',
    type: 'function',
    stateMutability: 'nonpayable',
    inputs: [
      { name: 'amountIn', type: 'uint256' },
      { name: 'amountOutMin', type: 'uint256' },
      { name: 'path', type: 'address[]' },
      { name: 'to', type: 'address' },
      { name: 'deadline', type: 'uint256' },
    ],
    outputs: [{ name: 'amounts', type: 'uint256[]' }],
  },
] as const

export function useSwap(
  routerAddress: Address,
  tokenIn: Address,
  tokenOut: Address
) {
  const { address } = useAccount()
  const { writeContract, data: hash, isPending, error } = useWriteContract()
  const { isLoading: isConfirming, isSuccess } = 
    useWaitForTransactionReceipt({ hash })

  const { data: allowance } = useReadContract({
    address: tokenIn,
    abi: erc20Abi,
    functionName: 'allowance',
    args: address ? [address, routerAddress] : undefined,
    query: { enabled: !!address },
  })

  const approve = (amount: bigint) => {
    writeContract({
      address: tokenIn,
      abi: erc20Abi,
      functionName: 'approve',
      args: [routerAddress, amount],
    })
  }

  const swap = (
    amountIn: bigint,
    amountOutMin: bigint,
    deadline: bigint
  ) => {
    writeContract({
      address: routerAddress,
      abi: ROUTER_ABI,
      functionName: 'swapExactTokensForTokens',
      args: [
        amountIn,
        amountOutMin,
        [tokenIn, tokenOut],
        address!,
        deadline,
      ],
    })
  }

  const needsApproval = allowance !== undefined && allowance < (amountIn ?? 0n)

  return {
    allowance,
    needsApproval,
    approve,
    swap,
    hash,
    isPending,
    isConfirming,
    isSuccess,
    error,
  }
}
```

## 4. 事件监听

### 实时事件监听

```typescript
// src/hooks/useContractEvents.ts
import { useState, useEffect } from 'react'
import { useWatchContractEvent, useContractEvent } from 'wagmi'
import { erc20Abi } from '../contracts/abis'
import type { Address, Log } from 'viem'

interface TransferEvent {
  from: Address
  to: Address
  value: bigint
  transactionHash: string
  blockNumber: bigint
}

export function useWatchTransfers(tokenAddress: Address) {
  const [transfers, setTransfers] = useState<TransferEvent[]>([])

  useWatchContractEvent({
    address: tokenAddress,
    abi: erc20Abi,
    eventName: 'Transfer',
    onLogs(logs) {
      const newTransfers = logs
        .filter((log): log is typeof log & { args: { from: Address; to: Address; value: bigint } } => 
          log.args !== undefined && 'from' in log.args
        )
        .map((log) => ({
          from: log.args.from,
          to: log.args.to,
          value: log.args.value,
          transactionHash: log.transactionHash,
          blockNumber: log.blockNumber ?? 0n,
        }))
      
      setTransfers((prev) => [...newTransfers, ...prev].slice(0, 100))
    },
  })

  return transfers
}

export function useWatchApprovals(tokenAddress: Address, owner?: Address) {
  const [approvals, setApprovals] = useState<Array<{
    owner: Address
    spender: Address
    value: bigint
    hash: string
  }>>([])

  useWatchContractEvent({
    address: tokenAddress,
    abi: erc20Abi,
    eventName: 'Approval',
    args: owner ? { owner } : undefined,
    onLogs(logs) {
      const newApprovals = logs.map((log: any) => ({
        owner: log.args.owner,
        spender: log.args.spender,
        value: log.args.value,
        hash: log.transactionHash,
      }))
      setApprovals((prev) => [...newApprovals, ...prev])
    },
  })

  return approvals
}
```

### 查询历史事件

```typescript
// src/hooks/useEventHistory.ts
import { useContractEvents } from 'wagmi'
import { erc20Abi } from '../contracts/abis'
import type { Address } from 'viem'

export function useTransferHistory(
  tokenAddress: Address,
  fromBlock?: bigint,
  toBlock?: bigint
) {
  const { data, isLoading, error, refetch } = useContractEvents({
    address: tokenAddress,
    abi: erc20Abi,
    eventName: 'Transfer',
    fromBlock,
    toBlock,
  })

  const transfers = data?.map((log) => ({
    from: log.args?.from as Address,
    to: log.args?.to as Address,
    value: log.args?.value as bigint,
    transactionHash: log.transactionHash,
    blockNumber: log.blockNumber,
  }))

  return {
    transfers,
    isLoading,
    error,
    refetch,
  }
}

export function useApprovalHistory(
  tokenAddress: Address,
  owner?: Address
) {
  const { data, isLoading, refetch } = useContractEvents({
    address: tokenAddress,
    abi: erc20Abi,
    eventName: 'Approval',
    args: owner ? { owner } : undefined,
    fromBlock: 'latest',
  })

  return {
    approvals: data,
    isLoading,
    refetch,
  }
}
```

## 5. 合约实例封装

```typescript
// src/lib/contract/ContractManager.ts
import { getContract, type Address, type Abi } from 'viem'
import { useAccount, usePublicClient, useWalletClient } from 'wagmi'

export class ContractManager<TAbi extends Abi> {
  constructor(
    public address: Address,
    public abi: TAbi
  ) {}

  getReadContract(client: ReturnType<typeof usePublicClient>) {
    if (!client) return null
    return getContract({
      address: this.address,
      abi: this.abi,
      client: { public: client },
    })
  }

  getWriteContract(
    client: ReturnType<typeof usePublicClient>,
    walletClient: NonNullable<ReturnType<typeof useWalletClient>['data']>
  ) {
    return getContract({
      address: this.address,
      abi: this.abi,
      client: {
        public: client!,
        wallet: walletClient,
      },
    })
  }
}

// 使用示例
// const erc20Manager = new ContractManager(tokenAddress, erc20Abi)
// const contract = erc20Manager.getReadContract(publicClient)
// const balance = await contract.read.balanceOf(['0x...'])
```

## 6. React Hook 完整封装

```typescript
// src/hooks/useERC20.ts
import { useAccount } from 'wagmi'
import { useTokenInfo, useTokenBalance } from './useERC20Read'
import { useTransfer, useApprove } from './useERC20Write'
import { useWatchTransfers } from './useContractEvents'
import type { Address } from 'viem'

export function useERC20(tokenAddress: Address) {
  const { address: userAddress, isConnected } = useAccount()

  const { name, symbol, decimals, totalSupply } = useTokenInfo(tokenAddress)
  const { balance, isLoading: isBalanceLoading, refetch: refetchBalance } = 
    useTokenBalance(tokenAddress, userAddress)

  const { 
    transfer, 
    hash: transferHash, 
    isPending: isTransferPending, 
    isConfirming: isTransferConfirming,
    isSuccess: isTransferSuccess,
    error: transferError,
    reset: resetTransfer,
  } = useTransfer()

  const {
    approve,
    hash: approveHash,
    isPending: isApprovePending,
    isConfirming: isApproveConfirming,
    isSuccess: isApproveSuccess,
    error: approveError,
    reset: resetApprove,
  } = useApprove()

  const transfers = useWatchTransfers(tokenAddress)

  const handleTransfer = (to: Address, amount: bigint) => {
    if (!isConnected) throw new Error('Wallet not connected')
    transfer(tokenAddress, to, amount)
  }

  const handleApprove = (spender: Address, amount: bigint) => {
    if (!isConnected) throw new Error('Wallet not connected')
    approve(tokenAddress, spender, amount)
  }

  return {
    token: {
      address: tokenAddress,
      name,
      symbol,
      decimals,
      totalSupply,
    },
    balance: {
      value: balance,
      isLoading: isBalanceLoading,
      refetch: refetchBalance,
    },
    transfer: {
      execute: handleTransfer,
      hash: transferHash,
      isPending: isTransferPending,
      isConfirming: isTransferConfirming,
      isSuccess: isTransferSuccess,
      error: transferError,
      reset: resetTransfer,
    },
    approve: {
      execute: handleApprove,
      hash: approveHash,
      isPending: isApprovePending,
      isConfirming: isApproveConfirming,
      isSuccess: isApproveSuccess,
      error: approveError,
      reset: resetApprove,
    },
    events: {
      transfers,
    },
  }
}
```

## 7. 组件示例

```typescript
// src/components/TokenTransfer.tsx
import { useState } from 'react'
import { useERC20 } from '../hooks/useERC20'
import { parseUnits, formatUnits } from 'viem'
import type { Address } from 'viem'

interface TokenTransferProps {
  tokenAddress: Address
}

export function TokenTransfer({ tokenAddress }: TokenTransferProps) {
  const [to, setTo] = useState('')
  const [amount, setAmount] = useState('')

  const { token, balance, transfer, approve } = useERC20(tokenAddress)

  const handleTransfer = async () => {
    if (!to || !amount) return

    const amountWei = parseUnits(amount, token.decimals ?? 18)
    transfer.execute(to as Address, amountWei)
  }

  const isSubmitting = transfer.isPending || transfer.isConfirming

  return (
    <div className="token-transfer">
      <h3>
        {token.name} ({token.symbol})
      </h3>

      <div className="balance">
        余额: {balance.value ? formatUnits(balance.value, token.decimals ?? 18) : '0'}
      </div>

      <div className="form">
        <input
          type="text"
          placeholder="接收地址"
          value={to}
          onChange={(e) => setTo(e.target.value)}
        />
        <input
          type="text"
          placeholder="转账数量"
          value={amount}
          onChange={(e) => setAmount(e.target.value)}
        />
        <button
          onClick={handleTransfer}
          disabled={isSubmitting}
        >
          {isSubmitting ? '处理中...' : '转账'}
        </button>
      </div>

      {transfer.hash && (
        <div className="tx-hash">
          交易哈希: {transfer.hash}
        </div>
      )}

      {transfer.isSuccess && (
        <div className="success">转账成功！</div>
      )}

      {transfer.error && (
        <div className="error">{transfer.error.message}</div>
      )}
    </div>
  )
}
```

## 8. 错误处理

```typescript
// src/lib/contract/errors.ts
import { ContractFunctionRevertedError, BaseError } from 'viem'

export interface ContractError {
  name: string
  reason?: string
  data?: unknown
  userMessage: string
}

export function parseContractError(error: unknown): ContractError {
  if (error instanceof BaseError) {
    const revertError = error.walk(
      (e) => e instanceof ContractFunctionRevertedError
    )

    if (revertError instanceof ContractFunctionRevertedError) {
      const errorName = revertError.data?.errorName ?? 'UnknownError'
      const args = revertError.data?.args

      return {
        name: errorName,
        reason: args ? JSON.stringify(args) : undefined,
        userMessage: getErrorUserMessage(errorName, args),
      }
    }

    return {
      name: error.name,
      reason: error.shortMessage,
      userMessage: error.shortMessage || '合约调用失败',
    }
  }

  return {
    name: 'UnknownError',
    userMessage: '发生未知错误',
  }
}

function getErrorUserMessage(name: string, args?: unknown[]): string {
  const messages: Record<string, string> = {
    InsufficientBalance: '余额不足',
    InsufficientAllowance: '授权额度不足',
    InvalidAmount: '无效的金额',
    TransferFailed: '转账失败',
    Paused: '合约已暂停',
    Unauthorized: '未授权',
    InvalidAddress: '无效地址',
  }

  return messages[name] || `合约错误: ${name}`
}
```

## 最佳实践

1. **分离读写**：读取操作使用 Provider，写入使用 Signer
2. **批量查询**：使用 Multicall 减少 RPC 调用
3. **乐观更新**：写入前更新 UI，失败后回滚
4. **错误解析**：将合约错误转换为用户友好的提示
5. **事件过滤**：合理使用事件过滤器减少监听开销
