# Solana 开发基础

> 理解账户模型，编写高性能链上程序

## Solana 架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                    Solana 架构层次                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  应用层        DApp 前端 (React/Vue)                         │
│                    ↓ JSON RPC                               │
│  ─────────────────────────────────────────────────────────  │
│                                                             │
│  SDK 层        @solana/web3.js  |  Anchor Client           │
│                    ↓                ↓                       │
│  ─────────────────────────────────────────────────────────  │
│                                                             │
│  运行时        Sealevel (并行执行引擎)                       │
│                    ↓                                        │
│  ─────────────────────────────────────────────────────────  │
│                                                             │
│  程序层        System Program | Token Program | 你的程序    │
│                    ↓                                        │
│  ─────────────────────────────────────────────────────────  │
│                                                             │
│  账户层        账户数据 (持久化存储)                         │
│                    ↓                                        │
│  ─────────────────────────────────────────────────────────  │
│                                                             │
│  共识层        Proof of History + Tower BFT                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 核心概念：账户模型

### 与 EVM 的根本差异

```
┌─────────────────────────────────────────────────────────────┐
│                  EVM vs Solana 存储模型                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  EVM (Solidity):                                            │
│  ┌─────────────────────────────────────┐                    │
│  │           合约存储                    │                    │
│  │  ┌─────────┬──────────────────────┐ │                    │
│  │  │ key     │ value                │ │                    │
│  │  ├─────────┼──────────────────────┤ │                    │
│  │  │ user1   │ 100                  │ │                    │
│  │  │ user2   │ 200                  │ │                    │
│  │  └─────────┴──────────────────────┘ │                    │
│  │  存储在合约地址下                    │                    │
│  └─────────────────────────────────────┘                    │
│                                                             │
│  Solana:                                                    │
│  ┌──────────────────────────────────────────────────────┐   │
│  │                独立账户                                │   │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐    │   │
│  │  │ Account 1   │ │ Account 2   │ │ Account 3   │    │   │
│  │  │ owner: user │ │ owner: user │ │ owner: prog │    │   │
│  │  │ data: 100   │ │ data: 200   │ │ data: ...   │    │   │
│  │  │ lamports: x │ │ lamports: x │ │ lamports: x │    │   │
│  │  └─────────────┘ └─────────────┘ └─────────────┘    │   │
│  │  每个账户独立存在，程序操作传入的账户                    │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 账户结构

```rust
pub struct Account {
    pub lamports: u64,           // 余额（1 SOL = 10^9 lamports）
    pub data: Vec<u8>,           // 数据（可序列化为任意结构）
    pub owner: Pubkey,           // 所有者程序
    pub executable: bool,        // 是否为可执行程序
    pub rent_epoch: u64,         // 租金周期
}
```

### Solidity vs Solana 数据存储

**Solidity（合约内存储）：**

```solidity
contract Token {
    mapping(address => uint256) public balances;
    
    function transfer(address to, uint256 amount) public {
        balances[msg.sender] -= amount;
        balances[to] += amount;
    }
}
```

**Solana（独立账户）：**

```rust
use anchor_lang::prelude::*;

#[program]
pub mod token {
    use super::*;
    
    pub fn transfer(ctx: Context<Transfer>, amount: u64) -> Result<()> {
        let from = &mut ctx.accounts.from;
        let to = &mut ctx.accounts.to;
        
        from.balance = from.balance.checked_sub(amount)
            .ok_or(ErrorCode::InsufficientBalance)?;
        to.balance = to.balance.checked_add(amount)
            .ok_or(ErrorCode::Overflow)?;
        
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Transfer<'info> {
    #[account(mut, has_one = owner)]
    pub from: Account<'info, TokenAccount>,
    #[account(mut)]
    pub to: Account<'info, TokenAccount>,
    pub owner: Signer<'info>,
}

#[account]
pub struct TokenAccount {
    pub owner: Pubkey,
    pub balance: u64,
}
```

## Anchor 框架

Anchor 是 Solana 开发的事实标准框架，类似 Solidity 的 Hardhat/Foundry。

### 项目结构

```
my-program/
├── Cargo.toml
├── Anchor.toml
├── programs/
│   └── my-program/
│       ├── Cargo.toml
│       └── src/
│           └── lib.rs
├── tests/
│   └── my-program.ts
└── app/
```

### 完整示例：计数器程序

```rust
use anchor_lang::prelude::*;

declare_id!("Counter11111111111111111111111111111111111");

#[program]
pub mod counter {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>, start: u64) -> Result<()> {
        let counter = &mut ctx.accounts.counter;
        counter.count = start;
        counter.authority = ctx.accounts.authority.key();
        Ok(())
    }

