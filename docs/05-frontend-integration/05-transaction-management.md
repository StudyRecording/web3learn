# 5.5 交易管理

## 交易生命周期

```
交易完整生命周期

┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  1. 构造交易                                                │
│     ├── 设置参数 (to, value, data, gas 等)                  │
│     └── 估算 Gas                                            │
│                                                             │
│  2. 签名交易                                                │
│     ├── 用户确认                                            │
│     └── 钱包签名                                            │
│                                                             │
│  3. 广播交易                                                │
│     └── 发送到节点                                          │
│                                                             │
│  4. 等待确认                                                │
│     ├── Pending 状态                                        │
│     └── 被打包进区块                                        │
│                                                             │
│  5. 处理结果                                                │
│     ├── 成功：更新 UI                                       │
│     └── 失败：显示错误                                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 构造交易

### 基本交易参数

```typescript
interface TransactionRequest {
  to?: string;                    // 接收地址 (null = 合约部署)
  from?: string;                  // 发送地址
  value?: bigint;                 // ETH 数量 (wei)
  data?: string;                  // 输入数据
  gasLimit?: bigint;              // Gas 上限
  maxFeePerGas?: bigint;          // 最大费用
  maxPriorityFeePerGas?: bigint;  // 小费
  nonce?: number;                 // 交易序号
  chainId?: number;               // 链 ID
  type?: number;                  // 交易类型 (2 = EIP-1559)
}
```

### 估算 Gas

```typescript
import { ethers } from 'ethers';

async function estimateGas(
  contract: ethers.Contract,
  methodName: string,
  args: any[]
): Promise<bigint> {
  try {
    // 获取 Gas 估算
    const gasEstimate = await contract[methodName].estimateGas(...args);
    
    // 增加 20% 缓冲
    const gasWithBuffer = gasEstimate * 120n / 100n;
    
    return gasWithBuffer;
  } catch (error) {
    console.error('Gas 估算失败:', error);
    throw error;
  }
}

// 使用示例
const gasLimit = await estimateGas(tokenContract, 'transfer', [
  recipientAddress,
  parseUnits('100', 18)
]);
```

### 获取 Gas 价格

```typescript
async function getGasPrice(provider: ethers.BrowserProvider) {
  const feeData = await provider.getFeeData();
  
  return {
    maxFeePerGas: feeData.maxFeePerGas,
    maxPriorityFeePerGas: feeData.maxPriorityFeePerGas,
    gasPrice: feeData.gasPrice,  // 旧版交易使用
  };
}

// 使用示例
const gasPrice = await getGasPrice(provider);
console.log('当前 Gas 价格:', {
  maxFee: formatUnits(gasPrice.maxFeePerGas!, 'gwei') + ' gwei',
  priorityFee: formatUnits(gasPrice.maxPriorityFeePerGas!, 'gwei') + ' gwei',
});
```

## 签名交易

### MetaMask 签名

```typescript
// 用户在 MetaMask 中确认签名
async function signTransaction(
  signer: ethers.Signer,
  tx: ethers.TransactionRequest
): Promise<string> {
  try {
    // 这会弹出 MetaMask 确认窗口
    const signedTx = await signer.signTransaction(tx);
    return signedTx;
  } catch (error: any) {
    if (error.code === 4001) {
      throw new Error('用户拒绝了签名请求');
    }
    throw error;
  }
}
```

### 签名消息 (EIP-191)

```typescript
// 签名任意消息
async function signMessage(
  signer: ethers.Signer,
  message: string
): Promise<string> {
  const signature = await signer.signMessage(message);
  return signature;
}

// 验证签名
function verifyMessage(
  message: string,
  signature: string
): string {
  return ethers.verifyMessage(message, signature);
}

// 使用示例
const message = '登录确认: ' + Date.now();
const signature = await signMessage(signer, message);
const signerAddress = verifyMessage(message, signature);
console.log('签名者地址:', signerAddress);
```

### 签名类型化数据 (EIP-712)

```typescript
// EIP-712 类型化签名
const domain = {
  name: 'MyDApp',
  version: '1',
  chainId: 1,
  verifyingContract: '0xContractAddress...'
};

