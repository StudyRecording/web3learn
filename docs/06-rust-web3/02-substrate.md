# Substrate 框架

> 用 Rust 构建自定义区块链

## Substrate 是什么

Substrate 是 Parity Technologies 开发的区块链开发框架，用于构建自定义区块链。Polkadot、Kusama 等知名区块链都基于 Substrate 构建。

```
┌─────────────────────────────────────────────────────────────┐
│                    Substrate 架构层次                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  应用层        Runtime (FRAME Pallets)                      │
│                    ↓                                        │
│  ─────────────────────────────────────────────────────────  │
│                                                             │
│  框架层        FRAME (模块化 Pallet 系统)                    │
│                    ↓                                        │
│  ─────────────────────────────────────────────────────────  │
│                                                             │
│  核心层        Substrate Core                               │
│               - 区块处理                                     │
│               - 共识机制                                     │
│               - 网络层                                       │
│               - 存储层                                       │
│                    ↓                                        │
│  ─────────────────────────────────────────────────────────  │
│                                                             │
│  底层         Rust 标准库 + 共享库                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 核心概念

### Runtime

Runtime 是区块链的业务逻辑层，定义了状态转换规则：

```rust
// runtime/src/lib.rs
#![cfg_attr(not(feature = "std"), no_std)]

pub use frame_support::{
    construct_runtime, parameter_types,
    traits::{ConstU32, ConstU64},
};
pub use frame_system;
pub use pallet_balances;
pub use sp_runtime::{traits::IdentityLookup, MultiSignature};

pub type AccountId = <<MultiSignature as sp_runtime::traits::Verify>::Signer as sp_runtime::traits::IdentifyAccount>::AccountId;
pub type Balance = u128;

impl frame_system::Config for Runtime {
    type AccountId = AccountId;
    type Lookup = IdentityLookup<Self::AccountId>;
    type Block = Block;
    type Hash = sp_core::H256;
    // ... 其他配置
}

impl pallet_balances::Config for Runtime {
    type Balance = Balance;
    type DustRemoval = ();
    type RuntimeEvent = RuntimeEvent;
    type ExistentialDeposit = ConstU128<1_000_000>;
    type AccountStore = System;
    // ... 其他配置
}

construct_runtime!(
    pub enum Runtime {
        System: frame_system,
        Balances: pallet_balances,
        // 添加自定义 pallet
        MyPallet: pallet_my_pallet,
    }
);
```

### Pallet

Pallet 是 Substrate 的模块化单元，类似 Solidity 的合约，但更底层：

```rust
// pallets/my-pallet/src/lib.rs
#![cfg_attr(not(feature = "std"), no_std)]

pub use pallet::*;

#[frame_support::pallet]
pub mod pallet {
    use frame_support::pallet_prelude::*;
    use frame_system::pallet_prelude::*;

    #[pallet::pallet]
    pub struct Pallet<T>(_);

    #[pallet::config]
    pub trait Config: frame_system::Config {
        type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
        #[pallet::constant]
        type MaxClaimLength: Get<u32>;
    }

    #[pallet::storage]
    #[pallet::getter(fn proofs)]
    pub type Proofs<T: Config> = StorageMap<
        _,
        Blake2_128Concat,
        BoundedVec<u8, T::MaxClaimLength>,
        (T::AccountId, BlockNumberFor<T>),
    >;

    #[pallet::event]
    #[pallet::generate_deposit(pub(super) fn deposit_event)]
    pub enum Event<T: Config> {
        ClaimCreated(T::AccountId, BoundedVec<u8, T::MaxClaimLength>),
        ClaimRevoked(T::AccountId, BoundedVec<u8, T::MaxClaimLength>),
    }

    #[pallet::error]
    pub enum Error<T> {
        ProofAlreadyExist,
        ClaimTooLong,
        ClaimNotExist,
        NotClaimOwner,
    }

    #[pallet::call]
    impl<T: Config> Pallet<T> {
        #[pallet::call_index(0)]
        #[pallet::weight(10_000)]
        pub fn create_claim(
            origin: OriginFor<T>,
            claim: BoundedVec<u8, T::MaxClaimLength>,
        ) -> DispatchResult {
            let sender = ensure_signed(origin)?;

            ensure!(!Proofs::<T>::contains_key(&claim), Error::<T>::ProofAlreadyExist);

            Proofs::<T>::insert(
                &claim,
                (sender.clone(), frame_system::Pallet::<T>::block_number()),
            );

            Self::deposit_event(Event::ClaimCreated(sender, claim));
            Ok(())
        }

