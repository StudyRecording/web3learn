# 7.4 项目四：DAO 治理

## 项目概述

创建一个完整的 DAO 治理系统，包含提案、投票、执行功能。

```
┌─────────────────────────────────────────────────────────────┐
│                    DAO 架构                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  治理代币 (Governance Token)                                │
│  └── 用于投票权重                                           │
│                                                             │
│  治理合约 (Governor)                                        │
│  ├── 创建提案                                               │
│  ├── 投票                                                   │
│  ├── 统计结果                                               │
│  └── 执行提案                                               │
│                                                             │
│  时间锁 (Timelock)                                          │
│  └── 延迟执行，给用户反应时间                               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 智能合约

### 治理代币

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Votes.sol";

contract GovernanceToken is ERC20Votes {
    constructor() ERC20("Governance Token", "GOV") ERC20Permit("Governance Token") {
        _mint(msg.sender, 1_000_000 * 10**decimals());
    }
}
```

### 时间锁合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/governance/TimelockController.sol";

// 直接使用 OpenZeppelin 的 TimelockController
// 部署时指定:
// - minDelay: 最小延迟时间
// - proposers: 可以提案的地址（Governor 合约）
// - executors: 可以执行的地址（任何人或特定地址）
```

### 治理合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/governance/Governor.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorVotes.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorVotesQuorumFraction.sol";
import "@openzeppelin/contracts/governance/extensions/GovernorTimelockControl.sol";

contract MyDAO is 
    Governor, 
    GovernorVotes, 
    GovernorVotesQuorumFraction,
    GovernorTimelockControl 
{
    constructor(
        IVotes _token,
        TimelockController _timelock
    ) 
        Governor("MyDAO")
        GovernorVotes(_token)
        GovernorVotesQuorumFraction(4) // 4% 法定人数
        GovernorTimelockControl(_timelock)
    {}
    
    // 投票延迟：提案创建后等待多少区块才能投票
    function votingDelay() public pure override returns (uint256) {
        return 1; // 1 个区块
    }
    
    // 投票周期：投票持续多少区块
    function votingPeriod() public pure override returns (uint256) {
        return 45818; // 约 1 周
    }
    
    // 提案门槛：需要多少代币才能创建提案
    function proposalThreshold() public pure override returns (uint256) {
        return 1000 * 10**18; // 1000 GOV
    }
    
    // 重写以支持时间锁
    function supportsInterface(
        bytes4 interfaceId
    ) public view override(Governor, GovernorTimelockControl) returns (bool) {
        return super.supportsInterface(interfaceId);
    }
    
    function _executeOperations(
        address target,
        uint256 value,
        bytes calldata data,
        bytes32 predecessor,
        bytes32 salt
    ) internal override(Governor, GovernorTimelockControl) {
        super._executeOperations(target, value, data, predecessor, salt);
    }
    
    function _cancel(
        address[] calldata targets,
        uint256[] calldata values,
        bytes[] calldata calldatas,
        bytes32 descriptionHash
    ) internal override(Governor, GovernorTimelockControl) returns (bytes32) {
        return super._cancel(targets, values, calldatas, descriptionHash);
    }
    
    function _executor() internal view override(Governor, GovernorTimelockControl) returns (address) {
        return super._executor();
    }
}
```

### 简化版 DAO

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";