const types = {
  Permit: [
    { name: 'owner', type: 'address' },
    { name: 'spender', type: 'address' },
    { name: 'value', type: 'uint256' },
    { name: 'nonce', type: 'uint256' },
    { name: 'deadline', type: 'uint256' },
  ],
};

const value = {
  owner: '0xOwner...',
  spender: '0xSpender...',
  value: parseUnits('100', 18),
  nonce: 0,
  deadline: Math.floor(Date.now() / 1000) + 3600, // 1小时后过期
};

// 签名类型化数据
async function signTypedData(
  signer: ethers.Signer,
  domain: any,
  types: any,
  value: any
): Promise<string> {
  const signature = await signer.signTypedData(domain, types, value);
  return signature;
}

// 验证类型化签名
function verifyTypedData(
  domain: any,
  types: any,
  value: any,
  signature: string
): string {
  return ethers.verifyTypedData(domain, types, value, signature);
}
```

## 发送交易

### 发送 ETH

```typescript
async function sendETH(
  signer: ethers.Signer,
  to: string,
  amount: string // ETH 数量，如 "0.1"
): Promise<ethers.TransactionReceipt> {
  const tx = await signer.sendTransaction({
    to,
    value: parseEther(amount),
  });
  
  console.log('交易哈希:', tx.hash);
  
  // 等待确认
  const receipt = await tx.wait();
  
  console.log('交易确认，区块:', receipt.blockNumber);
  
  return receipt;
}

// 使用示例
await sendETH(signer, '0xRecipient...', '0.1');
```

### 调用合约方法

```typescript
async function callContractMethod(
  contract: ethers.Contract,
  methodName: string,
  args: any[],
  overrides: ethers.Overrides = {}
): Promise<ethers.TransactionReceipt> {
  // 调用方法
  const tx = await contract[methodName](...args, overrides);
  
  console.log('交易哈希:', tx.hash);
  
  // 等待确认
  const receipt = await tx.wait();
  
  if (receipt.status === 0) {
    throw new Error('交易执行失败');
  }
  
  return receipt;
}

// 使用示例
await callContractMethod(
  tokenContract,
  'transfer',
  ['0xRecipient...', parseUnits('100', 18)],
  { gasLimit: 100000 }
);
```

### 带回调的交易

```typescript
interface TransactionCallbacks {
  onSubmitted?: (hash: string) => void;
  onConfirmed?: (receipt: ethers.TransactionReceipt) => void;
  onError?: (error: Error) => void;
}

async function sendTransactionWithCallbacks(
  contract: ethers.Contract,
  methodName: string,
  args: any[],
  callbacks: TransactionCallbacks = {}
): Promise<ethers.TransactionReceipt | null> {
  try {
    // 发送交易
    const tx = await contract[methodName](...args);
    
    callbacks.onSubmitted?.(tx.hash);
    
    // 等待确认
    const receipt = await tx.wait();
    
    callbacks.onConfirmed?.(receipt);
    
    return receipt;
  } catch (error: any) {
    callbacks.onError?.(error);
    return null;
  }
}

// 使用示例
await sendTransactionWithCallbacks(
  tokenContract,
  'transfer',
  [recipient, amount],
  {
    onSubmitted: (hash) => {
      console.log('交易已提交:', hash);
      showToast('交易已提交，等待确认...');
    },
    onConfirmed: (receipt) => {
      console.log('交易已确认');
      showToast('交易成功！', 'success');
    },
    onError: (error) => {
      console.error('交易失败:', error);
      showToast('交易失败: ' + error.message, 'error');
    },
  }
);
```

## 等待确认

### 等待单个确认

```typescript
// 默认等待 1 个确认
const receipt = await tx.wait();

// 等待多个确认
const receipt = await tx.wait(3); // 等待 3 个区块确认
```

### 监听交易状态

```typescript
import { useWaitForTransactionReceipt } from 'wagmi';

