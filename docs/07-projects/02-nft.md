# 7.2 项目二：NFT 市场

## 项目概述

创建一个完整的 NFT 市场，支持铸造、上架、购买功能。

```
┌─────────────────────────────────────────────────────────────┐
│                    NFT 市场架构                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  用户 ──铸造──> NFT 合约 ──上架──> 市场合约 ──购买──> 用户  │
│                          │                                  │
│                          ▼                                  │
│                     版税分配                                │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 智能合约

### NFT 合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Counters.sol";

contract MyNFT is ERC721, Ownable {
    using Counters for Counters.Counter;
    
    Counters.Counter private _tokenIdCounter;
    
    uint256 public mintPrice = 0.01 ether;
    uint256 public maxSupply = 10000;
    uint256 public royaltyPercentage = 500; // 5%
    
    string private _baseTokenURI;
    
    struct NFTMetadata {
        string name;
        string description;
        string imageURI;
    }
    
    mapping(uint256 => NFTMetadata) public nftMetadata;
    mapping(uint256 => address) public creators;
    
    event Minted(address indexed to, uint256 tokenId, string uri);
    
    constructor(
        string memory name,
        string memory symbol,
        string memory baseURI
    ) ERC721(name, symbol) Ownable(msg.sender) {
        _baseTokenURI = baseURI;
    }
    
    function mint(
        string memory name,
        string memory description,
        string memory imageURI
    ) public payable returns (uint256) {
        require(msg.value >= mintPrice, "Insufficient payment");
        require(_tokenIdCounter.current() < maxSupply, "Max supply reached");
        
        uint256 tokenId = _tokenIdCounter.current();
        _tokenIdCounter.increment();
        
        _safeMint(msg.sender, tokenId);
        
        nftMetadata[tokenId] = NFTMetadata({
            name: name,
            description: description,
            imageURI: imageURI
        });
        
        creators[tokenId] = msg.sender;
        
        emit Minted(msg.sender, tokenId, imageURI);
        
        return tokenId;
    }
    
    function tokenURI(uint256 tokenId) public view override returns (string memory) {
        require(_ownerOf(tokenId) != address(0), "Token does not exist");
        
        NFTMetadata memory meta = nftMetadata[tokenId];
        
        return string(abi.encodePacked(
            _baseTokenURI,
            Strings.toString(tokenId)
        ));
    }
    
    function getRoyaltyInfo(
        uint256 tokenId,
        uint256 salePrice
    ) external view returns (address receiver, uint256 royaltyAmount) {
        require(_ownerOf(tokenId) != address(0), "Token does not exist");
        
        receiver = creators[tokenId];
        royaltyAmount = (salePrice * royaltyPercentage) / 10000;
    }
    
    function setMintPrice(uint256 _price) public onlyOwner {
        mintPrice = _price;
    }
    
    function setRoyaltyPercentage(uint256 _percentage) public onlyOwner {
        require(_percentage <= 1000, "Royalty too high"); // Max 10%
        royaltyPercentage = _percentage;
    }
    
    function withdraw() public onlyOwner {
        payable(owner()).transfer(address(this).balance);
    }
    
    function totalSupply() public view returns (uint256) {
        return _tokenIdCounter.current();
    }
}
```

### 市场合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import "@openzeppelin/contracts/token/ERC721/IERC721Receiver.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";