        #[pallet::call_index(1)]
        #[pallet::weight(10_000)]
        pub fn revoke_claim(
            origin: OriginFor<T>,
            claim: BoundedVec<u8, T::MaxClaimLength>,
        ) -> DispatchResult {
            let sender = ensure_signed(origin)?;

            let (owner, _) = Proofs::<T>::get(&claim).ok_or(Error::<T>::ClaimNotExist)?;
            ensure!(owner == sender, Error::<T>::NotClaimOwner);

            Proofs::<T>::remove(&claim);

            Self::deposit_event(Event::ClaimRevoked(sender, claim));
            Ok(())
        }
    }
}
```

### Solidity vs Substrate Pallet

```
┌─────────────────────────────────────────────────────────────┐
│          Solidity Contract vs Substrate Pallet              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Solidity Contract:         Substrate Pallet:              │
│  ─────────────────         ──────────────────              │
│  contract MyContract {      #[pallet::pallet]               │
│    storage...               pub struct Pallet<T>;           │
│    events...                                              │
│    functions...             #[pallet::storage]              │
│  }                          pub type StorageName<T>         │
│                                                           │
│  mapping(...) storage;      #[pallet::event]                │
│                             pub enum Event<T>               │
│  event Transfer(...)                                      │
│                             #[pallet::call]                 │
│  function transfer(...)     impl<T: Config> Pallet<T>       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 对比表

| 特性 | Solidity | Substrate Pallet |
|------|----------|------------------|
| 语言 | Solidity | Rust |
| 执行 | EVM 字节码 | WASM |
| 状态 | 合约 storage | Storage Map/Value |
| 事件 | event | #[pallet::event] |
| 函数 | function | #[pallet::call] |
| 错误 | require/revert | ensure!/Error enum |
| 权限 | modifier | ensure_signed/root |
| 升级 | 代理模式 | 原生支持（set_code） |

## 存储类型

```rust
#[pallet::storage]
pub type MyValue<T: Config> = StorageValue<_, u128>;

#[pallet::storage]
pub type MyMap<T: Config> = StorageMap<
    _,
    Blake2_128Concat,  // Hash 算法
    T::AccountId,       // Key 类型
    u128,               // Value 类型
    ValueQuery,         // 查询类型
>;

#[pallet::storage]
pub type DoubleMap<T: Config> = StorageDoubleMap<
    _,
    Blake2_128Concat,
    T::AccountId,
    Blake2_128Concat,
    u32,
    u128,
    ValueQuery,
>;

#[pallet::storage]
pub type NMap<T: Config> = StorageNMap<
    _,
    (
        Key<Blake2_128Concat, T::AccountId>,
        Key<Blake2_128Concat, u32>,
        Key<Twox64Concat, u64>,
    ),
    u128,
>;
```

**对比 Solidity：**

```solidity
// Solidity 对应
uint128 public myValue;

mapping(address => uint128) public myMap;

mapping(address => mapping(uint32 => uint128)) public doubleMap;

// NMap 需要嵌套 mapping 或 struct
```

## 权重系统

Substrate 使用权重（Weight）来计算交易成本，类似 Gas：

```rust
#[pallet::call]
impl<T: Config> Pallet<T> {
    #[pallet::call_index(0)]
    #[pallet::weight({
        let dispatch_info = gets_dispatch_info(&call);
        let base_weight = 10_000;
        base_weight.saturating_add(dispatch_info.weight)
    })]
    pub fn complex_operation(
        origin: OriginFor<T>,
        iterations: u32,
    ) -> DispatchResult {
        let _sender = ensure_signed(origin)?;
        
        for _ in 0..iterations {
            // 执行操作
        }
        
        Ok(())
    }
}
```

## 实战：代币 Pallet