function TransactionStatus({ hash }: { hash: `0x${string}` }) {
  const { data: receipt, isLoading, isSuccess, isError } = useWaitForTransactionReceipt({
    hash,
  });
  
  if (isLoading) {
    return <div>交易确认中...</div>;
  }
  
  if (isError) {
    return <div>交易失败</div>;
  }
  
  if (isSuccess) {
    return (
      <div>
        交易成功！
        <br />
        区块: {receipt.blockNumber}
        <br />
        Gas 使用: {receipt.gasUsed.toString()}
      </div>
    );
  }
  
  return null;
}
```

### 手动轮询

```typescript
async function waitForTransaction(
  provider: ethers.BrowserProvider,
  txHash: string,
  confirmations: number = 1,
  timeout: number = 120000 // 2 分钟
): Promise<ethers.TransactionReceipt> {
  const startTime = Date.now();
  
  while (Date.now() - startTime < timeout) {
    try {
      const receipt = await provider.getTransactionReceipt(txHash);
      
      if (receipt && receipt.confirmations >= confirmations) {
        return receipt;
      }
    } catch (error) {
      // 继续等待
    }
    
    // 等待 2 秒后重试
    await new Promise(resolve => setTimeout(resolve, 2000));
  }
  
  throw new Error('等待交易超时');
}
```

## 处理交易错误

### 常见错误类型

```typescript
// 错误类型处理
function handleTransactionError(error: any): string {
  // 用户拒绝
  if (error.code === 4001) {
    return '用户拒绝了交易';
  }
  
  // 交易被替换
  if (error.code === 'TRANSACTION_REPLACED') {
    if (error.cancelled) {
      return '交易被取消';
    }
    return '交易被替换: ' + error.replacement.hash;
  }
  
  // Gas 不足
  if (error.code === 'UNPREDICTABLE_GAS_LIMIT') {
    return 'Gas 估算失败，交易可能失败';
  }
  
  // Nonce 错误
  if (error.code === 'NONCE_EXPIRED') {
    return '交易 Nonce 已过期';
  }
  
  // 余额不足
  if (error.code === 'INSUFFICIENT_FUNDS') {
    return 'ETH 余额不足';
  }
  
  // 合约执行错误
  if (error.reason) {
    return '合约错误: ' + error.reason;
  }
  
  // 未知错误
  return '交易失败: ' + error.message;
}
```

### 解析合约错误

```typescript
import { ethers } from 'ethers';

async function parseContractError(
  error: any,
  contract: ethers.Contract
): Promise<string> {
  // 尝试解析错误数据
  if (error.data) {
    try {
      const decodedError = contract.interface.parseError(error.data);
      return `错误: ${decodedError?.name} - ${JSON.stringify(decodedError?.args)}`;
    } catch {
      // 无法解析
    }
  }
  
  // 尝试解析 revert 原因
  if (error.reason) {
    return error.reason;
  }
  
  return error.message;
}
```

## 交易队列管理

### Nonce 管理

```typescript
class TransactionQueue {
  private provider: ethers.BrowserProvider;
  private pendingTx: Map<string, number> = new Map();
  
  constructor(provider: ethers.BrowserProvider) {
    this.provider = provider;
  }
  
  async getNextNonce(address: string): Promise<number> {
    // 获取链上 nonce
    const chainNonce = await this.provider.getTransactionCount(address, 'pending');
    
    // 检查本地待处理交易
    const localNonce = this.pendingTx.get(address) || 0;
    
    // 使用较大的 nonce
    const nonce = Math.max(chainNonce, localNonce);
    
    // 更新本地记录
    this.pendingTx.set(address, nonce + 1);
    
    return nonce;
  }
  
  async sendTransaction(
    signer: ethers.Signer,
    tx: ethers.TransactionRequest
  ): Promise<ethers.TransactionResponse> {
    const address = await signer.getAddress();
    
    // 获取正确的 nonce
    const nonce = await this.getNextNonce(address);
    
    // 发送交易
    const response = await signer.sendTransaction({
      ...tx,
      nonce,
    });
    
    return response;
  }
}
```

### 批量交易

```typescript
async function sendBatchTransactions(
  signer: ethers.Signer,
  transactions: ethers.TransactionRequest[]
): Promise<ethers.TransactionReceipt[]> {
  const address = await signer.getAddress();
  const nonce = await signer.getNonce();
  
  const receipts: ethers.TransactionReceipt[] = [];
  
  for (let i = 0; i < transactions.length; i++) {
    const tx = transactions[i];
    
    // 发送交易，指定 nonce
    const response = await signer.sendTransaction({
      ...tx,
      nonce: nonce + i,
    });
    
    // 等待确认
    const receipt = await response.wait();
    receipts.push(receipt);
  }
  
  return receipts;
}

