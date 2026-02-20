# 1.5 Web2 vs Web3 开发对比

## 核心差异

```
┌─────────────────────────────────────────────────────────────────┐
│                   Web2 vs Web3 开发核心差异                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Web2 开发                         Web3 开发                    │
│  ┌─────────────────┐              ┌─────────────────┐          │
│  │ 可修改          │              │ 不可变          │          │
│  │ ├── 热修复       │              │ ├── 代理模式     │          │
│  │ ├── 回滚        │              │ ├── 升级合约     │          │
│  │ └── 删除数据    │              │ └── 数据永久     │          │
│  └─────────────────┘              └─────────────────┘          │
│                                                                 │
│  ┌─────────────────┐              ┌─────────────────┐          │
│  │ 性能优化        │              │ Gas 优化        │          │
│  │ ├── 响应时间     │              │ ├── 存储优化     │          │
│  │ ├── 并发处理    │              │ ├── 计算优化     │          │
│  │ └── 资源管理    │              │ └── 真金白银     │          │
│  └─────────────────┘              └─────────────────┘          │
│                                                                 │
│  ┌─────────────────┐              ┌─────────────────┐          │
│  │ 私有数据        │              │ 公开数据        │          │
│  │ ├── 数据库加密   │              │ ├── 所有人可见   │          │
│  │ ├── 访问控制    │              │ ├── 透明审计     │          │
│  │ └── 中心化存储  │              │ └── 链上存储     │          │
│  └─────────────────┘              └─────────────────┘          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 架构对比

### 用户认证

```java
// Web2: 用户认证 (Java Spring)

@RestController
public class AuthController {
    
    @PostMapping("/login")
    public String login(@RequestBody LoginRequest request) {
        // 验证用户名密码
        User user = userService.authenticate(
            request.getUsername(), 
            request.getPassword()
        );
        
        // 生成 JWT
        String token = jwtService.generateToken(user);
        
        // 存储会话
        sessionService.createSession(user.getId(), token);
        
        return token;
    }
}

// 问题:
// 1. 需要管理用户密码
// 2. 需要处理密码重置
// 3. 数据库可能泄露
// 4. 平台可封禁用户
```

```solidity
// Web3: 用户认证 (智能合约)

contract AuthContract {
    // 无需存储用户信息
    // msg.sender 就是用户身份
    
    mapping(address => uint256) public balances;
    
    function deposit() public payable {
        // msg.sender 自动验证
        // 无需登录，无需密码
        balances[msg.sender] += msg.value;
    }
    
    function withdraw(uint256 amount) public {
        require(balances[msg.sender] >= amount, "Insufficient");
        
        // 调用者身份已验证
        balances[msg.sender] -= amount;
        payable(msg.sender).transfer(amount);
    }
}

// 优势:
// 1. 无需管理密码
// 2. 钱包签名验证
// 3. 无隐私数据泄露
// 4. 无人能封禁
```

```javascript
// Web3 前端: 连接钱包

import { useAccount, useSignMessage } from 'wagmi';

function LoginComponent() {
  const { address, isConnected } = useAccount();
  const { signMessageAsync } = useSignMessage();
  
  const handleLogin = async () => {
    // 生成随机消息
    const nonce = generateNonce();
    const message = `Sign this message to login: ${nonce}`;
    
    // 用户钱包签名
    const signature = await signMessageAsync({ message });
    
    // 后端验证签名
    const verified = verifySignature(address, message, signature);
    
    if (verified) {
      // 登录成功
      console.log('Logged in as', address);
    }
  };
  
  return (
    <button onClick={handleLogin}>
      {isConnected ? 'Login with Wallet' : 'Connect Wallet First'}
    </button>
  );
}
```

### 数据存储

```java
// Web2: 数据库存储

@Service
public class UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    public User createUser(String username, String email) {
        User user = new User();
        user.setUsername(username);
        user.setEmail(email);
        user.setCreatedAt(LocalDateTime.now());
        
        // 存储在中心化数据库
        return userRepository.save(user);
    }
    
    public void updateUser(Long id, String newEmail) {
        // 可以随意修改
        User user = userRepository.findById(id);
        user.setEmail(newEmail);
        userRepository.save(user);
    }
    
    public void deleteUser(Long id) {
        // 可以删除数据
        userRepository.deleteById(id);
    }
}
```

```solidity
// Web3: 链上存储

