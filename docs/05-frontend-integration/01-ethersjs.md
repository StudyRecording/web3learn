# ethers.js 基础使用

ethers.js 是目前最流行的以太坊 JavaScript 库，相比 web3.js 更轻量、类型安全更好。

## 安装

```bash
pnpm add ethers
```

## 核心概念

```
┌─────────────────────────────────────────────────────────────┐
│                    ethers.js 架构                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Provider (只读)                                            │
│      │                                                      │
│      ├── JsonRpcProvider    → 连接 RPC 节点                  │
│      ├── BrowserProvider    → 连接浏览器钱包                 │
│      └── InfuraProvider     → 连接 Infura                   │
│                                                             │
│  Signer (读写)                                              │
│      │                                                      │
│      ├── JsonRpcSigner      → 浏览器钱包签名                 │
│      └── Wallet             → 私钥钱包                      │
│                                                             │
│  Contract                                                   │
│      │                                                      │
│      ├── 只读方法            → 通过 Provider 调用            │
│      └── 写入方法            → 通过 Signer 调用              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 1. Provider 连接

```typescript
// src/lib/ethers/provider.ts
import { JsonRpcProvider, BrowserProvider, FallbackProvider } from 'ethers'

// 连接公共 RPC（只读）
export function createPublicProvider(rpcUrl: string): JsonRpcProvider {
  return new JsonRpcProvider(rpcUrl)
}

// 连接浏览器钱包
export async function createBrowserProvider(): Promise<BrowserProvider | null> {
  if (!window.ethereum) {
    console.warn('No wallet detected')
    return null
  }
  return new BrowserProvider(window.ethereum)
}

// 多节点容错
export function createFallbackProvider(rpcUrls: string[]): FallbackProvider {
  const providers = rpcUrls.map(url => ({
    provider: new JsonRpcProvider(url),
    priority: 1,
    stallTimeout: 1000,
    weight: 1
  }))
  return new FallbackProvider(providers)
}

// 网络配置
export const NETWORK_CONFIG = {
  mainnet: {
    chainId: 1,
    name: 'Ethereum Mainnet',
    rpcUrl: 'https://eth.llamarpc.com',
    explorer: 'https://etherscan.io'
  },
  sepolia: {
    chainId: 11155111,
    name: 'Sepolia Testnet',
    rpcUrl: 'https://rpc.sepolia.org',
    explorer: 'https://sepolia.etherscan.io'
  },
  localhost: {
    chainId: 31337,
    name: 'Localhost',
    rpcUrl: 'http://127.0.0.1:8545',
    explorer: ''
  }
} as const

export type NetworkName = keyof typeof NETWORK_CONFIG
```

## 2. 账户与签名

```typescript
// src/lib/ethers/signer.ts
import { BrowserProvider, JsonRpcSigner, Wallet, ethers } from 'ethers'

// 获取浏览器钱包签名者
export async function getBrowserSigner(): Promise<JsonRpcSigner | null> {
  const provider = await createBrowserProvider()
  if (!provider) return null
  
  await provider.send('eth_requestAccounts', [])
  return await provider.getSigner()
}

// 从私钥创建钱包（服务端用，前端禁用）
export function createWalletFromPrivateKey(
  privateKey: string, 
  rpcUrl: string
): Wallet {
  const provider = new JsonRpcProvider(rpcUrl)
  return new Wallet(privateKey, provider)
}

// 从助记词创建钱包
export function createWalletFromMnemonic(
  mnemonic: string,
  path: string = "m/44'/60'/0'/0/0"
): Wallet {
  return Wallet.fromPhrase(mnemonic, path)
}

// 验证签名
export function verifySignature(
  message: string, 
  signature: string
): string {
  return ethers.verifyMessage(message, signature)
}

// 签名消息（用于登录验证）
export async function signMessage(message: string): Promise<string> {
  const signer = await getBrowserSigner()
  if (!signer) throw new Error('No signer available')
  return await signer.signMessage(message)
}

// 签名类型化数据 (EIP-712)
export async function signTypedData(
  domain: {
    name: string
    version: string
    chainId: number
    verifyingContract: string
  },
  types: Record<string, Array<{ name: string; type: string }>>,
  value: Record<string, unknown>
): Promise<string> {
  const signer = await getBrowserSigner()
  if (!signer) throw new Error('No signer available')
  return await signer.signTypedData(domain, types, value)
}
```

## 3. 合约实例

```typescript
// src/lib/ethers/contract.ts
import { Contract, ContractInterface, ethers } from 'ethers'