// 使用示例：授权 + 转账
const receipts = await sendBatchTransactions(signer, [
  {
    to: tokenAddress,
    data: tokenInterface.encodeFunctionData('approve', [spender, amount]),
  },
  {
    to: tokenAddress,
    data: tokenInterface.encodeFunctionData('transfer', [recipient, amount]),
  },
]);
```

## 交易加速和取消

### 加速交易

```typescript
async function speedUpTransaction(
  signer: ethers.Signer,
  txHash: string,
  provider: ethers.BrowserProvider
): Promise<ethers.TransactionResponse> {
  // 获取原始交易
  const tx = await provider.getTransaction(txHash);
  
  if (!tx) {
    throw new Error('交易不存在');
  }
  
  // 提高 Gas 价格
  const feeData = await provider.getFeeData();
  const newMaxFeePerGas = (tx.maxFeePerGas! * 110n) / 100n; // 提高 10%
  const newMaxPriorityFeePerGas = (tx.maxPriorityFeePerGas! * 110n) / 100n;
  
  // 发送新交易
  const newTx = await signer.sendTransaction({
    to: tx.to,
    value: tx.value,
    data: tx.data,
    nonce: tx.nonce,
    maxFeePerGas: newMaxFeePerGas,
    maxPriorityFeePerGas: newMaxPriorityFeePerGas,
    gasLimit: tx.gasLimit,
  });
  
  console.log('新交易哈希:', newTx.hash);
  
  return newTx;
}
```

### 取消交易

```typescript
async function cancelTransaction(
  signer: ethers.Signer,
  nonce: number,
  provider: ethers.BrowserProvider
): Promise<ethers.TransactionResponse> {
  // 发送一个给自己、金额为 0 的交易
  const address = await signer.getAddress();
  const feeData = await provider.getFeeData();
  
  const tx = await signer.sendTransaction({
    to: address,
    value: 0,
    nonce,
    maxFeePerGas: feeData.maxFeePerGas,
    maxPriorityFeePerGas: feeData.maxPriorityFeePerGas,
    gasLimit: 21000,
  });
  
  console.log('取消交易哈希:', tx.hash);
  
  return tx;
}
```

## React Hook 封装

```typescript
import { useState, useCallback } from 'react';
import { useAccount, useWalletClient } from 'wagmi';

interface UseTransactionResult {
  isLoading: boolean;
  error: Error | null;
  txHash: string | null;
  receipt: ethers.TransactionReceipt | null;
  execute: () => Promise<void>;
}

function useTransaction(
  contractAddress: string,
  abi: any[],
  methodName: string,
  args: any[]
): UseTransactionResult {
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  const [txHash, setTxHash] = useState<string | null>(null);
  const [receipt, setReceipt] = useState<ethers.TransactionReceipt | null>(null);
  
  const { data: walletClient } = useWalletClient();
  
  const execute = useCallback(async () => {
    if (!walletClient) {
      throw new Error('钱包未连接');
    }
    
    setIsLoading(true);
    setError(null);
    setTxHash(null);
    setReceipt(null);
    
    try {
      // 创建合约实例
      const contract = new ethers.Contract(
        contractAddress,
        abi,
        walletClient as any
      );
      
      // 发送交易
      const tx = await contract[methodName](...args);
      setTxHash(tx.hash);
      
      // 等待确认
      const txReceipt = await tx.wait();
      setReceipt(txReceipt);
    } catch (err: any) {
      setError(err);
    } finally {
      setIsLoading(false);
    }
  }, [contractAddress, abi, methodName, args, walletClient]);
  
  return { isLoading, error, txHash, receipt, execute };
}

// 使用示例
function TransferButton() {
  const { isLoading, error, txHash, receipt, execute } = useTransaction(
    tokenAddress,
    tokenAbi,
    'transfer',
    [recipient, amount]
  );
  
  return (
    <div>
      <button onClick={execute} disabled={isLoading}>
        {isLoading ? '处理中...' : '转账'}
      </button>
      
      {txHash && <p>交易哈希: {txHash}</p>}
      {error && <p className="error">{error.message}</p>}
      {receipt && <p>交易成功！</p>}
    </div>
  );
}
```

## 下一步

[下一节：完整 DApp 示例 →](06-dapp-example.md)
