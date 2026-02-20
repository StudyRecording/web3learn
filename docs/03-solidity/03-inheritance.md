# 继承与组合

> 掌握 Solidity 的继承机制和代码复用

## 继承基础

Solidity 支持**多继承**，使用 `is` 关键字。

```
┌─────────────────────────────────────────────────────────────┐
│                   继承 vs Java                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Solidity                   Java                            │
│  ─────────                  ─────────                       │
│  单继承                     单继承                           │
│  多继承                     不支持 (需接口)                  │
│  contract A is B, C         class A extends B impl C        │
│  super                      super                           │
│  abstract contract          abstract class                  │
│  interface                  interface                       │
│                                                             │
│  关键差异:                                                  │
│  1. Solidity 直接支持多继承                                 │
│  2. 使用 is 关键字                                          │
│  3. 函数重写需要 override 关键字                            │
│  4. 父类叫"基类"不是"父类"                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 单继承

```solidity
// 基类
contract Animal {
    string public name;
    
    constructor(string memory _name) {
        name = _name;
    }
    
    function speak() public virtual returns (string memory) {
        return "Some sound";
    }
}

// 派生类
contract Dog is Animal {
    constructor(string memory _name) Animal(_name) {
        // 调用基类构造函数
    }
    
    function speak() public override returns (string memory) {
        return "Woof";
    }
}
```

### Java 对比

```solidity
// Solidity
contract Dog is Animal {
    function speak() public override returns (string memory) {
        return "Woof";
    }
}
```

```java
// Java
public class Dog extends Animal {
    @Override
    public String speak() {
        return "Woof";
    }
}
```

## 多继承

```solidity
contract A {
    function foo() public virtual pure returns (string memory) {
        return "A";
    }
}

contract B {
    function foo() public virtual pure returns (string memory) {
        return "B";
    }
}

contract C is A, B {
    function foo() public override(A, B) pure returns (string memory) {
        return super.foo();  // 调用 B.foo() (最后继承的)
    }
}
```

### 继承顺序

```
┌─────────────────────────────────────────────────────────────┐
│                    多继承顺序规则                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  contract C is A, B { }                                     │
│                   │  │                                      │
│                   │  └── 最后继承，优先级最高               │
│                   └── 先继承，优先级较低                    │
│                                                             │
│  方法查找顺序 (C3 线性化):                                  │
│  C → B → A                                                  │
│                                                             │
│  super.foo() 调用:                                          │
│  在 C 中调用 super.foo() 会调用 B.foo()                     │
│                                                             │
│  Java 对比:                                                 │
│  Java 不允许多继承，需要用接口模拟                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 钻石问题

```solidity
//     A
//    / \
//   B   C
//    \ /
//     D

contract A {
    function foo() public virtual pure returns (string memory) {
        return "A";
    }
}

contract B is A {
    function foo() public virtual override pure returns (string memory) {
        return "B";
    }
}

contract C is A {
    function foo() public virtual override pure returns (string memory) {
        return "C";
    }
}

contract D is B, C {
    function foo() public override(B, C) pure returns (string memory) {
        // super.foo() 会按顺序调用 C.foo() 和 B.foo()
        return super.foo();  // 返回 "C"
    }
}
```

## 函数重写

### virtual 和 override

```solidity
contract Base {
    // 可被重写
    function foo() public virtual pure returns (string memory) {
        return "Base";
    }
    
    // 不可被重写
    function bar() public pure returns (string memory) {
        return "Base";
    }
}

contract Derived is Base {
    function foo() public override pure returns (string memory) {
        return "Derived";
    }
    
    // 编译错误: bar() 不可重写
    // function bar() public pure returns (string memory) {
    //     return "Derived";
    // }
}
```

### 多重继承重写

```solidity
contract A {
    function foo() public virtual pure returns (string memory) {
        return "A";
    }
}

contract B is A {
    function foo() public virtual override pure returns (string memory) {
        return "B";
    }
}

contract C is A {
    function foo() public virtual override pure returns (string memory) {
        return "C";
    }
}

contract D is B, C {
    // 必须指定重写哪些基类
    function foo() public override(B, C) pure returns (string memory) {
        return "D";
    }
}
```

## super 关键字

```solidity
contract Level1 {
    event Log(string message);
    
    function process() public virtual {
        emit Log("Level1.process");
    }
}

contract Level2 is Level1 {
    function process() public override {
        emit Log("Level2.process start");
        super.process();  // 调用 Level1.process
        emit Log("Level2.process end");
    }
}

contract Level3 is Level2 {
    function process() public override {
        emit Log("Level3.process start");
        super.process();  // 调用 Level2.process
        emit Log("Level3.process end");
    }
}
```

## 抽象合约