// ABI 类型定义
export interface ContractABI {
  abi: ethers.Interface | ethers.InterfaceAbi
  bytecode?: string
}

// 创建合约实例（只读）
export function createReadContract(
  address: string,
  abi: ethers.InterfaceAbi,
  provider: ethers.Provider
): Contract {
  return new Contract(address, abi, provider)
}

// 创建合约实例（读写）
export function createWriteContract(
  address: string,
  abi: ethers.InterfaceAbi,
  signer: ethers.Signer
): Contract {
  return new Contract(address, abi, signer)
}

// 解析 ABI
export function parseABI(abi: string | object[]): ethers.Interface {
  return new ethers.Interface(abi)
}

// 获取合约方法列表
export function getContractMethods(abi: ethers.InterfaceAbi): {
  read: string[]
  write: string[]
  events: string[]
} {
  const iface = new ethers.Interface(abi)
  const read: string[] = []
  const write: string[] = []
  const events: string[] = []

  for (const fragment of Object.values(iface.fragments)) {
    if (fragment instanceof ethers.FunctionFragment) {
      if (fragment.constant || fragment.stateMutability === 'view' || fragment.stateMutability === 'pure') {
        read.push(fragment.name)
      } else {
        write.push(fragment.name)
      }
    } else if (fragment instanceof ethers.EventFragment) {
      events.push(fragment.name)
    }
  }

  return { read, write, events }
}
```

## 4. 读取合约数据

```typescript
// src/lib/ethers/read.ts
import { Contract, ethers } from 'ethers'

// 读取单个值
export async function readContractValue<T>(
  contract: Contract,
  methodName: string,
  args: unknown[] = []
): Promise<T> {
  const result = await contract[methodName](...args)
  return result as T
}

// 批量读取（ multicall 模式）
export async function multicallRead(
  contract: Contract,
  calls: Array<{
    methodName: string
    args?: unknown[]
  }>
): Promise<unknown[]> {
  const promises = calls.map(({ methodName, args = [] }) =>
    contract[methodName](...args)
  )
  return Promise.all(promises)
}

// 示例：读取 ERC20 信息
export async function getERC20Info(
  tokenAddress: string,
  userAddress: string,
  provider: ethers.Provider
): Promise<{
  name: string
  symbol: string
  decimals: number
  totalSupply: bigint
  balance: bigint
}> {
  const abi = [
    'function name() view returns (string)',
    'function symbol() view returns (string)',
    'function decimals() view returns (uint8)',
    'function totalSupply() view returns (uint256)',
    'function balanceOf(address) view returns (uint256)'
  ]

  const contract = new Contract(tokenAddress, abi, provider)
  
  const [name, symbol, decimals, totalSupply, balance] = await Promise.all([
    contract.name(),
    contract.symbol(),
    contract.decimals(),
    contract.totalSupply(),
    contract.balanceOf(userAddress)
  ])

  return { name, symbol, decimals, totalSupply, balance }
}

// 使用 staticCall 确保不会发送交易
export async function safeStaticCall(
  contract: Contract,
  methodName: string,
  args: unknown[] = []
): Promise<unknown> {
  const result = await contract[methodName].staticCall(...args)
  return result
}
```

## 5. 写入合约

```typescript
// src/lib/ethers/write.ts
import { Contract, TransactionResponse, TransactionReceipt } from 'ethers'

// 发送交易
export async function writeContract(
  contract: Contract,
  methodName: string,
  args: unknown[] = [],
  overrides: ethers.Overrides = {}
): Promise<TransactionResponse> {
  return await contract[methodName](...args, overrides)
}

// 带等待的写入
export async function writeAndWait(
  contract: Contract,
  methodName: string,
  args: unknown[] = [],
  overrides: ethers.Overrides = {},
  confirmations: number = 1
): Promise<TransactionReceipt> {
  const tx = await writeContract(contract, methodName, args, overrides)
  return await tx.wait(confirmations)
}

// 估算 Gas
export async function estimateGas(
  contract: Contract,
  methodName: string,
  args: unknown[] = []
): Promise<bigint> {
  return await contract[methodName].estimateGas(...args)
}

// 模拟调用（不发送交易，检查是否会失败）
export async function simulateCall(
  contract: Contract,
  methodName: string,
  args: unknown[] = []
): Promise<boolean> {
  try {
    await contract[methodName].staticCall(...args)
    return true
  } catch {
    return false
  }
}

// 示例：ERC20 转账
export async function transferERC20(
  tokenAddress: string,
  to: string,
  amount: bigint,
  signer: ethers.Signer
): Promise<TransactionReceipt> {
  const abi = [
    'function transfer(address to, uint256 amount) returns (bool)'
  ]
  
  const contract = new Contract(tokenAddress, abi, signer)
  const tx = await contract.transfer(to, amount)
  return await tx.wait()
}