contract NFTMarket is IERC721Receiver, Ownable, ReentrancyGuard {
    struct Listing {
        address seller;
        address nftContract;
        uint256 tokenId;
        uint256 price;
        bool active;
    }
    
    mapping(bytes32 => Listing) public listings;
    mapping(address => uint256) public balances;
    
    uint256 public platformFee = 250; // 2.5%
    
    event Listed(
        bytes32 indexed listingId,
        address indexed seller,
        address nftContract,
        uint256 tokenId,
        uint256 price
    );
    
    event Sold(
        bytes32 indexed listingId,
        address indexed buyer,
        uint256 price
    );
    
    event Cancelled(bytes32 indexed listingId);
    
    constructor() Ownable(msg.sender) {}
    
    function listNFT(
        address nftContract,
        uint256 tokenId,
        uint256 price
    ) public returns (bytes32) {
        require(price > 0, "Price must be greater than 0");
        
        IERC721 nft = IERC721(nftContract);
        require(nft.ownerOf(tokenId) == msg.sender, "Not the owner");
        require(
            nft.isApprovedForAll(msg.sender, address(this)) ||
            nft.getApproved(tokenId) == address(this),
            "Not approved"
        );
        
        bytes32 listingId = keccak256(
            abi.encodePacked(nftContract, tokenId, block.timestamp)
        );
        
        nft.safeTransferFrom(msg.sender, address(this), tokenId);
        
        listings[listingId] = Listing({
            seller: msg.sender,
            nftContract: nftContract,
            tokenId: tokenId,
            price: price,
            active: true
        });
        
        emit Listed(listingId, msg.sender, nftContract, tokenId, price);
        
        return listingId;
    }
    
    function buyNFT(bytes32 listingId) public payable nonReentrant {
        Listing storage listing = listings[listingId];
        
        require(listing.active, "Listing not active");
        require(msg.value >= listing.price, "Insufficient payment");
        
        listing.active = false;
        
        IERC721 nft = IERC721(listing.nftContract);
        
        // 计算费用
        uint256 platformFeeAmount = (msg.value * platformFee) / 10000;
        uint256 royaltyAmount = 0;
        address creator = address(0);
        
        // 检查是否有版税
        try this.getRoyalty(listing.nftContract, listing.tokenId, msg.value) 
            returns (address _creator, uint256 _royalty) 
        {
            creator = _creator;
            royaltyAmount = _royalty;
        } catch {}
        
        uint256 sellerProceeds = msg.value - platformFeeAmount - royaltyAmount;
        
        // 转移 NFT
        nft.safeTransferFrom(address(this), msg.sender, listing.tokenId);
        
        // 支付
        balances[owner()] += platformFeeAmount;
        if (royaltyAmount > 0 && creator != address(0)) {
            balances[creator] += royaltyAmount;
        }
        balances[listing.seller] += sellerProceeds;
        
        // 退还多余 ETH
        if (msg.value > listing.price) {
            payable(msg.sender).transfer(msg.value - listing.price);
        }
        
        emit Sold(listingId, msg.sender, listing.price);
    }
    
    function cancelListing(bytes32 listingId) public {
        Listing storage listing = listings[listingId];
        
        require(listing.active, "Listing not active");
        require(listing.seller == msg.sender, "Not the seller");
        
        listing.active = false;
        
        IERC721(listing.nftContract).safeTransferFrom(
            address(this),
            msg.sender,
            listing.tokenId
        );
        
        emit Cancelled(listingId);
    }
    
    function getRoyalty(
        address nftContract,
        uint256 tokenId,
        uint256 salePrice
    ) external view returns (address, uint256) {
        // 调用 NFT 合约获取版税信息
        (address receiver, uint256 royaltyAmount) = 
            IERC2981(nftContract).royaltyInfo(tokenId, salePrice);
        return (receiver, royaltyAmount);
    }
    
    function withdraw() public {
        uint256 amount = balances[msg.sender];
        require(amount > 0, "No balance");
        
        balances[msg.sender] = 0;
        payable(msg.sender).transfer(amount);
    }
    
    function setPlatformFee(uint256 _fee) public onlyOwner {
        require(_fee <= 1000, "Fee too high"); // Max 10%
        platformFee = _fee;
    }
    
    function onERC721Received(
        address,
        address,
        uint256,
        bytes calldata
    ) external pure override returns (bytes4) {
        return IERC721Receiver.onERC721Received.selector;
    }
}

interface IERC2981 {
    function royaltyInfo(
        uint256 tokenId,
        uint256 salePrice
    ) external view returns (address receiver, uint256 royaltyAmount);
}
```

## 测试文件

```solidity
// test/NFTMarket.t.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "forge-std/Test.sol";
import "../src/MyNFT.sol";
import "../src/NFTMarket.sol";

