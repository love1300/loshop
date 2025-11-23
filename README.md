# How Two Friends Built an NFT Marketplace "Loveshop: Fuzzy Ice Cubes" on Bali 🌴

## The Beginning: Sunset and Inspiration

It all started on Echo Beach in Canggu. My friend Mark and I were watching another magical Bali sunset, sipping fresh coconuts after a long day of surfing. We were both burned out from our corporate jobs and decided to take a 3-month sabbatical to Bali.

"Hey," Mark said, pointing at a local artist selling digital art prints to tourists. "What if we could help these guys reach global collectors without all the middlemen?"

That's how "Loveshop: Fuzzy Ice Cubes" was born - an NFT marketplace focused on digital art with a cool, icy aesthetic.

## Tech Stack Decisions

Working from Dojo Bali coworking space, we settled on:

- **Blockchain**: Ethereum (for its robust NFT ecosystem)
- **Backend**: Rust (for performance and safety)
- **Smart Contracts**: Solidity with Rust bindings
- **Frontend**: React with ethers.js

## The Core Smart Contract in Solidity

Here's our main NFT contract:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract FuzzyIceCubes is ERC721, ReentrancyGuard {
    uint256 public nextTokenId;
    address public owner;
    uint256 public mintPrice = 0.05 ether;
    
    struct IceCube {
        string name;
        string metadataURI;
        uint256 coolnessFactor; // 1-100
        uint256 creationTime;
        address creator;
    }
    
    mapping(uint256 => IceCube) public iceCubes;
    mapping(address => uint256[]) public userIceCubes;
    
    event IceCubeMinted(
        uint256 indexed tokenId,
        address indexed creator,
        string name,
        uint256 coolnessFactor
    );
    
    event IceCubeSold(
        uint256 indexed tokenId,
        address from,
        address to,
        uint256 price
    );
    
    constructor() ERC721("FuzzyIceCubes", "FIC") {
        owner = msg.sender;
        nextTokenId = 1;
    }
    
    function mintIceCube(
        string memory name,
        string memory metadataURI,
        uint256 coolnessFactor
    ) external payable nonReentrant {
        require(msg.value >= mintPrice, "Insufficient minting fee");
        require(coolnessFactor > 0 && coolnessFactor <= 100, "Invalid coolness");
        
        uint256 tokenId = nextTokenId++;
        _safeMint(msg.sender, tokenId);
        
        iceCubes[tokenId] = IceCube({
            name: name,
            metadataURI: metadataURI,
            coolnessFactor: coolnessFactor,
            creationTime: block.timestamp,
            creator: msg.sender
        });
        
        userIceCubes[msg.sender].push(tokenId);
        
        emit IceCubeMinted(tokenId, msg.sender, name, coolnessFactor);
    }
    
    function transferIceCube(
        address to,
        uint256 tokenId,
        uint256 price
    ) external {
        require(ownerOf(tokenId) == msg.sender, "Not owner");
        _transfer(msg.sender, to, tokenId);
        emit IceCubeSold(tokenId, msg.sender, to, price);
    }
    
    function getUserIceCubes(address user) 
        external 
        view 
        returns (uint256[] memory) 
    {
        return userIceCubes[user];
    }
    
    function getIceCubeDetails(uint256 tokenId)
        external
        view
        returns (IceCube memory)
    {
        require(_exists(tokenId), "Token doesn't exist");
        return iceCubes[tokenId];
    }
}
```

## Rust Backend Service

Here's our Rust backend for handling metadata and marketplace logic:

```rust
use actix_web::{web, App, HttpResponse, HttpServer, Result};
use serde::{Deserialize, Serialize};
use std::collections::HashMap;
use std::sync::Mutex;

#[derive(Serialize, Deserialize, Clone)]
pub struct IceCubeMetadata {
    pub name: String,
    pub description: String,
    pub image: String,
    pub attributes: Vec<Attribute>,
    pub coolness_factor: u8,
    pub creator: String,
}

#[derive(Serialize, Deserialize, Clone)]
pub struct Attribute {
    pub trait_type: String,
    pub value: String,
}