contract UserRegistry {
    struct User {
        string username;      // 注意: 链上数据完全公开
        address wallet;
        uint256 createdAt;
    }
    
    mapping(address => User) public users;
    address[] public userList;
    
    event UserCreated(address indexed wallet, string username);
    
    // 创建用户 - 需要支付 Gas
    function createUser(string memory _username) public {
        require(bytes(users[msg.sender].username).length == 0, "Already exists");
        
        users[msg.sender] = User({
            username: _username,
            wallet: msg.sender,
            createdAt: block.timestamp
        });
        
        userList.push(msg.sender);
        
        emit UserCreated(msg.sender, _username);
    }
    
    // 更新用户 - 数据仍然存在，只是标记
    // 实际上无法真正"删除"
    function updateUsername(string memory _newUsername) public {
        require(bytes(users[msg.sender].username).length > 0, "User not exists");
        users[msg.sender].username = _newUsername;
    }
    
    // 注意: 无法真正删除用户
    // 所有历史数据永久存在
}
```

### API 设计

```java
// Web2: REST API

@RestController
@RequestMapping("/api/nft")
public class NFTController {
    
    @GetMapping("/{id}")
    public NFT getNFT(@PathVariable Long id) {
        // 查询数据库
        return nftService.findById(id);
    }
    
    @PostMapping
    public NFT createNFT(@RequestBody NFTRequest request) {
        // 创建记录
        return nftService.create(request);
    }
    
    @PutMapping("/{id}")
    public NFT updateNFT(@PathVariable Long id, @RequestBody NFTRequest request) {
        // 更新记录
        return nftService.update(id, request);
    }
    
    @DeleteMapping("/{id}")
    public void deleteNFT(@PathVariable Long id) {
        // 删除记录
        nftService.delete(id);
    }
}

// 特点:
// - 中心化控制
// - 可以修改/删除
// - 响应快速
// - 可能宕机
```

```solidity
// Web3: 智能合约接口

contract SimpleNFT {
    // 状态变量（存储）
    mapping(uint256 => address) public ownerOf;
    mapping(uint256 => string) public tokenURI;
    mapping(address => uint256) public balanceOf;
    
    uint256 public totalSupply;
    
    // 事件（用于索引）
    event Transfer(address indexed from, address indexed to, uint256 tokenId);
    
    // "创建" - 铸造
    function mint(string memory _uri) public returns (uint256) {
        uint256 tokenId = totalSupply;
        
        ownerOf[tokenId] = msg.sender;
        tokenURI[tokenId] = _uri;
        balanceOf[msg.sender] += 1;
        totalSupply += 1;
        
        emit Transfer(address(0), msg.sender, tokenId);
        
        return tokenId;
    }
    
    // "更新" - 转移所有权
    function transfer(address to, uint256 tokenId) public {
        require(ownerOf[tokenId] == msg.sender, "Not owner");
        
        ownerOf[tokenId] = to;
        balanceOf[msg.sender] -= 1;
        balanceOf[to] += 1;
        
        emit Transfer(msg.sender, to, tokenId);
    }
    
    // 注意: 没有"删除"
    // NFT 一旦铸造，永久存在
}

// 特点:
// - 去中心化
// - 不可篡改
// - 响应较慢（需区块确认）
// - 永不停机
```

## 开发思维转换

### 不可变性思维

```
Web2 思维:
┌─────────────────────────────────────────┐
│                                         │
│  "先上线，有问题再修"                    │
│                                         │
│  ├── 热修复部署                         │
│  ├── 数据库迁移                         │
│  └── 回滚版本                           │
│                                         │
└─────────────────────────────────────────┘


Web3 思维:
┌─────────────────────────────────────────┐
│                                         │
│  "部署即永久，安全第一"                  │
│                                         │
│  ├── 充分测试                           │
│  ├── 代码审计                           │
│  ├── 代理模式（可升级）                  │
│  └── 紧急暂停机制                       │
│                                         │
└─────────────────────────────────────────┘
```

### Gas 优化思维

```solidity
// Gas 优化示例