```rust
#![cfg_attr(not(feature = "std"), no_std)]

pub use pallet::*;

#[frame_support::pallet]
pub mod pallet {
    use frame_support::pallet_prelude::*;
    use frame_system::pallet_prelude::*;

    #[pallet::pallet]
    #[pallet::without_storage_info]
    pub struct Pallet<T>(_);

    #[pallet::config]
    pub trait Config: frame_system::Config {
        type RuntimeEvent: From<Event<Self>> + IsType<<Self as frame_system::Config>::RuntimeEvent>;
        #[pallet::constant]
        type InitialSupply: Get<Balance>;
    }

    pub type Balance = u128;

    #[pallet::storage]
    #[pallet::getter(fn total_supply)]
    pub type TotalSupply<T: Config> = StorageValue<_, Balance, ValueQuery>;

    #[pallet::storage]
    #[pallet::getter(fn balance_of)]
    pub type Balances<T: Config> = StorageMap<
        _,
        Blake2_128Concat,
        T::AccountId,
        Balance,
        ValueQuery,
    >;

    #[pallet::storage]
    #[pallet::getter(fn allowance)]
    pub type Allowances<T: Config> = StorageDoubleMap<
        _,
        Blake2_128Concat,
        T::AccountId,
        Blake2_128Concat,
        T::AccountId,
        Balance,
        ValueQuery,
    >;

    #[pallet::event]
    #[pallet::generate_deposit(pub(super) fn deposit_event)]
    pub enum Event<T: Config> {
        Transfer {
            from: T::AccountId,
            to: T::AccountId,
            value: Balance,
        },
        Approval {
            owner: T::AccountId,
            spender: T::AccountId,
            value: Balance,
        },
        Mint {
            to: T::AccountId,
            value: Balance,
        },
        Burn {
            from: T::AccountId,
            value: Balance,
        },
    }

    #[pallet::error]
    pub enum Error<T> {
        InsufficientBalance,
        InsufficientAllowance,
        Overflow,
        ZeroTransfer,
    }

    #[pallet::genesis_config]
    pub struct GenesisConfig<T: Config> {
        pub initial_balances: Vec<(T::AccountId, Balance)>,
    }

    #[cfg(feature = "std")]
    impl<T: Config> Default for GenesisConfig<T> {
        fn default() -> Self {
            Self {
                initial_balances: Default::default(),
            }
        }
    }

    #[pallet::genesis_build]
    impl<T: Config> BuildGenesisConfig for GenesisConfig<T> {
        fn build(&self) {
            let mut total: Balance = 0;
            for (account, balance) in &self.initial_balances {
                Balances::<T>::insert(account, balance);
                total = total.saturating_add(*balance);
            }
            TotalSupply::<T>::put(total);
        }
    }

    #[pallet::call]
    impl<T: Config> Pallet<T> {
        #[pallet::call_index(0)]
        #[pallet::weight(10_000)]
        pub fn transfer(
            origin: OriginFor<T>,
            to: T::AccountId,
            value: Balance,
        ) -> DispatchResult {
            let sender = ensure_signed(origin)?;
            ensure!(value > 0, Error::<T>::ZeroTransfer);
            
            Self::_transfer(&sender, &to, value)?;
            
            Self::deposit_event(Event::Transfer {
                from: sender,
                to,
                value,
            });
            
            Ok(())
        }

        #[pallet::call_index(1)]
        #[pallet::weight(10_000)]
        pub fn approve(
            origin: OriginFor<T>,
            spender: T::AccountId,
            value: Balance,
        ) -> DispatchResult {
            let sender = ensure_signed(origin)?;
            
            Allowances::<T>::insert(&sender, &spender, value);
            
            Self::deposit_event(Event::Approval {
                owner: sender,
                spender,
                value,
            });
            
            Ok(())
        }

        #[pallet::call_index(2)]
        #[pallet::weight(10_000)]
        pub fn transfer_from(
            origin: OriginFor<T>,
            from: T::AccountId,
            to: T::AccountId,
            value: Balance,
        ) -> DispatchResult {
            let sender = ensure_signed(origin)?;
            
            let allowance = Allowances::<T>::get(&from, &sender);
            ensure!(allowance >= value, Error::<T>::InsufficientAllowance);
            
            Allowances::<T>::insert(&from, &sender, allowance - value);
            
            Self::_transfer(&from, &to, value)?;
            
            Self::deposit_event(Event::Transfer {
                from,
                to,
                value,
            });
            
            Ok(())
        }

        #[pallet::call_index(3)]
        #[pallet::weight(10_000)]
        pub fn mint(
            origin: OriginFor<T>,
            to: T::AccountId,
            value: Balance,
        ) -> DispatchResult {
            ensure_root(origin)?;
            
            let balance = Balances::<T>::get(&to);
            let new_balance = balance.checked_add(value).ok_or(Error::<T>::Overflow)?;
            Balances::<T>::insert(&to, new_balance);
            
            let supply = TotalSupply::<T>::get();
            TotalSupply::<T>::put(supply.saturating_add(value));
            
            Self::deposit_event(Event::Mint { to, value });
            
            Ok(())
        }

        #[pallet::call_index(4)]
        #[pallet::weight(10_000)]
        pub fn burn(
            origin: OriginFor<T>,
            from: T::AccountId,
            value: Balance,
        ) -> DispatchResult {
            ensure_root(origin)?;
            
            let balance = Balances::<T>::get(&from);
            ensure!(balance >= value, Error::<T>::InsufficientBalance);
            Balances::<T>::insert(&from, balance - value);
            
            let supply = TotalSupply::<T>::get();
            TotalSupply::<T>::put(supply.saturating_sub(value));
            
            Self::deposit_event(Event::Burn { from, value });
            
            Ok(())
        }
    }

    impl<T: Config> Pallet<T> {
        fn _transfer(
            from: &T::AccountId,
            to: &T::AccountId,
            value: Balance,
        ) -> DispatchResult {
            let from_balance = Balances::<T>::get(from);
            ensure!(from_balance >= value, Error::<T>::InsufficientBalance);
            
            let to_balance = Balances::<T>::get(to);
            let new_to_balance = to_balance.checked_add(value).ok_or(Error::<T>::Overflow)?;
            
            Balances::<T>::insert(from, from_balance - value);
            Balances::<T>::insert(to, new_to_balance);
            
            Ok(())
        }
    }
}
```

