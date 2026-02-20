# 钱包连接

钱包连接是 DApp 的入口，本章介绍如何实现安全、用户友好的钱包连接方案。

## 钱包生态概览

```
┌─────────────────────────────────────────────────────────────┐
│                      钱包类型                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  浏览器扩展钱包                                              │
│  ├── MetaMask          最流行，开发者首选                    │
│  ├── Coinbase Wallet   交易所背景                           │
│  └── Rabby             多链支持好                           │
│                                                             │
│  移动端钱包                                                  │
│  ├── Trust Wallet      支持多链                             │
│  ├── Rainbow           用户体验好                           │
│  └── imToken           国内用户多                           │
│                                                             │
│  硬件钱包                                                    │
│  ├── Ledger            冷存储首选                           │
│  └── Trezor            开源硬件                             │
│                                                             │
│  连接协议                                                    │
│  ├── injected          浏览器注入（MetaMask）                │
│  ├── WalletConnect     跨平台扫码连接                        │
│  └── WalletLink        Coinbase 专用                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 1. MetaMask 连接

```typescript
// src/lib/wallet/metamask.ts
import { detectEthereumProvider } from '@metamask/detect-provider'

export interface MetaMaskProvider {
  isMetaMask: boolean
  request: (args: { method: string; params?: unknown[] }) => Promise<unknown>
  on: (event: string, callback: (...args: unknown[]) => void) => void
  removeListener: (event: string, callback: (...args: unknown[]) => void) => void
  selectedAddress: string | null
  chainId: string
}

export async function detectMetaMask(): Promise<MetaMaskProvider | null> {
  const provider = await detectEthereumProvider()
  
  if (provider && (provider as MetaMaskProvider).isMetaMask) {
    return provider as MetaMaskProvider
  }
  
  return null
}

export async function connectMetaMask(): Promise<{
  address: string
  chainId: number
}> {
  const provider = await detectMetaMask()
  
  if (!provider) {
    throw new Error('MetaMask not installed')
  }

  const accounts = await provider.request({
    method: 'eth_requestAccounts',
  }) as string[]

  if (!accounts || accounts.length === 0) {
    throw new Error('No accounts found')
  }

  const chainId = await provider.request({
    method: 'eth_chainId',
  }) as string

  return {
    address: accounts[0],
    chainId: parseInt(chainId, 16),
  }
}

export function isMetaMaskInstalled(): boolean {
  return typeof window !== 'undefined' && !!window.ethereum?.isMetaMask
}
```

### 监听 MetaMask 事件

```typescript
// src/lib/wallet/metamask-events.ts
import type { MetaMaskProvider } from './metamask'

export interface WalletEvents {
  onAccountsChanged: (callback: (accounts: string[]) => void) => () => void
  onChainChanged: (callback: (chainId: number) => void) => () => void
  onDisconnect: (callback: () => void) => () => void
}

export function subscribeWalletEvents(
  provider: MetaMaskProvider
): WalletEvents {
  return {
    onAccountsChanged(callback) {
      const handler = (accounts: unknown) => {
        callback(accounts as string[])
      }
      provider.on('accountsChanged', handler)
      return () => provider.removeListener('accountsChanged', handler)
    },

    onChainChanged(callback) {
      const handler = (chainIdHex: unknown) => {
        callback(parseInt(chainIdHex as string, 16))
      }
      provider.on('chainChanged', handler)
      return () => provider.removeListener('chainChanged', handler)
    },

    onDisconnect(callback) {
      const handler = () => callback()
      provider.on('disconnect', handler)
      return () => provider.removeListener('disconnect', handler)
    },
  }
}
```

## 2. WalletConnect 集成

```typescript
// src/lib/wallet/walletconnect.ts
import { EthereumProvider, RPC_URLS_MAP } from '@walletconnect/ethereum-provider'
import type { EthereumProviderOptions } from '@walletconnect/ethereum-provider'

export interface WalletConnectConfig {
  projectId: string
  chains: number[]
  rpcMap?: Record<number, string>
  metadata?: {
    name: string
    description: string
    url: string
    icons: string[]
  }
}