contract NFTMarketTest is Test {
    MyNFT public nft;
    NFTMarket public market;
    
    address public owner;
    address public seller;
    address public buyer;
    
    function setUp() public {
        owner = address(this);
        seller = address(0x1);
        buyer = address(0x2);
        
        vm.deal(seller, 10 ether);
        vm.deal(buyer, 10 ether);
        
        nft = new MyNFT("My NFT", "MNFT", "https://api.example.com/");
        market = new NFTMarket();
    }
    
    function test_MintNFT() public {
        vm.prank(seller);
        nft.mint{value: 0.01 ether}("Test NFT", "Description", "ipfs://...");
        
        assertEq(nft.ownerOf(0), seller);
        assertEq(nft.totalSupply(), 1);
    }
    
    function test_ListNFT() public {
        vm.startPrank(seller);
        nft.mint{value: 0.01 ether}("Test NFT", "Description", "ipfs://...");
        nft.setApprovalForAll(address(market), true);
        
        bytes32 listingId = market.listNFT(address(nft), 0, 1 ether);
        vm.stopPrank();
        
        (address _seller,,,, bool active) = market.listings(listingId);
        assertEq(_seller, seller);
        assertTrue(active);
        assertEq(nft.ownerOf(0), address(market));
    }
    
    function test_BuyNFT() public {
        // 上架
        vm.startPrank(seller);
        nft.mint{value: 0.01 ether}("Test NFT", "Description", "ipfs://...");
        nft.setApprovalForAll(address(market), true);
        bytes32 listingId = market.listNFT(address(nft), 0, 1 ether);
        vm.stopPrank();
        
        // 购买
        vm.prank(buyer);
        market.buyNFT{value: 1 ether}(listingId);
        
        assertEq(nft.ownerOf(0), buyer);
        assertEq(market.balances(seller), 0.975 ether); // 1 ETH - 2.5% fee
    }
    
    function test_CancelListing() public {
        vm.startPrank(seller);
        nft.mint{value: 0.01 ether}("Test NFT", "Description", "ipfs://...");
        nft.setApprovalForAll(address(market), true);
        bytes32 listingId = market.listNFT(address(nft), 0, 1 ether);
        
        market.cancelListing(listingId);
        vm.stopPrank();
        
        assertEq(nft.ownerOf(0), seller);
    }
}
```

## 前端组件

```tsx
import { useAccount, useReadContract, useWriteContract } from 'wagmi';
import { parseEther, formatEther } from 'viem';
import { useState } from 'react';

export function NFTMarketplace() {
  const { address } = useAccount();
  const { writeContract } = useWriteContract();
  
  const [listingId, setListingId] = useState('');
  const [price, setPrice] = useState('');
  
  const handleMint = async () => {
    writeContract({
      address: NFT_ADDRESS,
      abi: NFT_ABI,
      functionName: 'mint',
      args: ['NFT Name', 'Description', 'ipfs://...'],
      value: parseEther('0.01'),
    });
  };
  
  const handleList = (tokenId: number, price: string) => {
    writeContract({
      address: MARKET_ADDRESS,
      abi: MARKET_ABI,
      functionName: 'listNFT',
      args: [NFT_ADDRESS, BigInt(tokenId), parseEther(price)],
    });
  };
  
  const handleBuy = (listingId: string, price: string) => {
    writeContract({
      address: MARKET_ADDRESS,
      abi: MARKET_ABI,
      functionName: 'buyNFT',
      args: [listingId as `0x${string}`],
      value: parseEther(price),
    });
  };
  
  return (
    <div className="nft-market">
      <h1>NFT 市场</h1>
      
      <section className="mint-section">
        <h2>铸造 NFT</h2>
        <button onClick={handleMint}>
          铸造 (0.01 ETH)
        </button>
      </section>
      
      {/* 更多 UI 组件... */}
    </div>
  );
}
```

## 下一步

[下一项目：简易 DEX →](03-defi.md)