// ❌ 高 Gas 消耗
contract Expensive {
    uint256[] public numbers;
    
    function sum() public view returns (uint256) {
        uint256 total = 0;
        // 动态数组长度每次读取都消耗 Gas
        for (uint256 i = 0; i < numbers.length; i++) {
            total += numbers[i];
        }
        return total;
    }
}

// ✓ Gas 优化
contract Optimized {
    uint256[] public numbers;
    uint256 public totalSum;  // 缓存结果
    
    function addNumber(uint256 _num) public {
        numbers.push(_num);
        totalSum += _num;  // 更新时计算
    }
    
    function sum() public view returns (uint256) {
        return totalSum;  // 直接返回
    }
}

// Gas 节省技巧:
// 1. 使用 uint256（EVM 原生支持）
// 2. 避免循环中读取存储
// 3. 使用 events 存储历史数据
// 4. 合并存储变量
// 5. 使用 calldata 代替 memory
```

### 安全思维

```solidity
// 常见安全漏洞示例

// ❌ 重入攻击漏洞
contract Vulnerable {
    mapping(address => uint256) public balances;
    
    function withdraw() public {
        uint256 amount = balances[msg.sender];
        
        // 先转账，后更新状态 = 危险！
        payable(msg.sender).call{value: amount}("");
        
        balances[msg.sender] = 0;
    }
    
    // 攻击者可以在 call 中递归调用 withdraw
    // 在状态更新前多次提取资金
}

// ✓ 防止重入攻击
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract Secure is ReentrancyGuard {
    mapping(address => uint256) public balances;
    
    function withdraw() public nonReentrant {
        uint256 amount = balances[msg.sender];
        require(amount > 0, "No balance");
        
        // 先更新状态，后转账
        balances[msg.sender] = 0;
        
        (bool success, ) = payable(msg.sender).call{value: amount}("");
        require(success, "Transfer failed");
    }
}
```

## 成本对比

```
┌─────────────────────────────────────────────────────────────────┐
│                     开发和运维成本对比                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Web2 成本                       Web3 成本                      │
│  ├── 服务器费用                   ├── 部署 Gas 费用              │
│  ├── 数据库费用                   ├── 交易 Gas 费用              │
│  ├── CDN 费用                    ├── 节点服务费用                │
│  ├── 运维人员                    └── 审计费用                    │
│  └── 带宽费用                                                   │
│                                                                 │
│  特点:                          特点:                           │
│  - 持续运营成本                  - 部署一次性成本                │
│  - 随用户增长增加                - 交易频繁则成本高              │
│  - 可预测                        - Gas 价格波动                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## 开发流程对比

```
Web2 开发流程:
┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐
│ 开发   │──>│ 测试   │──>│ 部署   │──>│ 运维   │
└────────┘   └────────┘   └────────┘   └────────┘
                 │              │
                 └──────────────┘
                  可快速迭代


Web3 开发流程:
┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐
│ 开发   │──>│ 测试   │──>│ 审计   │──>│ 部署   │──>│ 监控   │
└────────┘   └────────┘   └────────┘   └────────┘   └────────┘
                 │              │
                 │    ┌─────────┴─────────┐
                 │    │  安全审计很重要    │
                 │    │  代码公开透明      │
                 │    │  Bug可能导致资金损失│
                 │    └───────────────────┘
                 │
                 └── 必须充分测试
```

## 总结: 关键思维转变

| 方面 | Web2 思维 | Web3 思维 |
|------|----------|----------|
| 修改代码 | 随时更新 | 不可变，需代理模式 |
| 数据存储 | 中心化数据库 | 链上公开 + IPFS |
| 用户认证 | 用户名密码 | 钱包签名 |
| 性能优化 | 响应时间 | Gas 成本 |
| 安全 | 保护服务器 | 保护资金 |
| 测试 | 功能测试 | 安全测试 + 审计 |
| 部署 | CI/CD | 一次部署永久运行 |
| 运维 | 持续运维 | 几乎无需运维 |

## 思考题

1. 为什么 Web3 合约部署后难以修改？这是优势还是劣势？
2. 如果你是一个 Web2 开发者，最难适应 Web3 的哪个方面？
3. 什么样的应用适合 Web3？什么样的应用不适合？

## 下一步

恭喜完成第一章！

现在你已经了解了 Web3 的基础概念，接下来让我们深入学习以太坊：

[第二章：以太坊基础 →](../02-ethereum-fundamentals/README.md)
