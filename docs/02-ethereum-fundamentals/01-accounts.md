# 2.1 以太坊账户

## 账户类型

以太坊有两种账户类型：

```
┌─────────────────────────────────────────────────────────────┐
│                   以太坊账户类型                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  EOA (Externally Owned Account) 外部拥有账户                │
│  ├── 由私钥控制                                             │
│  ├── 用户钱包地址                                           │
│  ├── 可以发起交易                                           │
│  └── 没有代码                                               │
│                                                             │
│  CA (Contract Account) 合约账户                             │
│  ├── 由合约代码控制                                         │
│  ├── 智能合约地址                                           │
│  ├── 不能主动发起交易                                       │
│  └── 包含代码和存储                                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## EOA (外部账户)

### 组成

```
EOA 组成部分

┌─────────────────────────────────────────┐
│                                         │
│  私钥 (Private Key)                     │
│  ├── 256 位随机数                        │
│  ├── 只有你知道                          │
│  ├── 示例: 0x4c9a2e... (64个十六进制字符) │
│  └── 丢失 = 丢失资产                     │
│                                         │
│  公钥 (Public Key)                      │
│  ├── 由私钥推导                          │
│  ├── 椭圆曲线加密 (secp256k1)            │
│  └── 512 位                              │
│                                         │
│  地址 (Address)                         │
│  ├── 公钥的后 160 位                     │
│  ├── 40 个十六进制字符                   │
│  └── 示例: 0x742d35Cc6634C0532925a3b8... │
│                                         │
└─────────────────────────────────────────┘

关系:
私钥 ──ECDSA──> 公钥 ──SHA3-256 + 截取──> 地址
```

### 生成过程

```javascript
// 地址生成过程 (伪代码)

// 1. 生成随机私钥
const privateKey = randomBytes(32); // 256位随机数
// 示例: 0x4c9a2e8b1f3d5e7a9c0b2d4f6e8a0c2b4d6e8f0a2c4b6d8e0f2a4b6c8d0e2f4a

// 2. 从私钥推导公钥 (椭圆曲线)
const publicKey = ec.secp256k1.publicFromPrivate(privateKey);
// 示例: 0x04a34b99dc22b18c1b... (512位，以04开头表示未压缩)

// 3. 对公钥进行 Keccak-256 哈希
const hash = keccak256(publicKey.slice(1)); // 去掉04前缀
// 示例: 0x742d35cc6634c0532925a3b844bc9e759...

// 4. 取后 20 字节作为地址
const address = '0x' + hash.slice(-40);
// 示例: 0x742d35Cc6634C0532925a3b844Bc9e759...

console.log(address);
// 0x742d35Cc6634C0532925a3b844Bc9e759...
```

### 助记词

直接管理私钥很困难，因此有了助记词：

```
助记词 (Mnemonic)

┌─────────────────────────────────────────┐
│                                         │
│  助记词示例:                             │
│  "witch collapse practice feed shame   │
│   open despair creek road again ice    │
│   least"                                │
│                                         │
│  特点:                                   │
│  ├── 12 或 24 个英文单词                 │
│  ├── BIP-39 标准                         │
│  ├── 可派生多个私钥                      │
│  └── 备份只需记录助记词                  │
│                                         │
└─────────────────────────────────────────┘

派生路径:
m/44'/60'/0'/0/0  ──> 第一个地址
m/44'/60'/0'/0/1  ──> 第二个地址
m/44'/60'/0'/0/2  ──> 第三个地址
...

一个助记词可以管理无限个地址
```

### 钱包文件 (Keystore)

```json
// Keystore 文件 (JSON 格式)
{
  "version": 3,
  "id": "04e9bcbb-96fa-4e91-9e1a-...",
  "address": "742d35cc6634c0532925a3b844bc9e759...",
  "crypto": {
    "ciphertext": "a0b1c2d3e4f5...",
    "cipherparams": {
      "iv": "d0e1f2a3b4c5..."
    },
    "cipher": "aes-128-ctr",
    "kdf": "scrypt",
    "kdfparams": {
      "dklen": 32,
      "salt": "e0f1a2b3c4d5...",
      "n": 262144,
      "r": 8,
      "p": 1
    },
    "mac": "c0d1e2f3a4b5..."
  }
}

// 私钥被密码加密存储
// 需要: keystore 文件 + 密码 = 私钥
```

## CA (合约账户)

### 组成

```
合约账户组成

┌─────────────────────────────────────────┐
│                                         │
│  地址 (Address)                         │
│  ├── 部署时确定                         │
│  ├── 由部署者地址 + nonce 计算          │
│  └── 示例: 0x1f9840a85d5aF5bf1...      │
│                                         │
│  余额 (Balance)                         │
│  ├── 持有的 ETH 数量                    │
│  └── 合约可以持有资产                   │
│                                         │
│  代码 (Code)                            │
│  ├── 编译后的字节码                     │
│  ├── 部署后不可修改                     │
│  └── 可以是空 (被销毁后)                │
│                                         │
│  存储 (Storage)                         │
│  ├── 状态变量                           │
│  ├── 键值对存储                         │
│  └── 持久化数据                         │
│                                         │
│  Nonce                                  │
│  ├── 合约创建的合约数量                 │
│  └── 用于生成新合约地址                 │
│                                         │
└─────────────────────────────────────────┘
```

### 合约地址计算

```javascript
// 合约地址计算

// 方法 1: CREATE 操作码
// address = keccak256(sender + nonce)[12:]

function getCreateAddress(sender, nonce) {
  const encoded = rlp.encode([sender, nonce]);
  const hash = keccak256(encoded);
  return '0x' + hash.slice(-40);
}

// 示例:
// 部署者: 0x742d35Cc...
// nonce: 0
// 合约地址: 0xa4e4b1C5...


