# 代币标准

> 掌握以太坊主流代币标准 ERC-20、ERC-721、ERC-1155

## ERC-20 (同质化代币)

### 概述

```
┌─────────────────────────────────────────────────────────────┐
│                     ERC-20 标准                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  特点:                                                      │
│  ├── 同质化: 每个代币等价                                   │
│  ├── 可分割: 支持小数                                       │
│  └── 可互换: 1 个代币 = 1 个代币                            │
│                                                             │
│  用途:                                                      │
│  ├── 治理代币                                               │
│  ├── 稳定币 (USDT, USDC)                                   │
│  ├── 平台代币 (UNI, AAVE)                                  │
│  └── 证券代币                                               │
│                                                             │
│  Java 对比:                                                 │
│  ERC-20 ≈ 货币系统                                         │
│  balanceOf ≈ 查询账户余额                                  │
│  transfer ≈ 转账                                           │
│  approve + transferFrom ≈ 授权扣款                         │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 接口定义

```solidity
interface IERC20 {
    // 总供应量
    function totalSupply() external view returns (uint256);
    
    // 查询余额
    function balanceOf(address account) external view returns (uint256);
    
    // 转账
    function transfer(address to, uint256 amount) external returns (bool);
    
    // 查询授权额度
    function allowance(address owner, address spender) external view returns (uint256);
    
    // 授权
    function approve(address spender, uint256 amount) external returns (bool);
    
    // 授权转账
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
    
    // 事件
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}
```

### 完整实现

```solidity
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract MyToken is ERC20, Ownable {
    uint8 private constant _decimals = 18;
    
    constructor(
        string memory name,
        string memory symbol,
        uint256 initialSupply
    ) ERC20(name, symbol) {
        _mint(msg.sender, initialSupply * 10 ** decimals());
    }
    
    function decimals() public pure override returns (uint8) {
        return _decimals;
    }
    
    function mint(address to, uint256 amount) public onlyOwner {
        _mint(to, amount);
    }
    
    function burn(uint256 amount) public {
        _burn(msg.sender, amount);
    }
}
```

### 从零实现

```solidity
contract ERC20Basic {
    string public name;
    string public symbol;
    uint8 public decimals = 18;
    uint256 public totalSupply;
    
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;
    
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
    
    constructor(string memory _name, string memory _symbol, uint256 _initialSupply) {
        name = _name;
        symbol = _symbol;
        totalSupply = _initialSupply * 10 ** uint256(decimals);
        balanceOf[msg.sender] = totalSupply;
    }
    
    function transfer(address _to, uint256 _value) public returns (bool success) {
        require(_to != address(0), "Invalid address");
        require(balanceOf[msg.sender] >= _value, "Insufficient balance");
        
        balanceOf[msg.sender] -= _value;
        balanceOf[_to] += _value;
        
        emit Transfer(msg.sender, _to, _value);
        return true;
    }
    
    function approve(address _spender, uint256 _value) public returns (bool success) {
        allowance[msg.sender][_spender] = _value;
        emit Approval(msg.sender, _spender, _value);
        return true;
    }
    
    function transferFrom(address _from, address _to, uint256 _value) 
        public returns (bool success) 
    {
        require(_to != address(0), "Invalid address");
        require(balanceOf[_from] >= _value, "Insufficient balance");
        require(allowance[_from][msg.sender] >= _value, "Allowance exceeded");
        
        balanceOf[_from] -= _value;
        balanceOf[_to] += _value;
        allowance[_from][msg.sender] -= _value;
        
        emit Transfer(_from, _to, _value);
        return true;
    }
}
```

### 授权流程

```
┌─────────────────────────────────────────────────────────────┐
│                   ERC-20 授权流程                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  用户 (Owner)                合约 (Spender)                 │
│       │                           │                         │
│       │ 1. approve(1000)          │                         │
│       │ ──────────────────────►   │                         │
│       │                           │ allowance[owner][spender]│
│       │                           │ = 1000                   │
│       │                           │                         │
│       │ 2. 使用合约功能            │                         │
│       │ ──────────────────────►   │                         │
│       │                           │                         │
│       │                           │ 3. transferFrom()       │
│       │                           │    检查 allowance       │
│       │                           │    扣减 allowance       │
│       │                           │    转移代币             │
│       │                           │                         │
│  常见场景:                                                  │
│  ├── DEX 交易: 授权 DEX 合约                                │
│  ├── 借贷: 授权借贷协议                                     │
│  └── 质押: 授权质押合约                                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## ERC-721 (非同质化代币 NFT)