export async function initWalletConnect(
  config: WalletConnectConfig
): Promise<EthereumProvider> {
  const options: EthereumProviderOptions = {
    projectId: config.projectId,
    chains: config.chains,
    optionalChains: [1, 137, 42161, 10, 56, 43114],
    rpcMap: config.rpcMap || {},
    metadata: config.metadata || {
      name: 'My DApp',
      description: 'A Web3 Application',
      url: window.location.origin,
      icons: [`${window.location.origin}/logo.png`],
    },
    showQrModal: true,
    qrModalOptions: {
      themeMode: 'dark',
    },
  }

  const provider = await EthereumProvider.init(options)
  return provider
}

export async function connectWalletConnect(
  provider: EthereumProvider
): Promise<{ address: string; chainId: number }> {
  const accounts = await provider.enable()
  
  if (!accounts || accounts.length === 0) {
    throw new Error('No accounts selected')
  }

  return {
    address: accounts[0],
    chainId: provider.chainId,
  }
}

export function disconnectWalletConnect(
  provider: EthereumProvider
): Promise<void> {
  return provider.disconnect()
}
```

## 3. 使用 wagmi 统一管理

```typescript
// src/components/WalletConnect/index.tsx
import { useState } from 'react'
import { useConnect, useAccount, useDisconnect } from 'wagmi'
import { useWeb3Modal, useWeb3ModalState } from '@web3modal/wagmi/react'

export function WalletConnectButton() {
  const { isConnected, address } = useAccount()
  const { disconnect } = useDisconnect()
  const { open } = useWeb3Modal()
  const { selectedNetworkId } = useWeb3ModalState()

  if (isConnected && address) {
    return (
      <div className="wallet-connected">
        <button onClick={() => open({ view: 'Account' })}>
          {address.slice(0, 6)}...{address.slice(-4)}
        </button>
        <button onClick={() => disconnect()}>Disconnect</button>
      </div>
    )
  }

  return (
    <button onClick={() => open()}>
      Connect Wallet
    </button>
  )
}
```

### 自定义钱包选择器

```typescript
// src/components/WalletConnect/ConnectorList.tsx
import { useConnect } from 'wagmi'
import { injected, walletConnect, coinbaseWallet } from '@wagmi/connectors'

const WALLET_ICONS: Record<string, string> = {
  metamask: '/icons/metamask.svg',
  walletconnect: '/icons/walletconnect.svg',
  coinbase: '/icons/coinbase.svg',
  rabby: '/icons/rabby.svg',
}

export function ConnectorList() {
  const { connectors, connect, isPending, error } = useConnect()

  const getConnectorIcon = (name: string): string => {
    const normalizedName = name.toLowerCase()
    for (const [key, icon] of Object.entries(WALLET_ICONS)) {
      if (normalizedName.includes(key)) return icon
    }
    return '/icons/wallet-default.svg'
  }

  return (
    <div className="connector-list">
      {connectors.map((connector) => (
        <button
          key={connector.uid}
          onClick={() => connect({ connector })}
          disabled={isPending}
          className="connector-item"
        >
          <img 
            src={getConnectorIcon(connector.name)} 
            alt={connector.name}
            width={32}
            height={32}
          />
          <span>{connector.name}</span>
        </button>
      ))}
      {error && <div className="error">{error.message}</div>}
    </div>
  )
}
```

## 4. 网络切换与添加

```typescript
// src/lib/wallet/network-switch.ts
import { SwitchChainErrorType, UserRejectedRequestError } from 'viem'

export interface AddChainParams {
  chainId: number
  chainName: string
  nativeCurrency: {
    name: string
    symbol: string
    decimals: number
  }
  rpcUrls: string[]
  blockExplorerUrls?: string[]
}

export async function addNetwork(
  provider: { request: (args: { method: string; params: unknown[] }) => Promise<unknown> },
  params: AddChainParams
): Promise<void> {
  await provider.request({
    method: 'wallet_addEthereumChain',
    params: [{
      chainId: `0x${params.chainId.toString(16)}`,
      chainName: params.chainName,
      nativeCurrency: params.nativeCurrency,
      rpcUrls: params.rpcUrls,
      blockExplorerUrls: params.blockExplorerUrls,
    }],
  })
}