pub struct AppState {
    pub metadata_store: Mutex<HashMap<u64, IceCubeMetadata>>,
    pub user_profiles: Mutex<HashMap<String, UserProfile>>,
}

#[derive(Serialize, Deserialize, Clone)]
pub struct UserProfile {
    pub wallet_address: String,
    pub username: String,
    pub ice_cubes_owned: Vec<u64>,
    pub total_coolness: u64,
}

#[derive(Deserialize)]
pub struct MintRequest {
    pub name: String,
    pub description: String,
    pub image_url: String,
    pub attributes: Vec<Attribute>,
    pub coolness_factor: u8,
    pub creator_wallet: String,
}

pub async fn mint_metadata(
    data: web::Data<AppState>,
    req: web::Json<MintRequest>,
) -> Result<HttpResponse> {
    let token_id = generate_token_id();
    
    let metadata = IceCubeMetadata {
        name: req.name.clone(),
        description: req.description.clone(),
        image: req.image_url.clone(),
        attributes: req.attributes.clone(),
        coolness_factor: req.coolness_factor,
        creator: req.creator_wallet.clone(),
    };
    
    // Store metadata
    {
        let mut store = data.metadata_store.lock().unwrap();
        store.insert(token_id, metadata.clone());
    }
    
    // Update user profile
    {
        let mut profiles = data.user_profiles.lock().unwrap();
        let user_profile = profiles
            .entry(req.creator_wallet.clone())
            .or_insert_with(|| UserProfile {
                wallet_address: req.creator_wallet.clone(),
                username: format!("user_{}", &req.creator_wallet[..8]),
                ice_cubes_owned: Vec::new(),
                total_coolness: 0,
            });
        
        user_profile.ice_cubes_owned.push(token_id);
        user_profile.total_coolness += req.coolness_factor as u64;
    }
    
    Ok(HttpResponse::Ok().json(serde_json::json!({
        "token_id": token_id,
        "metadata": metadata,
        "status": "minted"
    })))
}

pub async fn get_ice_cube_metadata(
    data: web::Data<AppState>,
    path: web::Path<u64>,
) -> Result<HttpResponse> {
    let token_id = path.into_inner();
    let store = data.metadata_store.lock().unwrap();
    
    match store.get(&token_id) {
        Some(metadata) => Ok(HttpResponse::Ok().json(metadata)),
        None => Ok(HttpResponse::NotFound().body("IceCube not found")),
    }
}

pub async fn get_user_profile(
    data: web::Data<AppState>,
    path: web::Path<String>,
) -> Result<HttpResponse> {
    let wallet_address = path.into_inner();
    let profiles = data.user_profiles.lock().unwrap();
    
    match profiles.get(&wallet_address) {
        Some(profile) => Ok(HttpResponse::Ok().json(profile)),
        None => Ok(HttpResponse::NotFound().body("User not found")),
    }
}