### 概述

```
┌─────────────────────────────────────────────────────────────┐
│                     ERC-721 标准                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  特点:                                                      │
│  ├── 非同质化: 每个 NFT 独一无二                            │
│  ├── 不可分割: 只能整数转移                                 │
│  └── 唯一标识: tokenId 标识每个 NFT                         │
│                                                             │
│  用途:                                                      │
│  ├── 数字艺术                                               │
│  ├── 游戏道具                                               │
│  ├── 收藏品                                                 │
│  └── 数字身份                                               │
│                                                             │
│  Java 对比:                                                 │
│  ERC-721 ≈ 唯一资产系统                                    │
│  tokenId ≈ 资产ID                                          │
│  ownerOf ≈ 查询资产所有者                                  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 接口定义

```solidity
interface IERC721 {
    // 查询余额
    function balanceOf(address owner) external view returns (uint256);
    
    // 查询所有者
    function ownerOf(uint256 tokenId) external view returns (address);
    
    // 安全转账
    function safeTransferFrom(address from, address to, uint256 tokenId) external;
    
    // 安全转账(带数据)
    function safeTransferFrom(address from, address to, uint256 tokenId, bytes calldata data) external;
    
    // 普通转账
    function transferFrom(address from, address to, uint256 tokenId) external;
    
    // 授权
    function approve(address to, uint256 tokenId) external;
    
    // 查询被授权者
    function getApproved(uint256 tokenId) external view returns (address);
    
    // 全局授权
    function setApprovalForAll(address operator, bool approved) external;
    
    // 查询全局授权
    function isApprovedForAll(address owner, address operator) external view returns (bool);
    
    // 事件
    event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
    event Approval(address indexed owner, address indexed approved, uint256 indexed tokenId);
    event ApprovalForAll(address indexed owner, address indexed operator, bool approved);
}
```

### 完整实现

```solidity
import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Counters.sol";