export async function switchNetwork(
  provider: { request: (args: { method: string; params: unknown[] }) => Promise<unknown> },
  chainId: number
): Promise<void> {
  try {
    await provider.request({
      method: 'wallet_switchEthereumChain',
      params: [{ chainId: `0x${chainId.toString(16)}` }],
    })
  } catch (error: unknown) {
    const switchError = error as { code?: number }
    if (switchError.code === 4902) {
      throw new Error('Chain not added')
    }
    throw error
  }
}

// 常用网络配置
export const NETWORK_PARAMS: Record<number, AddChainParams> = {
  137: {
    chainId: 137,
    chainName: 'Polygon Mainnet',
    nativeCurrency: { name: 'MATIC', symbol: 'MATIC', decimals: 18 },
    rpcUrls: ['https://polygon-rpc.com'],
    blockExplorerUrls: ['https://polygonscan.com'],
  },
  42161: {
    chainId: 42161,
    chainName: 'Arbitrum One',
    nativeCurrency: { name: 'Ether', symbol: 'ETH', decimals: 18 },
    rpcUrls: ['https://arb1.arbitrum.io/rpc'],
    blockExplorerUrls: ['https://arbiscan.io'],
  },
  56: {
    chainId: 56,
    chainName: 'BNB Smart Chain',
    nativeCurrency: { name: 'BNB', symbol: 'BNB', decimals: 18 },
    rpcUrls: ['https://bsc-dataseed.binance.org'],
    blockExplorerUrls: ['https://bscscan.com'],
  },
}
```

## 5. React Hook 封装

```typescript
// src/hooks/useWalletConnect.ts
import { useState, useCallback, useEffect } from 'react'
import { useAccount, useConnect, useDisconnect, useSwitchChain, useChainId } from 'wagmi'
import type { Connector } from '@wagmi/connectors'

export interface UseWalletConnectResult {
  address: `0x${string}` | undefined
  chainId: number
  isConnected: boolean
  isConnecting: boolean
  isReconnecting: boolean
  connectors: Connector[]
  activeConnector: Connector | undefined
  connect: (connector: Connector) => Promise<void>
  disconnect: () => void
  switchChain: (chainId: number) => Promise<void>
  error: Error | null
}

export function useWalletConnect(): UseWalletConnectResult {
  const { address, isConnected, isConnecting, isReconnecting, connector } = useAccount()
  const { connectors, connect, isPending, error: connectError } = useConnect()
  const { disconnect } = useDisconnect()
  const { switchChain, error: switchError } = useSwitchChain()
  const chainId = useChainId()
  const [error, setError] = useState<Error | null>(null)

  const handleConnect = useCallback(async (connector: Connector) => {
    setError(null)
    try {
      connect({ connector })
    } catch (err) {
      setError(err instanceof Error ? err : new Error('Connect failed'))
    }
  }, [connect])

  const handleSwitchChain = useCallback(async (targetChainId: number) => {
    setError(null)
    try {
      switchChain({ chainId: targetChainId })
    } catch (err) {
      setError(err instanceof Error ? err : new Error('Switch chain failed'))
    }
  }, [switchChain])

  useEffect(() => {
    if (connectError) {
      setError(connectError)
    }
    if (switchError) {
      setError(switchError)
    }
  }, [connectError, switchError])

  return {
    address,
    chainId,
    isConnected,
    isConnecting: isConnecting || isPending,
    isReconnecting,
    connectors,
    activeConnector: connector,
    connect: handleConnect,
    disconnect,
    switchChain: handleSwitchChain,
    error,
  }
}
```

## 6. 错误处理

```typescript
// src/lib/wallet/errors.ts
import { UserRejectedRequestError, SwitchChainError } from 'viem'

export type WalletError = {
  code: number
  message: string
  userMessage: string
}