    pub fn increment(ctx: Context<Increment>) -> Result<()> {
        let counter = &mut ctx.accounts.counter;
        counter.count = counter.count.checked_add(1)
            .ok_or(ErrorCode::Overflow)?;
        msg!("Count: {}", counter.count);
        Ok(())
    }

    pub fn decrement(ctx: Context<Decrement>) -> Result<()> {
        let counter = &mut ctx.accounts.counter;
        require!(counter.count > 0, ErrorCode::Underflow);
        counter.count -= 1;
        Ok(())
    }

    pub fn reset(ctx: Context<Reset>, new_value: u64) -> Result<()> {
        let counter = &mut ctx.accounts.counter;
        counter.count = new_value;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init,
        payer = authority,
        space = 8 + Counter::INIT_SPACE
    )]
    pub counter: Account<'info, Counter>,
    #[account(mut)]
    pub authority: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct Increment<'info> {
    #[account(mut, has_one = authority)]
    pub counter: Account<'info, Counter>,
    pub authority: Signer<'info>,
}

#[derive(Accounts)]
pub struct Decrement<'info> {
    #[account(mut, has_one = authority)]
    pub counter: Account<'info, Counter>,
    pub authority: Signer<'info>,
}

#[derive(Accounts)]
pub struct Reset<'info> {
    #[account(mut, has_one = authority)]
    pub counter: Account<'info, Counter>,
    pub authority: Signer<'info>,
}

#[account]
#[derive(InitSpace)]
pub struct Counter {
    pub authority: Pubkey,  // 32 bytes
    pub count: u64,         // 8 bytes
}

#[error_code]
pub enum ErrorCode {
    #[msg("Counter overflow")]
    Overflow,
    #[msg("Counter underflow")]
    Underflow,
}
```

**对比 Solidity 实现：**

```solidity
contract Counter {
    address public authority;
    uint256 public count;
    
    event CountUpdated(uint256 newCount);
    
    constructor(uint256 start) {
        authority = msg.sender;
        count = start;
    }
    
    function increment() public {
        count += 1;
        emit CountUpdated(count);
    }
    
    function decrement() public {
        require(count > 0, "Underflow");
        count -= 1;
        emit CountUpdated(count);
    }
    
    function reset(uint256 newValue) public {
        require(msg.sender == authority, "Unauthorized");
        count = newValue;
        emit CountUpdated(count);
    }
}
```

## 账户约束（Constraints）

Anchor 的账户约束类似 Solidity 的 modifier，但功能更强大：

```rust
#[derive(Accounts)]
#[instruction(amount: u64)]
pub struct Transfer<'info> {
    #[account(
        mut,
        seeds = [b"vault", authority.key().as_ref()],
        bump,
        has_one = authority @ ErrorCode::WrongOwner,
        constraint = account.balance >= amount @ ErrorCode::InsufficientBalance
    )]
    pub vault: Account<'info, Vault>,
    
    #[account(
        mut,
        associated_token::mint = mint,
        associated_token::authority = authority
    )]
    pub token_account: Account<'info, TokenAccount>,
    
    #[account(mut)]
    pub authority: Signer<'info>,
    
    pub mint: Account<'info, Mint>,
    pub token_program: Program<'info, Token>,
    pub system_program: Program<'info, System>,
}
```

### 常用约束对照表

| Solidity | Anchor 约束 |
|----------|-------------|
| `require(msg.sender == owner)` | `has_one = owner` |
| `require(balance >= amount)` | `constraint = account.balance >= amount` |
| `onlyOwner modifier` | `#[account(..., has_one = authority)]` |
| `mapping` | PDA (Program Derived Address) |
| `new Contract()` | `init` 约束 |

## PDA（Program Derived Address）

PDA 是 Solana 的核心概念，类似 Solidity 的确定性地址生成：

```rust
#[derive(Accounts)]
#[instruction(user: Pubkey)]
pub struct CreateUser<'info> {
    #[account(
        init,
        payer = payer,
        space = 8 + UserProfile::INIT_SPACE,
        seeds = [b"user", user.as_ref()],
        bump
    )]
    pub user_profile: Account<'info, UserProfile>,
    
    #[account(mut)]
    pub payer: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[account]
#[derive(InitSpace)]
pub struct UserProfile {
    pub user: Pubkey,
    #[max_len(32)]
    pub name: String,
    pub created_at: i64,
}
```