```solidity
abstract contract Animal {
    string public name;
    
    constructor(string memory _name) {
        name = _name;
    }
    
    // 抽象函数 - 没有实现
    function speak() public virtual returns (string memory);
    
    // 具体函数 - 有实现
    function sleep() public pure returns (string memory) {
        return "Zzz...";
    }
}

contract Cat is Animal {
    constructor(string memory _name) Animal(_name) {}
    
    function speak() public override returns (string memory) {
        return "Meow";
    }
}
```

### Java 对比

```solidity
// Solidity
abstract contract Animal {
    function speak() public virtual returns (string memory);
}
```

```java
// Java
public abstract class Animal {
    public abstract String speak();
}
```

## 接口

### 接口规则

```
┌─────────────────────────────────────────────────────────────┐
│                    接口限制                                  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  可以包含:                                                  │
│  ├── 函数声明 (必须)                                        │
│  └── 事件声明                                               │
│                                                             │
│  不能包含:                                                  │
│  ├── 构造函数                                               │
│  ├── 状态变量                                               │
│  ├── 函数实现                                               │
│  ├── 修饰器                                                 │
│  └── 非 external 函数                                       │
│                                                             │
│  所有函数默认:                                              │
│  ├── external                                               │
│  └── virtual                                                │
│                                                             │
│  Java 对比:                                                 │
│  interface ≈ Java interface (更严格)                       │
│  不能有 default 方法实现                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 接口定义

```solidity
interface IERC20 {
    function totalSupply() external view returns (uint256);
    function balanceOf(address account) external view returns (uint256);
    function transfer(address to, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transferFrom(address from, address to, uint256 amount) external returns (bool);
    
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner, address indexed spender, uint256 value);
}

contract MyToken is IERC20 {
    mapping(address => uint256) private _balances;
    mapping(address => mapping(address => uint256)) private _allowances;
    uint256 private _totalSupply;
    
    function totalSupply() external view override returns (uint256) {
        return _totalSupply;
    }
    
    function balanceOf(address account) external view override returns (uint256) {
        return _balances[account];
    }
    
    function transfer(address to, uint256 amount) external override returns (bool) {
        _transfer(msg.sender, to, amount);
        return true;
    }
    
    function allowance(address owner, address spender) external view override returns (uint256) {
        return _allowances[owner][spender];
    }
    
    function approve(address spender, uint256 amount) external override returns (bool) {
        _approve(msg.sender, spender, amount);
        return true;
    }
    
    function transferFrom(address from, address to, uint256 amount) external override returns (bool) {
        uint256 currentAllowance = _allowances[from][msg.sender];
        require(currentAllowance >= amount, "ERC20: transfer amount exceeds allowance");
        _approve(from, msg.sender, currentAllowance - amount);
        _transfer(from, to, amount);
        return true;
    }
    
    function _transfer(address from, address to, uint256 amount) internal {
        require(from != address(0), "ERC20: transfer from zero address");
        require(to != address(0), "ERC20: transfer to zero address");
        
        uint256 fromBalance = _balances[from];
        require(fromBalance >= amount, "ERC20: transfer amount exceeds balance");
        _balances[from] = fromBalance - amount;
        _balances[to] += amount;
        
        emit Transfer(from, to, amount);
    }
    
    function _approve(address owner, address spender, uint256 amount) internal {
        require(owner != address(0), "ERC20: approve from zero address");
        require(spender != address(0), "ERC20: approve to zero address");
        
        _allowances[owner][spender] = amount;
        emit Approval(owner, spender, amount);
    }
    
    function _mint(address account, uint256 amount) internal {
        require(account != address(0), "ERC20: mint to zero address");
        
        _totalSupply += amount;
        _balances[account] += amount;
        emit Transfer(address(0), account, amount);
    }
}
```

## 组合模式

### 策略模式

```solidity
interface IPricingStrategy {
    function calculatePrice(uint256 basePrice) external view returns (uint256);
}

contract StandardPricing is IPricingStrategy {
    function calculatePrice(uint256 basePrice) external pure override returns (uint256) {
        return basePrice;
    }
}

contract DiscountPricing is IPricingStrategy {
    uint256 public discountPercent = 10;  // 10%
    
    function calculatePrice(uint256 basePrice) external view override returns (uint256) {
        return basePrice * (100 - discountPercent) / 100;
    }
}

contract Shop {
    IPricingStrategy public pricingStrategy;
    
    constructor(address _strategy) {
        pricingStrategy = IPricingStrategy(_strategy);
    }
    
    function setStrategy(address _strategy) public {
        pricingStrategy = IPricingStrategy(_strategy);
    }
    
    function buy(uint256 basePrice) public view returns (uint256) {
        return pricingStrategy.calculatePrice(basePrice);
    }
}
```

### 依赖注入

```solidity
interface IOracle {
    function getPrice() external view returns (uint256);
}

contract ChainlinkOracle is IOracle {
    function getPrice() external pure override returns (uint256) {
        return 2000 * 10**8;  // 模拟价格
    }
}