export const WALLET_ERRORS: Record<number, WalletError> = {
  4001: {
    code: 4001,
    message: 'User rejected the request',
    userMessage: '您拒绝了请求，请重试并确认操作',
  },
  4100: {
    code: 4100,
    message: 'The requested account and/or method has not been authorized',
    userMessage: '请先解锁钱包并授权访问',
  },
  4200: {
    code: 4200,
    message: 'The requested method is not supported',
    userMessage: '当前钱包不支持此功能',
  },
  4900: {
    code: 4900,
    message: 'The provider is disconnected',
    userMessage: '钱包连接已断开，请重新连接',
  },
  4901: {
    code: 4901,
    message: 'The provider is not connected to the requested chain',
    userMessage: '请切换到正确的网络',
  },
  4902: {
    code: 4902,
    message: 'The chain is not recognized',
    userMessage: '当前网络未添加，请在钱包中添加',
  },
  -32002: {
    code: -32002,
    message: 'Request already pending',
    userMessage: '已有待处理的请求，请在钱包中确认',
  },
  -32003: {
    code: -32003,
    message: 'Transaction rejected',
    userMessage: '交易被拒绝',
  },
  -32000: {
    code: -32000,
    message: 'Transaction failed',
    userMessage: '交易执行失败，请检查参数',
  },
}

export function parseWalletError(error: unknown): WalletError {
  if (error instanceof UserRejectedRequestError) {
    return WALLET_ERRORS[4001]
  }
  
  if (error instanceof SwitchChainError) {
    return WALLET_ERRORS[4901]
  }

  if (error && typeof error === 'object' && 'code' in error) {
    const code = (error as { code: number }).code
    return WALLET_ERRORS[code] || {
      code,
      message: String(error),
      userMessage: '发生未知错误，请重试',
    }
  }

  return {
    code: -1,
    message: String(error),
    userMessage: '发生未知错误，请重试',
  }
}
```

### 错误处理组件

```typescript
// src/components/WalletConnect/ErrorDisplay.tsx
import { parseWalletError, type WalletError } from '../../lib/wallet/errors'

interface ErrorDisplayProps {
  error: unknown
  onRetry?: () => void
}

export function ErrorDisplay({ error, onRetry }: ErrorDisplayProps) {
  if (!error) return null

  const walletError = parseWalletError(error)

  return (
    <div className="wallet-error">
      <p className="error-message">{walletError.userMessage}</p>
      <p className="error-code">错误码: {walletError.code}</p>
      {onRetry && (
        <button onClick={onRetry} className="retry-button">
          重试
        </button>
      )}
    </div>
  )
}
```

## 7. 完整连接组件

```typescript
// src/components/WalletConnect/Modal.tsx
import { useState } from 'react'
import { useWalletConnect } from '../../hooks/useWalletConnect'
import { ErrorDisplay } from './ErrorDisplay'
import { NETWORK_PARAMS } from '../../lib/wallet/networkSwitch'

interface WalletModalProps {
  isOpen: boolean
  onClose: () => void
  requiredChainId?: number
}

export function WalletModal({ isOpen, onClose, requiredChainId }: WalletModalProps) {
  const {
    address,
    chainId,
    isConnected,
    isConnecting,
    connectors,
    connect,
    disconnect,
    switchChain,
    error,
  } = useWalletConnect()

  const [pendingConnector, setPendingConnector] = useState<string | null>(null)

  if (!isOpen) return null

  const handleConnect = async (connector: typeof connectors[0]) => {
    setPendingConnector(connector.uid)
    try {
      await connect(connector)
      if (requiredChainId && chainId !== requiredChainId) {
        await switchChain(requiredChainId)
      }
      onClose()
    } finally {
      setPendingConnector(null)
    }
  }

  const handleSwitchNetwork = async () => {
    if (requiredChainId) {
      await switchChain(requiredChainId)
      onClose()
    }
  }

  const isWrongNetwork = requiredChainId && chainId !== requiredChainId

  return (
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal-content" onClick={(e) => e.stopPropagation()}>
        <button className="close-button" onClick={onClose}>×</button>
        
        {!isConnected ? (
          <>
            <h3>连接钱包</h3>
            <div className="connectors">
              {connectors.map((connector) => (
                <button
                  key={connector.uid}
                  onClick={() => handleConnect(connector)}
                  disabled={isConnecting && pendingConnector === connector.uid}
                  className="connector-button"
                >
                  {connector.name}
                  {isConnecting && pendingConnector === connector.uid && '...'}
                </button>
              ))}
            </div>
          </>
        ) : isWrongNetwork ? (
          <>
            <h3>切换网络</h3>
            <p>请切换到 {NETWORK_PARAMS[requiredChainId!]?.chainName || '正确的网络'}</p>
            <button onClick={handleSwitchNetwork} className="switch-button">
              切换网络
            </button>
            <button onClick={disconnect} className="disconnect-button">
              断开连接
            </button>
          </>
        ) : (
          <>
            <h3>已连接</h3>
            <p className="address">{address}</p>
            <p className="network">网络: {NETWORK_PARAMS[chainId]?.chainName || chainId}</p>
            <button onClick={disconnect} className="disconnect-button">
              断开连接
            </button>
          </>
        )}

        {error && <ErrorDisplay error={error} />}
      </div>
    </div>
  )
}
```

## 8. 自动重连

```typescript
// src/hooks/useAutoReconnect.ts
import { useEffect } from 'react'
import { useAccount, useConnect } from 'wagmi'
import { storage } from '../../lib/storage'