**Solidity 等价概念：**

```solidity
// Solidity 中使用 CREATE2 实现确定性地址
function getExpectedAddress(address user) public pure returns (address) {
    return address(uint160(uint256(keccak256(abi.encodePacked(
        bytes1(0xff),
        address(this),
        uint256(0), // salt
        keccak256(abi.encodePacked(user))
    ))));
}
```

## CPI（Cross-Program Invocation）

CPI 类似 Solidity 的外部合约调用：

```rust
use anchor_spl::token::{self, Transfer, Token, TokenAccount};

#[program]
pub mod my_program {
    use super::*;

    pub fn transfer_tokens(
        ctx: Context<TransferTokens>,
        amount: u64
    ) -> Result<()> {
        let cpi_accounts = Transfer {
            from: ctx.accounts.from.to_account_info(),
            to: ctx.accounts.to.to_account_info(),
            authority: ctx.accounts.authority.to_account_info(),
        };
        
        let cpi_program = ctx.accounts.token_program.to_account_info();
        let cpi_ctx = CpiContext::new(cpi_program, cpi_accounts);
        
        token::transfer(cpi_ctx, amount)?;
        Ok(())
    }
}

#[derive(Accounts)]
pub struct TransferTokens<'info> {
    pub from: Account<'info, TokenAccount>,
    pub to: Account<'info, TokenAccount>,
    pub authority: Signer<'info>,
    pub token_program: Program<'info, Token>,
}
```

**对比 Solidity：**

```solidity
// Solidity 外部调用
IERC20(token).transferFrom(from, to, amount);

// 或使用低级调用
(bool success, ) = token.call(
    abi.encodeWithSignature("transferFrom(address,address,uint256)", from, to, amount)
);
```

## 错误处理

```rust
#[error_code]
pub enum CustomError {
    #[msg("Insufficient balance for transfer")]
    InsufficientBalance,
    #[msg("Unauthorized access")]
    Unauthorized,
    #[msg("Invalid input: {}", reason)]
    InvalidInput { reason: String },
}

pub fn some_function(ctx: Context<SomeStruct>) -> Result<()> {
    require!(
        ctx.accounts.user.balance >= 100,
        CustomError::InsufficientBalance
    );
    
    require!(
        ctx.accounts.user.authority == ctx.accounts.signer.key(),
        CustomError::Unauthorized
    );
    
    Ok(())
}
```

## 事件（Events）

```rust
#[event]
pub struct TransferEvent {
    pub from: Pubkey,
    pub to: Pubkey,
    pub amount: u64,
    pub timestamp: i64,
}

pub fn transfer(ctx: Context<Transfer>, amount: u64) -> Result<()> {
    emit!(TransferEvent {
        from: ctx.accounts.from.key(),
        to: ctx.accounts.to.key(),
        amount,
        timestamp: Clock::get()?.unix_timestamp,
    });
    Ok(())
}
```

## 测试

Anchor 使用 TypeScript/Mocha 进行测试：

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { Counter } from "../target/types/counter";
import { assert } from "chai";

describe("counter", () => {
  const provider = anchor.AnchorProvider.env();
  anchor.setProvider(provider);

  const program = anchor.workspace.Counter as Program<Counter>;
  
  const counter = anchor.web3.Keypair.generate();

  it("Initializes the counter", async () => {
    await program.methods
      .initialize(new anchor.BN(0))
      .accounts({
        counter: counter.publicKey,
        authority: provider.wallet.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
      })
      .signers([counter])
      .rpc();

    const account = await program.account.counter.fetch(counter.publicKey);
    assert.equal(account.count.toNumber(), 0);
  });

  it("Increments the counter", async () => {
    await program.methods
      .increment()
      .accounts({
        counter: counter.publicKey,
        authority: provider.wallet.publicKey,
      })
      .rpc();

    const account = await program.account.counter.fetch(counter.publicKey);
    assert.equal(account.count.toNumber(), 1);
  });
});
```

## 实战：简单代币程序

```rust
use anchor_lang::prelude::*;

declare_id!("Token11111111111111111111111111111111111");

#[program]
pub mod simple_token {
    use super::*;

    pub fn initialize_mint(
        ctx: Context<InitializeMint>,
        decimals: u8,
        initial_supply: u64
    ) -> Result<()> {
        let mint = &mut ctx.accounts.mint;
        mint.authority = ctx.accounts.authority.key();
        mint.decimals = decimals;
        mint.total_supply = initial_supply;

        let treasury = &mut ctx.accounts.treasury;
        treasury.mint = mint.key();
        treasury.owner = ctx.accounts.authority.key();
        treasury.amount = initial_supply;

        Ok(())
    }