// 示例：ERC20 Approve
export async function approveERC20(
  tokenAddress: string,
  spender: string,
  amount: bigint,
  signer: ethers.Signer
): Promise<TransactionReceipt> {
  const abi = [
    'function approve(address spender, uint256 amount) returns (bool)'
  ]
  
  const contract = new Contract(tokenAddress, abi, signer)
  const tx = await contract.approve(spender, amount)
  return await tx.wait()
}
```

## 6. 事件监听

```typescript
// src/lib/ethers/events.ts
import { Contract, ethers, Log } from 'ethers'

// 监听事件
export function listenToEvent(
  contract: Contract,
  eventName: string,
  callback: (...args: unknown[]) => void
): () => void {
  contract.on(eventName, callback)
  
  return () => {
    contract.off(eventName, callback)
  }
}

// 查询历史事件
export async function queryEvents(
  contract: Contract,
  eventName: string,
  fromBlock: number | string = -1000,
  toBlock: number | string = 'latest'
): Promise<ethers.Log[]> {
  const filter = contract.filters[eventName]()
  return await contract.queryFilter(filter, fromBlock, toBlock)
}

// 解析事件日志
export function parseEventLog(
  contract: Contract,
  log: Log
): {
  name: string
  args: ethers.Result
} | null {
  try {
    const parsed = contract.interface.parseLog({
      topics: log.topics,
      data: log.data
    })
    
    if (!parsed) return null
    
    return {
      name: parsed.name,
      args: parsed.args
    }
  } catch {
    return null
  }
}

// 监听 Pending 交易
export async function listenPendingTransactions(
  provider: ethers.Provider,
  callback: (hash: string) => void
): Promise<() => void> {
  provider.on('pending', callback)
  return () => {
    provider.off('pending', callback)
  }
}

// 监听新区块
export async function listenNewBlocks(
  provider: ethers.Provider,
  callback: (blockNumber: number) => void
): Promise<() => void> {
  provider.on('block', callback)
  return () => {
    provider.off('block', callback)
  }
}

// 示例：监听 ERC20 Transfer 事件
export function listenERC20Transfers(
  tokenAddress: string,
  provider: ethers.Provider,
  onTransfer: (from: string, to: string, amount: bigint, event: Log) => void
): () => void {
  const abi = [
    'event Transfer(address indexed from, address indexed to, uint256 value)'
  ]
  
  const contract = new Contract(tokenAddress, abi, provider)
  
  contract.on('Transfer', (from, to, amount, event) => {
    onTransfer(from, to, amount, event.log)
  })
  
  return () => {
    contract.removeAllListeners('Transfer')
  }
}
```

## 7. 工具函数

```typescript
// src/lib/ethers/utils.ts
import { ethers, formatUnits, parseUnits, getAddress } from 'ethers'

// 地址校验和
export function toChecksumAddress(address: string): string {
  return getAddress(address)
}

// 验证地址
export function isValidAddress(address: string): boolean {
  try {
    getAddress(address)
    return true
  } catch {
    return false
  }
}

// 单位转换
export function formatEther(wei: bigint): string {
  return formatUnits(wei, 18)
}

export function parseEther(ether: string): bigint {
  return parseUnits(ether, 18)
}

export function formatToken(amount: bigint, decimals: number): string {
  return formatUnits(amount, decimals)
}

export function parseToken(amount: string, decimals: number): bigint {
  return parseUnits(amount, decimals)
}

// 获取账户余额
export async function getBalance(
  address: string,
  provider: ethers.Provider
): Promise<bigint> {
  return await provider.getBalance(address)
}

// 获取交易计数（nonce）
export async function getTransactionCount(
  address: string,
  provider: ethers.Provider
): Promise<number> {
  return await provider.getTransactionCount(address)
}

// 获取 Gas 价格
export async function getGasPrice(provider: ethers.Provider): Promise<bigint> {
  return await provider.getFeeData().then(f => f.gasPrice || 0n)
}

// 获取当前区块号
export async function getBlockNumber(provider: ethers.Provider): Promise<number> {
  return await provider.getBlockNumber()
}

// 等待交易确认
export async function waitForTransaction(
  hash: string,
  provider: ethers.Provider,
  confirmations: number = 1
): Promise<ethers.TransactionReceipt | null> {
  return await provider.waitForTransaction(hash, confirmations)
}