const LAST_CONNECTOR_KEY = 'wagmi:lastConnector'

export function useAutoReconnect() {
  const { isConnected, connector } = useAccount()
  const { connectors, connect } = useConnect()

  useEffect(() => {
    if (isConnected && connector) {
      storage.setItem(LAST_CONNECTOR_KEY, connector.uid)
    }
  }, [isConnected, connector])

  useEffect(() => {
    if (isConnected) return

    const lastConnectorId = storage.getItem(LAST_CONNECTOR_KEY)
    if (!lastConnectorId) return

    const lastConnector = connectors.find((c) => c.uid === lastConnectorId)
    if (!lastConnector) return

    const shouldAutoConnect = async () => {
      if (lastConnector.name === 'MetaMask') {
        const accounts = await window.ethereum?.request({ 
          method: 'eth_accounts' 
        }) as string[] | undefined
        return accounts && accounts.length > 0
      }
      return false
    }

    shouldAutoConnect().then((should) => {
      if (should) {
        connect({ connector: lastConnector })
      }
    })
  }, [isConnected, connectors, connect])
}
```

## 9. 检测已安装钱包

```typescript
// src/lib/wallet/detection.ts
export interface DetectedWallet {
  id: string
  name: string
  icon: string
  installed: boolean
  provider: unknown
}

export function detectInstalledWallets(): DetectedWallet[] {
  const wallets: DetectedWallet[] = []

  if (typeof window === 'undefined') return wallets

  const ethereum = (window as Window & { ethereum?: Record<string, unknown> }).ethereum

  if (!ethereum) return wallets

  const walletChecks = [
    {
      id: 'metamask',
      name: 'MetaMask',
      check: (e: Record<string, unknown>) => e.isMetaMask && !e.isBraveWallet,
    },
    {
      id: 'rabby',
      name: 'Rabby',
      check: (e: Record<string, unknown>) => e.isRabby,
    },
    {
      id: 'coinbase',
      name: 'Coinbase Wallet',
      check: (e: Record<string, unknown>) => e.isCoinbaseWallet,
    },
    {
      id: 'brave',
      name: 'Brave Wallet',
      check: (e: Record<string, unknown>) => e.isBraveWallet,
    },
    {
      id: 'trust',
      name: 'Trust Wallet',
      check: (e: Record<string, unknown>) => e.isTrust,
    },
  ]

  for (const wallet of walletChecks) {
    if (wallet.check(ethereum)) {
      wallets.push({
        id: wallet.id,
        name: wallet.name,
        icon: `/icons/${wallet.id}.svg`,
        installed: true,
        provider: ethereum,
      })
    }
  }

  if (wallets.length === 0 && ethereum) {
    wallets.push({
      id: 'injected',
      name: 'Browser Wallet',
      icon: '/icons/wallet-default.svg',
      installed: true,
      provider: ethereum,
    })
  }

  return wallets
}
```

## 最佳实践

1. **提供多种连接方式**：支持 MetaMask、WalletConnect 等
2. **优雅降级**：钱包未安装时提供下载引导
3. **网络检测**：自动检测并提示切换网络
4. **错误友好**：将技术错误转换为用户可理解的提示
5. **自动重连**：刷新页面后自动恢复连接状态
6. **安全存储**：不存储敏感信息，只存储连接器标识