fn generate_token_id() -> u64 {
    use std::time::{SystemTime, UNIX_EPOCH};
    let start = SystemTime::now();
    start.duration_since(UNIX_EPOCH).unwrap().as_nanos() as u64
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let app_state = web::Data::new(AppState {
        metadata_store: Mutex::new(HashMap::new()),
        user_profiles: Mutex::new(HashMap::new()),
    });

    println!("🚀 Starting Loveshop Fuzzy Ice Cubes API server at http://localhost:8080");
    
    HttpServer::new(move || {
        App::new()
            .app_data(app_state.clone())
            .route("/metadata/mint", web::post().to(mint_metadata))
            .route("/metadata/{token_id}", web::get().to(get_ice_cube_metadata))
            .route("/user/{wallet_address}", web::get().to(get_user_profile))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

## Frontend Integration with React

Here's our main React component for minting NFTs:

```jsx
import React, { useState, useEffect } from 'react';
import { ethers } from 'ethers';
import './Loveshop.css';

const LoveshopMarketplace = () => {
    const [currentAccount, setCurrentAccount] = useState('');
    const [iceCubes, setIceCubes] = useState([]);
    const [isMinting, setIsMinting] = useState(false);
    const [mintForm, setMintForm] = useState({
        name: '',
        description: '',
        imageUrl: '',
        coolnessFactor: 50
    });

    const CONTRACT_ADDRESS = "YOUR_CONTRACT_ADDRESS";
    const CONTRACT_ABI = [/* Your Contract ABI */];

    useEffect(() => {
        checkWalletConnection();
        loadUserIceCubes();
    }, [currentAccount]);

    const checkWalletConnection = async () => {
        if (window.ethereum) {
            try {
                const accounts = await window.ethereum.request({
                    method: 'eth_requestAccounts'
                });
                setCurrentAccount(accounts[0]);
            } catch (error) {
                console.error("Error connecting to wallet:", error);
            }
        }
    };

    const connectWallet = async () => {
        if (window.ethereum) {
            try {
                const accounts = await window.ethereum.request({
                    method: 'eth_requestAccounts'
                });
                setCurrentAccount(accounts[0]);
            } catch (error) {
                console.error("Error connecting wallet:", error);
            }
        } else {
            alert('Please install MetaMask!');
        }
    };

    const mintIceCube = async () => {
        if (!currentAccount) {
            alert('Please connect your wallet first!');
            return;
        }

        setIsMinting(true);
        try {
            const provider = new ethers.providers.Web3Provider(window.ethereum);
            const signer = provider.getSigner();
            const contract = new ethers.Contract(CONTRACT_ADDRESS, CONTRACT_ABI, signer);

            // First, upload metadata to our backend
            const metadataResponse = await fetch('http://localhost:8080/metadata/mint', {
                method: 'POST',
                headers: {
                    'Content-Type': 'application/json',
                },
                body: JSON.stringify({
                    ...mintForm,
                    creator_wallet: currentAccount,
                    attributes: [
                        { trait_type: "Ice Type", value: "Fuzzy" },
                        { trait_type: "Temperature", value: "Freezing" }
                    ]
                })
            });

            const metadata = await metadataResponse.json();

            // Then mint on blockchain
            const mintTx = await contract.mintIceCube(
                mintForm.name,
                metadata.metadata_uri,
                mintForm.coolnessFactor,
                { value: ethers.utils.parseEther("0.05") }
            );

            await mintTx.wait();
            alert('🎉 Ice Cube minted successfully!');
            setMintForm({ name: '', description: '', imageUrl: '', coolnessFactor: 50 });
            loadUserIceCubes();
        } catch (error) {
            console.error('Error minting Ice Cube:', error);
            alert('Error minting Ice Cube');
        } finally {
            setIsMinting(false);
        }
    };

    const loadUserIceCubes = async () => {
        if (!currentAccount) return;

        try {
            const response = await fetch(`http://localhost:8080/user/${currentAccount}`);
            const userData = await response.json();
            setIceCubes(userData.ice_cubes_owned || []);
        } catch (error) {
            console.error('Error loading ice cubes:', error);
        }
    };

    return (
        <div className="loveshop-container">
            <header className="loveshop-header">
                <h1>❄️ Loveshop: Fuzzy Ice Cubes 🧊</h1>
                {!currentAccount ? (
                    <button onClick={connectWallet} className="connect-wallet-btn">
                        Connect Wallet
                    </button>
                ) : (
                    <p className="wallet-address">
                        Connected: {currentAccount.slice(0, 6)}...{currentAccount.slice(-4)}
                    </p>
                )}
            </header>

            <div className="mint-section">
                <h2>Mint Your Fuzzy Ice Cube</h2>
                <div className="mint-form">
                    <input
                        type="text"
                        placeholder="Ice Cube Name"
                        value={mintForm.name}
                        onChange={(e) => setMintForm({...mintForm, name: e.target.value})}
                    />
                    <textarea
                        placeholder="Description"
                        value={mintForm.description}
                        onChange={(e) => setMintForm({...mintForm, description: e.target.value})}
                    />
                    <input
                        type="text"
                        placeholder="Image URL"
                        value={mintForm.imageUrl}
                        onChange={(e) => setMintForm({...mintForm, imageUrl: e.target.value})}
                    />
                    <div className="coolness-slider">
                        <label>Coolness Factor: {mintForm.coolnessFactor}</label>
                        <input
                            type="range"
                            min="1"
                            max="100"
                            value={mintForm.coolnessFactor}
                            onChange={(e) => setMintForm({...mintForm, coolnessFactor: parseInt(e.target.value)})}
                        />
                    </div>
                    <button 
                        onClick={mintIceCube} 
                        disabled={isMinting}
                        className="mint-btn"
                    >
                        {isMinting ? 'Minting...' : 'Mint Ice Cube (0.05 ETH)'}
                    </button>
                </div>
            </div>

            <div className="gallery-section">
                <h2>Your Fuzzy Ice Cubes Collection</h2>
                <div className="ice-cubes-grid">
                    {iceCubes.map((cube, index) => (
                        <div key={index} className="ice-cube-card">
                            <h3>{cube.name}</h3>
                            <p>Coolness: {cube.coolness_factor}/100</p>
                        </div>
                    ))}
                </div>
            </div>
        </div>
    );
};