// 格式化交易用于显示
export function formatTransactionForDisplay(tx: ethers.TransactionResponse): {
  hash: string
  from: string
  to: string | null
  value: string
  gasLimit: string
  gasPrice: string
  nonce: number
} {
  return {
    hash: tx.hash,
    from: tx.from,
    to: tx.to,
    value: formatEther(tx.value),
    gasLimit: tx.gasLimit.toString(),
    gasPrice: tx.gasPrice ? formatEther(tx.gasPrice) : '0',
    nonce: tx.nonce
  }
}
```

## 8. React Hook 封装

```typescript
// src/hooks/useEthers.ts
import { useState, useEffect, useCallback } from 'react'
import { BrowserProvider, JsonRpcSigner, ethers } from 'ethers'

interface UseEthersReturn {
  provider: BrowserProvider | null
  signer: JsonRpcSigner | null
  address: string | null
  chainId: number | null
  isConnected: boolean
  isConnecting: boolean
  connect: () => Promise<void>
  disconnect: () => void
  switchChain: (chainId: number) => Promise<void>
  error: Error | null
}

export function useEthers(): UseEthersReturn {
  const [provider, setProvider] = useState<BrowserProvider | null>(null)
  const [signer, setSigner] = useState<JsonRpcSigner | null>(null)
  const [address, setAddress] = useState<string | null>(null)
  const [chainId, setChainId] = useState<number | null>(null)
  const [isConnecting, setIsConnecting] = useState(false)
  const [error, setError] = useState<Error | null>(null)

  const connect = useCallback(async () => {
    if (!window.ethereum) {
      setError(new Error('No wallet detected'))
      return
    }

    setIsConnecting(true)
    setError(null)

    try {
      const browserProvider = new BrowserProvider(window.ethereum)
      await browserProvider.send('eth_requestAccounts', [])
      
      const browserSigner = await browserProvider.getSigner()
      const userAddress = await browserSigner.getAddress()
      const network = await browserProvider.getNetwork()

      setProvider(browserProvider)
      setSigner(browserSigner)
      setAddress(userAddress)
      setChainId(Number(network.chainId))
    } catch (err) {
      setError(err instanceof Error ? err : new Error('Unknown error'))
    } finally {
      setIsConnecting(false)
    }
  }, [])

  const disconnect = useCallback(() => {
    setProvider(null)
    setSigner(null)
    setAddress(null)
    setChainId(null)
  }, [])

  const switchChain = useCallback(async (targetChainId: number) => {
    if (!window.ethereum) return

    try {
      await window.ethereum.request({
        method: 'wallet_switchEthereumChain',
        params: [{ chainId: `0x${targetChainId.toString(16)}` }]
      })
    } catch (switchError: unknown) {
      const error = switchError as { code?: number }
      if (error.code === 4902) {
        throw new Error('Chain not added to wallet')
      }
      throw switchError
    }
  }, [])

  useEffect(() => {
    if (!window.ethereum) return

    const handleAccountsChanged = (accounts: string[]) => {
      if (accounts.length === 0) {
        disconnect()
      } else {
        setAddress(accounts[0])
      }
    }

    const handleChainChanged = (chainIdHex: string) => {
      setChainId(parseInt(chainIdHex, 16))
    }

    window.ethereum.on('accountsChanged', handleAccountsChanged)
    window.ethereum.on('chainChanged', handleChainChanged)

    return () => {
      window.ethereum.removeListener('accountsChanged', handleAccountsChanged)
      window.ethereum.removeListener('chainChanged', handleChainChanged)
    }
  }, [disconnect])

  return {
    provider,
    signer,
    address,
    chainId,
    isConnected: !!address,
    isConnecting,
    connect,
    disconnect,
    switchChain,
    error
  }
}
```

## 9. 类型声明扩展

```typescript
// src/types/ethereum.d.ts
interface EthereumProvider {
  isMetaMask?: boolean
  request: (args: { method: string; params?: unknown[] }) => Promise<unknown>
  on: (event: string, callback: (...args: unknown[]) => void) => void
  removeListener: (event: string, callback: (...args: unknown[]) => void) => void
}

declare global {
  interface Window {
    ethereum?: EthereumProvider
  }
}

export {}
```

## 最佳实践

1. **永远不要在前端存储私钥**：私钥只存在于服务端或硬件钱包
2. **使用 staticCall 预检查**：发送交易前模拟调用，避免浪费 Gas
3. **处理所有错误情况**：用户拒绝签名、网络错误、合约 Revert 等
4. **监听账户和网络变化**：及时更新 UI 状态
5. **批量读取使用 Multicall**：减少 RPC 调用次数