contract MyNFT is ERC721, Ownable {
    using Counters for Counters.Counter;
    Counters.Counter private _tokenIdCounter;
    
    uint256 public constant MAX_SUPPLY = 10000;
    uint256 public constant MINT_PRICE = 0.05 ether;
    
    string private _baseTokenURI;
    
    event Minted(address indexed to, uint256 tokenId);
    
    constructor(string memory name, string memory symbol, string memory baseURI) 
        ERC721(name, symbol) 
    {
        _baseTokenURI = baseURI;
    }
    
    function _baseURI() internal view override returns (string memory) {
        return _baseTokenURI;
    }
    
    function mint() public payable {
        require(_tokenIdCounter.current() < MAX_SUPPLY, "Max supply reached");
        require(msg.value == MINT_PRICE, "Wrong price");
        
        uint256 tokenId = _tokenIdCounter.current();
        _tokenIdCounter.increment();
        
        _safeMint(msg.sender, tokenId);
        emit Minted(msg.sender, tokenId);
    }
    
    function safeMint(address to) public onlyOwner {
        uint256 tokenId = _tokenIdCounter.current();
        _tokenIdCounter.increment();
        _safeMint(to, tokenId);
    }
    
    function tokenURI(uint256 tokenId) public view override returns (string memory) {
        require(_exists(tokenId), "Nonexistent token");
        return string(abi.encodePacked(_baseURI(), Strings.toString(tokenId), ".json"));
    }
    
    function withdraw() public onlyOwner {
        uint256 balance = address(this).balance;
        (bool success, ) = payable(owner()).call{value: balance}("");
        require(success, "Transfer failed");
    }
}
```

### 从零实现

```solidity
contract ERC721Basic {
    string public name;
    string public symbol;
    
    mapping(uint256 => address) private _owners;
    mapping(address => uint256) private _balances;
    mapping(uint256 => address) private _tokenApprovals;
    mapping(address => mapping(address => bool)) private _operatorApprovals;
    
    event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
    event Approval(address indexed owner, address indexed approved, uint256 indexed tokenId);
    event ApprovalForAll(address indexed owner, address indexed operator, bool approved);
    
    constructor(string memory _name, string memory _symbol) {
        name = _name;
        symbol = _symbol;
    }
    
    function balanceOf(address owner) public view returns (uint256) {
        require(owner != address(0), "Zero address");
        return _balances[owner];
    }
    
    function ownerOf(uint256 tokenId) public view returns (address) {
        address owner = _owners[tokenId];
        require(owner != address(0), "Nonexistent token");
        return owner;
    }
    
    function transferFrom(address from, address to, uint256 tokenId) public {
        require(_isApprovedOrOwner(msg.sender, tokenId), "Not approved");
        require(from == ownerOf(tokenId), "Wrong from address");
        require(to != address(0), "Zero address");
        
        _balances[from] -= 1;
        _balances[to] += 1;
        _owners[tokenId] = to;
        
        delete _tokenApprovals[tokenId];
        
        emit Transfer(from, to, tokenId);
    }
    
    function safeTransferFrom(address from, address to, uint256 tokenId) public {
        transferFrom(from, to, tokenId);
        
        require(
            to.code.length == 0 || 
            IERC721Receiver(to).onERC721Received(msg.sender, from, tokenId, "") 
                == IERC721Receiver.onERC721Received.selector,
            "Non ERC721Receiver"
        );
    }
    
    function approve(address to, uint256 tokenId) public {
        address owner = ownerOf(tokenId);
        require(to != owner, "Approve to owner");
        require(msg.sender == owner || isApprovedForAll(owner, msg.sender), "Not approved");
        
        _tokenApprovals[tokenId] = to;
        emit Approval(owner, to, tokenId);
    }
    
    function getApproved(uint256 tokenId) public view returns (address) {
        require(_exists(tokenId), "Nonexistent token");
        return _tokenApprovals[tokenId];
    }
    
    function setApprovalForAll(address operator, bool approved) public {
        _operatorApprovals[msg.sender][operator] = approved;
        emit ApprovalForAll(msg.sender, operator, approved);
    }
    
    function isApprovedForAll(address owner, address operator) public view returns (bool) {
        return _operatorApprovals[owner][operator];
    }
    
    function _exists(uint256 tokenId) internal view returns (bool) {
        return _owners[tokenId] != address(0);
    }
    
    function _isApprovedOrOwner(address spender, uint256 tokenId) internal view returns (bool) {
        address owner = ownerOf(tokenId);
        return (
            spender == owner ||
            getApproved(tokenId) == spender ||
            isApprovedForAll(owner, spender)
        );
    }
    
    function _mint(address to, uint256 tokenId) internal {
        require(to != address(0), "Zero address");
        require(!_exists(tokenId), "Token exists");
        
        _balances[to] += 1;
        _owners[tokenId] = to;
        
        emit Transfer(address(0), to, tokenId);
    }
    
    function _burn(uint256 tokenId) internal {
        address owner = ownerOf(tokenId);
        
        delete _tokenApprovals[tokenId];
        _balances[owner] -= 1;
        delete _owners[tokenId];
        
        emit Transfer(owner, address(0), tokenId);
    }
}