// 方法 2: CREATE2 操作码 (确定性地计算地址)
// address = keccak256(0xff + sender + salt + keccak256(bytecode))[12:]

function getCreate2Address(sender, salt, bytecode) {
  const hash = keccak256(
    Buffer.concat([
      Buffer.from('ff', 'hex'),
      sender,
      salt,
      keccak256(bytecode)
    ])
  );
  return '0x' + hash.slice(-40);
}

// CREATE2 优势: 可以在部署前知道合约地址
// 用于: 闪电贷、代理合约、工厂模式
```

## EOA vs CA 对比

| 特性 | EOA | CA |
|------|-----|-----|
| 控制方式 | 私钥 | 合约代码 |
| 创建方式 | 生成私钥 | 部署合约 |
| 发起交易 | 可以 | 不能 (只能响应) |
| 执行代码 | 不能 | 可以 |
| 存储数据 | 无 | 有 (Storage) |
| Gas 费用 | 较低 | 较高 (执行代码) |

## 账户状态

以太坊维护一个全局状态树，每个账户都有状态：

```
世界状态 (World State)

                    State Root
                        │
          ┌─────────────┼─────────────┐
          │             │             │
    账户1的状态    账户2的状态    账户3的状态
         │
    ┌────┴────┐
    │         │
  nonce    balance
    │
    ├── codeHash (合约账户)
    └── storageRoot (合约账户)


账户状态字段:
┌─────────────────────────────────────────┐
│                                         │
│  nonce: uint64                          │
│  ├── EOA: 发送的交易数量                │
│  └── CA: 创建的合约数量                 │
│                                         │
│  balance: uint256                       │
│  └── 持有的 Wei 数量                    │
│                                         │
│  codeHash: bytes32                      │
│  ├── EOA: 空字符串的哈希                │
│  └── CA: 合约字节码的哈希               │
│                                         │
│  storageRoot: bytes32                   │
│  ├── EOA: 空树根                        │
│  └── CA: 存储树的根哈希                 │
│                                         │
└─────────────────────────────────────────┘
```

## 账户抽象 (Account Abstraction)

### 传统问题

```
传统 EOA 的限制:

1. 只能用私钥签名
   └── 无法实现多签、社交恢复等

2. 不能批量操作
   └── approve + transferFrom 需要两笔交易

3. 安全性依赖私钥
   └── 私钥丢失 = 资产丢失

4. 隐私问题
   └── 所有交易公开可见
```

### ERC-4337 账户抽象

```
ERC-4337 解决方案:

┌─────────────────────────────────────────┐
│                                         │
│  传统流程:                              │
│  EOA ──签名交易──> 区块链               │
│                                         │
│  ERC-4337 流程:                         │
│  用户 ──UserOperation──> EntryPoint     │
│  EntryPoint ──验证──> 智能合约钱包       │
│  验证通过 ──执行──> 链上操作            │
│                                         │
└─────────────────────────────────────────┘

智能合约钱包功能:
├── 多签 (多个私钥控制一个账户)
├── 社交恢复 (朋友帮助恢复)
├── 交易限额 (每日转账上限)
├── 批量交易 (一笔交易多操作)
├── Gas 代付 (他人支付 Gas)
└── 自定义验证逻辑
```

### 智能合约钱包示例

```solidity
// 简化的智能合约钱包
contract SimpleSmartWallet {
    address public owner;
    mapping(address => bool) public guardians;
    
    event Executed(address to, uint256 value, bytes data);
    
    constructor(address _owner) {
        owner = _owner;
    }
    
    // 执行交易
    function execute(address to, uint256 value, bytes calldata data) 
        public 
    {
        require(msg.sender == owner, "Not owner");
        
        (bool success, ) = to.call{value: value}(data);
        require(success, "Call failed");
        
        emit Executed(to, value, data);
    }
    
    // 社交恢复
    function recover(address newOwner) public {
        // 需要 3 个守护者中的 2 个签名
        // 简化示例，实际需要签名验证
        require(guardians[msg.sender], "Not guardian");
        owner = newOwner;
    }
    
    // 接收 ETH
    receive() external payable {}
}
```

## 实践：创建账户

### 使用 MetaMask

1. 安装 MetaMask 浏览器插件
2. 创建新钱包或导入已有钱包
3. 查看地址和私钥

### 使用 ethers.js

```javascript
import { Wallet } from 'ethers';

// 创建新账户
const wallet = Wallet.createRandom();

console.log('地址:', wallet.address);
console.log('私钥:', wallet.privateKey);
console.log('助记词:', wallet.mnemonic.phrase);

// 从私钥恢复
const walletFromKey = new Wallet('0x私钥...');

// 从助记词恢复
const walletFromMnemonic = Wallet.fromPhrase('witch collapse practice...');
```

## 安全提示

```
┌─────────────────────────────────────────────────────────────┐
│                     私钥安全指南                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  绝对不要:                                                   │
│  ├── 在网上分享私钥/助记词                                  │
│  ├── 存储在云端或邮件中                                     │
│  ├── 截图保存私钥                                          │
│  ├── 在不安全的网站输入                                     │
│  └── 点击不明链接                                           │
│                                                             │
│  推荐做法:                                                   │
│  ├── 使用硬件钱包 (Ledger, Trezor)                          │
│  ├── 助记词写在纸上，多处存放                               │
│  ├── 大额资产使用多签钱包                                   │
│  └── 定期检查授权 (revoke.cash)                             │
│                                                             │
│  记住:                                                      │
│  "Not your keys, not your coins"                           │
│  不是你的私钥，就不是你的资产                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 下一步

[下一节：交易与 Gas →](02-transactions-gas.md)