contract SimpleDAO {
    using SafeERC20 for IERC20;
    
    IERC20 public governanceToken;
    
    struct Proposal {
        uint256 id;
        string description;
        address[] targets;
        uint256[] values;
        bytes[] calldatas;
        uint256 startBlock;
        uint256 endBlock;
        uint256 forVotes;
        uint256 againstVotes;
        uint256 abstainVotes;
        bool executed;
        bool canceled;
        mapping(address => bool) hasVoted;
    }
    
    mapping(uint256 => Proposal) public proposals;
    uint256 public proposalCount;
    
    uint256 public constant VOTING_DELAY = 1;
    uint256 public constant VOTING_PERIOD = 100; // 约 20 分钟
    uint256 public constant QUORUM = 1000 * 10**18; // 1000 tokens
    uint256 public constant PROPOSAL_THRESHOLD = 100 * 10**18; // 100 tokens
    
    event ProposalCreated(
        uint256 indexed id,
        address proposer,
        string description,
        uint256 startBlock,
        uint256 endBlock
    );
    
    event Voted(
        uint256 indexed proposalId,
        address voter,
        uint8 support, // 0: Against, 1: For, 2: Abstain
        uint256 weight
    );
    
    event ProposalExecuted(uint256 indexed id);
    event ProposalCanceled(uint256 indexed id);
    
    constructor(address _governanceToken) {
        governanceToken = IERC20(_governanceToken);
    }
    
    function propose(
        address[] calldata targets,
        uint256[] calldata values,
        bytes[] calldata calldatas,
        string calldata description
    ) external returns (uint256) {
        require(
            governanceToken.balanceOf(msg.sender) >= PROPOSAL_THRESHOLD,
            "Below threshold"
        );
        
        proposalCount++;
        uint256 proposalId = proposalCount;
        
        Proposal storage proposal = proposals[proposalId];
        proposal.id = proposalId;
        proposal.description = description;
        proposal.targets = targets;
        proposal.values = values;
        proposal.calldatas = calldatas;
        proposal.startBlock = block.number + VOTING_DELAY;
        proposal.endBlock = block.number + VOTING_DELAY + VOTING_PERIOD;
        
        emit ProposalCreated(
            proposalId,
            msg.sender,
            description,
            proposal.startBlock,
            proposal.endBlock
        );
        
        return proposalId;
    }
    
    function castVote(
        uint256 proposalId,
        uint8 support
    ) external returns (uint256) {
        Proposal storage proposal = proposals[proposalId];
        
        require(block.number >= proposal.startBlock, "Not started");
        require(block.number <= proposal.endBlock, "Ended");
        require(!proposal.hasVoted[msg.sender], "Already voted");
        require(support <= 2, "Invalid support");
        
        uint256 weight = governanceToken.balanceOf(msg.sender);
        require(weight > 0, "No voting power");
        
        proposal.hasVoted[msg.sender] = true;
        
        if (support == 0) {
            proposal.againstVotes += weight;
        } else if (support == 1) {
            proposal.forVotes += weight;
        } else {
            proposal.abstainVotes += weight;
        }
        
        emit Voted(proposalId, msg.sender, support, weight);
        
        return weight;
    }
    
    function execute(uint256 proposalId) external {
        Proposal storage proposal = proposals[proposalId];
        
        require(block.number > proposal.endBlock, "Not ended");
        require(!proposal.executed, "Already executed");
        require(!proposal.canceled, "Canceled");
        
        // 检查是否通过
        require(
            proposal.forVotes > proposal.againstVotes,
            "Not passed"
        );
        
        // 检查法定人数
        require(
            proposal.forVotes + proposal.abstainVotes >= QUORUM,
            "Quorum not reached"
        );
        
        proposal.executed = true;
        
        // 执行操作
        for (uint256 i = 0; i < proposal.targets.length; i++) {
            (bool success, ) = proposal.targets[i].call{
                value: proposal.values[i]
            }(proposal.calldatas[i]);
            
            require(success, "Execution failed");
        }
        
        emit ProposalExecuted(proposalId);
    }
    
    function cancel(uint256 proposalId) external {
        Proposal storage proposal = proposals[proposalId];
        
        require(!proposal.executed, "Already executed");
        
        proposal.canceled = true;
        
        emit ProposalCanceled(proposalId);
    }
    
    function getProposal(uint256 proposalId) external view returns (
        uint256 id,
        string memory description,
        uint256 startBlock,
        uint256 endBlock,
        uint256 forVotes,
        uint256 againstVotes,
        uint256 abstainVotes,
        bool executed,
        bool canceled
    ) {
        Proposal storage proposal = proposals[proposalId];
        return (
            proposal.id,
            proposal.description,
            proposal.startBlock,
            proposal.endBlock,
            proposal.forVotes,
            proposal.againstVotes,
            proposal.abstainVotes,
            proposal.executed,
            proposal.canceled
        );
    }
    
    function state(uint256 proposalId) external view returns (uint8) {
        Proposal storage proposal = proposals[proposalId];
        
        if (proposal.executed) return 3; // Executed
        if (proposal.canceled) return 2; // Canceled
        if (block.number <= proposal.endBlock) return 1; // Active
        if (proposal.forVotes <= proposal.againstVotes) return 4; // Defeated
        return 5; // Succeeded
    }
}
```

## 前端组件

```tsx
import { useAccount, useReadContract, useWriteContract } from 'wagmi';
import { useState } from 'react';

export function DAODashboard() {
  const { address } = useAccount();
  const { writeContract } = useWriteContract();
  
  const [description, setDescription] = useState('');
  const [targets, setTargets] = useState('');
  const [values, setValues] = useState('');
  const [calldatas, setCalldatas] = useState('');
  
  const { data: proposalCount } = useReadContract({
    address: DAO_ADDRESS,
    abi: DAO_ABI,
    functionName: 'proposalCount',
  });
  
  const handleCreateProposal = () => {
    const targetArray = targets.split(',').map(t => t.trim());
    const valueArray = values.split(',').map(v => BigInt(v.trim()));
    const calldataArray = calldatas.split(',').map(c => c.trim());
    
    writeContract({
      address: DAO_ADDRESS,
      abi: DAO_ABI,
      functionName: 'propose',
      args: [targetArray, valueArray, calldataArray, description],
    });
  };
  
  const handleVote = (proposalId: bigint, support: number) => {
    writeContract({
      address: DAO_ADDRESS,
      abi: DAO_ABI,
      functionName: 'castVote',
      args: [proposalId, support],
    });
  };
  
  const handleExecute = (proposalId: bigint) => {
    writeContract({
      address: DAO_ADDRESS,
      abi: DAO_ABI,
      functionName: 'execute',
      args: [proposalId],
    });
  };
  
  return (
    <div className="dao-dashboard">
      <h1>DAO 治理</h1>
      
      <section className="create-proposal">
        <h2>创建提案</h2>
        <textarea
          placeholder="提案描述"
          value={description}
          onChange={(e) => setDescription(e.target.value)}
        />
        <input
          placeholder="目标地址（逗号分隔）"
          value={targets}
          onChange={(e) => setTargets(e.target.value)}
        />
        <input
          placeholder="ETH 数量（逗号分隔）"
          value={values}
          onChange={(e) => setValues(e.target.value)}
        />
        <input
          placeholder="调用数据（逗号分隔）"
          value={calldatas}
          onChange={(e) => setCalldatas(e.target.value)}
        />
        <button onClick={handleCreateProposal}>创建提案</button>
      </section>
      
      {/* 提案列表组件... */}
    </div>
  );
}
```

## 下一步

[进阶项目建议 →](05-advance.md)