    pub fn mint_to(
        ctx: Context<MintTo>,
        amount: u64
    ) -> Result<()> {
        let mint = &mut ctx.accounts.mint;
        mint.total_supply = mint.total_supply
            .checked_add(amount)
            .ok_or(ErrorCode::Overflow)?;

        let recipient = &mut ctx.accounts.recipient;
        recipient.amount = recipient.amount
            .checked_add(amount)
            .ok_or(ErrorCode::Overflow)?;

        emit!(MintEvent {
            to: recipient.key(),
            amount,
            timestamp: Clock::get()?.unix_timestamp,
        });

        Ok(())
    }

    pub fn transfer(
        ctx: Context<Transfer>,
        amount: u64
    ) -> Result<()> {
        require!(
            ctx.accounts.from.amount >= amount,
            ErrorCode::InsufficientBalance
        );

        ctx.accounts.from.amount -= amount;
        ctx.accounts.to.amount = ctx.accounts.to.amount
            .checked_add(amount)
            .ok_or(ErrorCode::Overflow)?;

        emit!(TransferEvent {
            from: ctx.accounts.from.key(),
            to: ctx.accounts.to.key(),
            amount,
            timestamp: Clock::get()?.unix_timestamp,
        });

        Ok(())
    }
}

#[derive(Accounts)]
pub struct InitializeMint<'info> {
    #[account(
        init,
        payer = authority,
        space = 8 + Mint::INIT_SPACE
    )]
    pub mint: Account<'info, Mint>,
    
    #[account(
        init,
        payer = authority,
        space = 8 + TokenAccount::INIT_SPACE
    )]
    pub treasury: Account<'info, TokenAccount>,
    
    #[account(mut)]
    pub authority: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct MintTo<'info> {
    #[account(mut, has_one = authority)]
    pub mint: Account<'info, Mint>,
    #[account(mut)]
    pub recipient: Account<'info, TokenAccount>,
    #[account(mut)]
    pub authority: Signer<'info>,
}

#[derive(Accounts)]
pub struct Transfer<'info> {
    #[account(
        mut,
        has_one = owner,
        constraint = from.mint == to.mint @ ErrorCode::MintMismatch
    )]
    pub from: Account<'info, TokenAccount>,
    #[account(mut)]
    pub to: Account<'info, TokenAccount>,
    pub owner: Signer<'info>,
}

#[account]
#[derive(InitSpace)]
pub struct Mint {
    pub authority: Pubkey,
    pub decimals: u8,
    pub total_supply: u64,
}

#[account]
#[derive(InitSpace)]
pub struct TokenAccount {
    pub mint: Pubkey,
    pub owner: Pubkey,
    pub amount: u64,
}

#[event]
pub struct TransferEvent {
    pub from: Pubkey,
    pub to: Pubkey,
    pub amount: u64,
    pub timestamp: i64,
}

#[event]
pub struct MintEvent {
    pub to: Pubkey,
    pub amount: u64,
    pub timestamp: i64,
}

#[error_code]
pub enum ErrorCode {
    #[msg("Insufficient balance")]
    InsufficientBalance,
    #[msg("Arithmetic overflow")]
    Overflow,
    #[msg("Token mint mismatch")]
    MintMismatch,
}
```

## 关键差异总结

```
┌─────────────────────────────────────────────────────────────┐
│         Solidity vs Solana 开发关键差异                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. 数据存储                                                 │
│     Solidity: 合约内 mapping/storage                        │
│     Solana:  独立账户，程序无状态                             │
│                                                             │
│  2. 账户传递                                                 │
│     Solidity: 隐式（msg.sender, address(this))              │
│     Solana:  显式（必须传入所有操作的账户）                   │
│                                                             │
│  3. 并发执行                                                 │
│     Solidity: 顺序执行                                       │
│     Solana:  并行执行（不同账户的操作可并行）                 │
│                                                             │
│  4. 租金机制                                                 │
│     Solidity: 无（只需 gas）                                 │
│     Solana:  账户需维持最低余额否则被清理                     │
│                                                             │
│  5. 升级性                                                   │
│     Solidity: 需要代理模式                                   │
│     Solana:  程序本身可升级（authority 控制）                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

[下一节：Substrate 框架 →](02-substrate.md)