interface IERC721Receiver {
    function onERC721Received(
        address operator,
        address from,
        uint256 tokenId,
        bytes calldata data
    ) external returns (bytes4);
}
```

## ERC-1155 (多代币标准)

### 概述

```
┌─────────────────────────────────────────────────────────────┐
│                     ERC-1155 标准                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  特点:                                                      │
│  ├── 单合约多代币: 一个合约管理多种代币                     │
│  ├── 混合类型: 支持同质化和非同质化                         │
│  ├── 批量操作: 一次转移多个代币                             │
│  └── 高效: 比 ERC-20/721 更省 Gas                           │
│                                                             │
│  用途:                                                      │
│  ├── 游戏: 金币(FT) + 装备(NFT)                            │
│  ├── 数字藏品: 不同版本/稀有度                              │
│  └── 票务: 不同类型/座位                                    │
│                                                             │
│  对比:                                                      │
│  ERC-20:   1 合约 = 1 种代币                                │
│  ERC-721:  1 合约 = 1 种 NFT 集合                           │
│  ERC-1155: 1 合约 = 多种代币 + 多种 NFT                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 接口定义

```solidity
interface IERC1155 {
    // 查询余额
    function balanceOf(address account, uint256 id) external view returns (uint256);
    
    // 批量查询余额
    function balanceOfBatch(address[] calldata accounts, uint256[] calldata ids) 
        external view returns (uint256[] memory);
    
    // 全局授权
    function setApprovalForAll(address operator, bool approved) external;
    
    // 查询全局授权
    function isApprovedForAll(address account, address operator) external view returns (bool);
    
    // 安全转账
    function safeTransferFrom(address from, address to, uint256 id, uint256 amount, bytes calldata data) external;
    
    // 批量安全转账
    function safeBatchTransferFrom(
        address from,
        address to,
        uint256[] calldata ids,
        uint256[] calldata amounts,
        bytes calldata data
    ) external;
    
    // 事件
    event TransferSingle(address indexed operator, address indexed from, address indexed to, uint256 id, uint256 value);
    event TransferBatch(address indexed operator, address indexed from, address indexed to, uint256[] ids, uint256[] values);
    event ApprovalForAll(address indexed account, address indexed operator, bool approved);
    event URI(string value, uint256 indexed id);
}
```

### 完整实现

```solidity
import "@openzeppelin/contracts/token/ERC1155/ERC1155.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract GameItems is ERC1155, Ownable {
    uint256 public constant GOLD = 0;
    uint256 public constant SWORD = 1;
    uint256 public constant SHIELD = 2;
    uint256 public constant CROWN = 3;
    
    mapping(uint256 => string) private _uris;
    
    constructor() ERC1155("") {
        _mint(msg.sender, GOLD, 1000000 * 10**18, "");  // FT
        _mint(msg.sender, SWORD, 100, "");              // Limited
        _mint(msg.sender, SHIELD, 50, "");              // Limited
        _mint(msg.sender, CROWN, 1, "");                // NFT
    }
    
    function uri(uint256 tokenId) public view override returns (string memory) {
        return _uris[tokenId];
    }
    
    function setURI(uint256 tokenId, string memory newURI) public onlyOwner {
        _uris[tokenId] = newURI;
        emit URI(newURI, tokenId);
    }
    
    function mint(address to, uint256 id, uint256 amount, bytes memory data) public onlyOwner {
        _mint(to, id, amount, data);
    }
    
    function mintBatch(address to, uint256[] memory ids, uint256[] memory amounts, bytes memory data) 
        public onlyOwner 
    {
        _mintBatch(to, ids, amounts, data);
    }
    
    function burn(address from, uint256 id, uint256 amount) public {
        require(from == msg.sender || isApprovedForAll(from, msg.sender), "Not approved");
        _burn(from, id, amount);
    }
    
    function burnBatch(address from, uint256[] memory ids, uint256[] memory amounts) public {
        require(from == msg.sender || isApprovedForAll(from, msg.sender), "Not approved");
        _burnBatch(from, ids, amounts);
    }
}
```

## 标准对比