**对比 Solidity ERC20：**

```solidity
contract ERC20 {
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);

    function transfer(address to, uint256 value) public returns (bool) {
        require(balanceOf[msg.sender] >= value, "Insufficient balance");
        balanceOf[msg.sender] -= value;
        balanceOf[to] += value;
        emit Transfer(msg.sender, to, value);
        return true;
    }

    function approve(address spender, uint256 value) public returns (bool) {
        allowance[msg.sender][spender] = value;
        emit Approval(msg.sender, spender, value);
        return true;
    }

    function transferFrom(address from, address to, uint256 value) public returns (bool) {
        require(value <= balanceOf[from], "Insufficient balance");
        require(value <= allowance[from][msg.sender], "Insufficient allowance");
        balanceOf[from] -= value;
        balanceOf[to] += value;
        allowance[from][msg.sender] -= value;
        emit Transfer(from, to, value);
        return true;
    }
}
```

## 项目结构

```
substrate-node/
├── Cargo.toml
├── node/
│   ├── Cargo.toml
│   └── src/
│       ├── chain_spec.rs
│       ├── cli.rs
│       ├── command.rs
│       ├── main.rs
│       ├── rpc.rs
│       └── service.rs
├── pallets/
│   └── my-pallet/
│       ├── Cargo.toml
│       └── src/
│           ├── lib.rs
│           ├── benchmarking.rs
│           └── weights.rs
├── primitives/
│   ├── Cargo.toml
│   └── src/
│       └── lib.rs
└── runtime/
    ├── Cargo.toml
    └── src/
        └── lib.rs
```

## 开发命令

```bash
# 安装 Substrate
curl https://getsubstrate.io -sSf | bash

# 创建新项目
substrate-new my-node

# 或使用模板
git clone https://github.com/substrate-developer-hub/substrate-node-template
cd substrate-node-template

# 构建
cargo build --release

# 运行开发节点
./target/release/node-template --dev

# 运行测试
cargo test

# 生成 metadata
./target/release/node-template export-metadata > metadata.json
```

## 与前端交互

```typescript
import { ApiPromise, WsProvider } from '@polkadot/api';

async function main() {
  const wsProvider = new WsProvider('ws://127.0.0.1:9944');
  const api = await ApiPromise.create({ provider: wsProvider });

  // 查询存储
  const balance = await api.query.myPallet.balanceOf(accountId);
  console.log('Balance:', balance.toHuman());

  // 发送交易
  const txHash = await api.tx.myPallet
    .transfer(recipient, 1000)
    .signAndSend(alice);
  console.log('Transaction hash:', txHash.toHex());

  // 订阅事件
  api.query.system.events((events) => {
    events.forEach(({ event }) => {
      if (event.section === 'myPallet') {
        console.log('Event:', event.method);
      }
    });
  });
}
```

## 关键差异总结

```
┌─────────────────────────────────────────────────────────────┐
│         Solidity vs Substrate 开发关键差异                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 运行层级                                                 │
│     Solidity: 在 EVM 上运行（智能合约）                       │
│     Substrate: 构建整个区块链（Runtime）                      │
│                                                             │
│  2. 升级机制                                                 │
│     Solidity: 代理模式，复杂且有限                           │
│     Substrate: 原生支持，无分叉升级                           │
│                                                             │
│  3. 性能                                                     │
│     Solidity: 受限于 EVM                                    │
│     Substrate: WASM 原生性能，可并行执行                      │
│                                                             │
│  4. 自定义程度                                               │
│     Solidity: 只能定义合约逻辑                               │
│     Substrate: 共识、加密、治理、全可定制                     │
│                                                             │
│  5. 学习曲线                                                 │
│     Solidity: 相对简单                                       │
│     Substrate: 较陡，需要深入理解区块链原理                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

[下一节：Alloy 库使用 →](03-alloy.md)