contract PriceConsumer {
    IOracle public oracle;
    address public owner;
    
    constructor(address _oracle) {
        oracle = IOracle(_oracle);
        owner = msg.sender;
    }
    
    function setOracle(address _oracle) public {
        require(msg.sender == owner, "Not owner");
        oracle = IOracle(_oracle);
    }
    
    function getValueInUSD(uint256 amount) public view returns (uint256) {
        uint256 price = oracle.getPrice();
        return amount * price / 10**8;
    }
}
```

## 实用继承模式

### Ownable 模式

```solidity
abstract contract Ownable {
    address public owner;
    
    event OwnershipTransferred(address indexed previousOwner, address indexed newOwner);
    
    constructor() {
        owner = msg.sender;
        emit OwnershipTransferred(address(0), msg.sender);
    }
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Not owner");
        _;
    }
    
    function transferOwnership(address newOwner) public onlyOwner {
        require(newOwner != address(0), "Zero address");
        emit OwnershipTransferred(owner, newOwner);
        owner = newOwner;
    }
    
    function renounceOwnership() public onlyOwner {
        emit OwnershipTransferred(owner, address(0));
        owner = address(0);
    }
}

contract MyContract is Ownable {
    uint256 public value;
    
    function setValue(uint256 _value) public onlyOwner {
        value = _value;
    }
}
```

### Pausable 模式

```solidity
abstract contract Pausable {
    bool public paused;
    address public pauser;
    
    event Paused(address account);
    event Unpaused(address account);
    
    constructor() {
        pauser = msg.sender;
    }
    
    modifier whenNotPaused() {
        require(!paused, "Paused");
        _;
    }
    
    modifier whenPaused() {
        require(paused, "Not paused");
        _;
    }
    
    modifier onlyPauser() {
        require(msg.sender == pauser, "Not pauser");
        _;
    }
    
    function pause() public onlyPauser whenNotPaused {
        paused = true;
        emit Paused(msg.sender);
    }
    
    function unpause() public onlyPauser whenPaused {
        paused = false;
        emit Unpaused(msg.sender);
    }
}

contract Bank is Pausable {
    mapping(address => uint256) public balances;
    
    function deposit() public payable whenNotPaused {
        balances[msg.sender] += msg.value;
    }
    
    function withdraw(uint256 amount) public whenNotPaused {
        require(balances[msg.sender] >= amount, "Insufficient");
        balances[msg.sender] -= amount;
        payable(msg.sender).transfer(amount);
    }
}
```

### 可升级模式 (代理)

```solidity
contract ImplementationV1 {
    uint256 public value;
    
    function setValue(uint256 _value) public {
        value = _value;
    }
}

contract ImplementationV2 {
    uint256 public value;
    
    function setValue(uint256 _value) public {
        value = _value * 2;  // 新逻辑
    }
    
    function doubleValue() public {
        value *= 2;
    }
}

// 代理合约 (简化版)
contract Proxy {
    address public implementation;
    address public admin;
    
    constructor(address _implementation) {
        implementation = _implementation;
        admin = msg.sender;
    }
    
    function upgrade(address _implementation) public {
        require(msg.sender == admin, "Not admin");
        implementation = _implementation;
    }
    
    fallback() external payable {
        address impl = implementation;
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
}
```

## 继承图示

```
┌─────────────────────────────────────────────────────────────┐
│                  实际项目继承结构                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│                      ┌──────────┐                           │
│                      │ Context  │ (地址检查)                │
│                      └────┬─────┘                           │
│                           │                                 │
│              ┌────────────┼────────────┐                    │
│              │            │            │                    │
│        ┌─────▼─────┐ ┌────▼────┐ ┌─────▼─────┐             │
│        │  Ownable  │ │ Pausable│ │  ERC20    │             │
│        └─────┬─────┘ └────┬────┘ └───────────┘             │
│              │            │                                 │
│              └─────┬──────┘                                 │
│                    │                                        │
│              ┌─────▼─────────┐                              │
│              │  MyContract   │                              │
│              └───────────────┘                              │
│                                                             │
│  contract MyContract is Ownable, Pausable, ERC20 { }        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 最佳实践

```solidity
// 好的继承设计
contract TokenBase is Ownable, Pausable {
    string public name;
    string public symbol;
}

contract MyToken is TokenBase {
    constructor() {
        name = "My Token";
        symbol = "MTK";
    }
}

// 避免过深继承
// 坏: A → B → C → D → E → F
// 好: A → B, C, D (扁平化)

// 使用接口定义交互边界
interface IExchange {
    function trade(address token, uint256 amount) external;
}

contract DEX is IExchange {
    function trade(address token, uint256 amount) external override {
        // 实现
    }
}
```

[下一节：高级特性 →](04-advanced.md)