```
┌─────────────────────────────────────────────────────────────┐
│                   代币标准对比                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│              ERC-20      ERC-721      ERC-1155              │
│  ─────────────────────────────────────────────────          │
│  类型         FT          NFT         混合                  │
│  唯一性       相同        唯一        ID 决定               │
│  可分割       是          否          ID 决定               │
│  批量操作     无          无          有                    │
│  Gas 效率     中          低          高                    │
│  复杂度       低          中          高                    │
│                                                             │
│  选择指南:                                                  │
│  ├── 简单代币 (治理、支付) → ERC-20                        │
│  ├── 唯一收藏品 → ERC-721                                   │
│  └── 游戏道具、复杂系统 → ERC-1155                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 实用扩展

### ERC-20 扩展

```solidity
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Burnable.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Pausable.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/ERC20Capped.sol";

// 可销毁、可暂停、有上限的代币
contract AdvancedToken is ERC20Burnable, ERC20Pausable, Ownable {
    constructor(
        string memory name,
        string memory symbol,
        uint256 cap
    ) ERC20(name, symbol) {
        _mint(msg.sender, cap);
    }
    
    function pause() public onlyOwner {
        _pause();
    }
    
    function unpause() public onlyOwner {
        _unpause();
    }
    
    function _beforeTokenTransfer(
        address from,
        address to,
        uint256 amount
    ) internal override whenNotPaused {
        super._beforeTokenTransfer(from, to, amount);
    }
}
```

### ERC-721 扩展

```solidity
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721Enumerable.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol";
import "@openzeppelin/contracts/token/ERC721/extensions/ERC721Royalty.sol";

// 可枚举、自定义URI、版税的NFT
contract AdvancedNFT is ERC721Enumerable, ERC721URIStorage, ERC721Royalty, Ownable {
    uint256 private _tokenIdCounter;
    
    constructor(string memory name, string memory symbol) 
        ERC721(name, symbol) 
    {}
    
    function safeMint(address to, string memory uri) public onlyOwner {
        uint256 tokenId = _tokenIdCounter++;
        _safeMint(to, tokenId);
        _setTokenURI(tokenId, uri);
    }
    
    function setDefaultRoyalty(address receiver, uint96 feeNumerator) public onlyOwner {
        _setDefaultRoyalty(receiver, feeNumerator);
    }
    
    function supportsInterface(bytes4 interfaceId) 
        public 
        view 
        override(ERC721Enumerable, ERC721URIStorage, ERC721Royalty) 
        returns (bool) 
    {
        return super.supportsInterface(interfaceId);
    }
    
    function tokenURI(uint256 tokenId) 
        public 
        view 
        override(ERC721, ERC721URIStorage) 
        returns (string memory) 
    {
        return super.tokenURI(tokenId);
    }
    
    function _burn(uint256 tokenId) 
        internal 
        override(ERC721, ERC721URIStorage, ERC721Royalty) 
    {
        super._burn(tokenId);
    }
}
```

## 常用 OpenZeppelin 合约

```solidity
// 导入常用合约
import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/token/ERC1155/ERC1155.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/access/AccessControl.sol";
import "@openzeppelin/contracts/security/Pausable.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

// 使用 AccessControl 实现角色权限
contract RoleBasedToken is ERC20, AccessControl {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    bytes32 public constant BURNER_ROLE = keccak256("BURNER_ROLE");
    
    constructor() ERC20("Role Token", "RT") {
        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(MINTER_ROLE, msg.sender);
        _grantRole(BURNER_ROLE, msg.sender);
    }
    
    function mint(address to, uint256 amount) public onlyRole(MINTER_ROLE) {
        _mint(to, amount);
    }
    
    function burn(address from, uint256 amount) public onlyRole(BURNER_ROLE) {
        _burn(from, amount);
    }
}
```

## 总结

| 标准 | 用途 | 特点 |
|------|------|------|
| ERC-20 | 同质化代币 | 可互换、可分割、最简单 |
| ERC-721 | NFT | 唯一性、不可分割 |
| ERC-1155 | 多代币 | 混合类型、批量操作、最高效 |

## 返回

[← 返回章节概览](README.md)