export default LoveshopMarketplace;
```

## Rust Blockchain Interaction

Here's our Rust service for blockchain events:

```rust
use web3::types::{H160, U256, Filter, TransactionId};
use std::time::Duration;
use tokio::time::sleep;

pub struct BlockchainListener {
    web3: web3::Web3<web3::transports::WebSocket>,
    contract_address: H160,
}

impl BlockchainListener {
    pub async fn new(ws_url: &str, contract_address: H160) -> Result<Self, web3::Error> {
        let transport = web3::transports::WebSocket::new(ws_url).await?;
        let web3 = web3::Web3::new(transport);
        Ok(Self { web3, contract_address })
    }

    pub async fn listen_for_mints(&self) {
        println!("👂 Listening for new Ice Cube mints...");
        
        loop {
            match self.get_mint_events().await {
                Ok(events) => {
                    for event in events {
                        println!("🎉 New Ice Cube minted: {:?}", event);
                        self.handle_new_mint(event).await;
                    }
                }
                Err(e) => eprintln!("Error reading events: {}", e),
            }
            
            sleep(Duration::from_secs(5)).await;
        }
    }

    async fn get_mint_events(&self) -> web3::Result<Vec<MintEvent>> {
        let filter = Filter::builder()
            .address(vec![self.contract_address])
            .topics(
                Some(vec![
                    "0x0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f0f".into() // Mint event signature
                ]),
                None,
                None,
                None,
            )
            .build();

        let logs = self.web3.eth().logs(filter).await?;
        let events = self.parse_mint_events(logs).await;
        Ok(events)
    }

    async fn handle_new_mint(&self, event: MintEvent) {
        // Update our backend database, send notifications, etc.
        println!("🔄 Processing new mint: Token #{}, Creator: {}", 
                 event.token_id, event.creator);
        
        // Here you'd typically update your database or trigger other business logic
    }
}

#[derive(Debug)]
pub struct MintEvent {
    pub token_id: U256,
    pub creator: H160,
    pub name: String,
    pub coolness_factor: u64,
}
```

## The Bali Development Experience

Working from Canggu was magical. Our typical day looked like:

1. **Morning**: Surf session at Echo Beach
2. **Late Morning**: Coding at Dojo Bali with fresh coconut water
3. **Afternoon**: Smart contract development and testing
4. **Evening**: Deployment and planning sessions at a beach warung

The most memorable moment was when we deployed our first smart contract. We celebrated with nasi campur at our favorite local warung, watching the sunset over the Indian Ocean.

## Lessons Learned

1. **Rust + Ethereum** is a powerful but challenging combo
2. **Testing is crucial** - we lost some test ETH to bugs early on
3. **Bali's digital nomad community** is incredibly supportive
4. **Simple UX** matters more than complex features for NFT adoption

## What's Next for Loveshop

We're planning to:

- Add fractional ownership of high-value Ice Cubes
- Implement a royalty system for creators
- Build a mobile app with React Native
- Add social features and Ice Cube "melting" mechanics

The journey continues from our next destination - likely Mexico or Portugal! 🌍

---

*Built with ❤️ from Bali. Follow our journey on GitHub!*
