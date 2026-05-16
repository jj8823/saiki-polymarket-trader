---
name: polymarket-api
description: Deep integration guide for Polymarket's CLOB API V2, Gamma API, and on-chain data. Use when building trading functionality, fetching market data, or implementing order execution with Deposit Wallets.
---

# Polymarket API Integration Skill

## Overview

This skill provides comprehensive guidance for integrating with Polymarket's V2 APIs and smart contracts, utilizing the new Deposit Wallet architecture.

## API Endpoints

### CLOB API (Central Limit Order Book)
Base URL: `https://clob.polymarket.com`

#### Authentication Levels
- **Level 0 (Public)**: Market data, orderbooks, prices
- **Level 1 (Signer)**: Create/derive API keys
- **Level 2 (Authenticated)**: Trading, orders, positions

#### Key Endpoints
```
GET  /markets              # List all markets
GET  /markets/{token_id}   # Get specific market
GET  /price?token_id=X     # Get current price
GET  /midpoint?token_id=X  # Get midpoint price
GET  /book?token_id=X      # Get orderbook
GET  /trades               # Get user trades
POST /order                # Place order
DELETE /order/{id}         # Cancel order
GET  /positions            # Get positions
```

### Gamma API (Market Metadata)
Base URL: `https://gamma-api.polymarket.com`

```
GET /events              # List events
GET /events/{slug}       # Get event details
GET /markets             # List markets
GET /markets/{id}        # Get market details
```

## Python Implementation Patterns (V2 API)

### 1. Deposit Wallet Deployment & Funding
New API users must use a Deposit Wallet. You deploy it via the Relayer client.

```python
import os
import time
from py_builder_relayer_client.client import RelayClient
from py_builder_signing_sdk.config import BuilderApiKeyCreds, BuilderConfig
from py_builder_relayer_client.models import DepositWalletCall, TransactionType

# Initialize Relayer Client
builder_config = BuilderConfig(
    local_builder_creds=BuilderApiKeyCreds(
        key=os.environ["BUILDER_API_KEY"],
        secret=os.environ["BUILDER_SECRET"],
        passphrase=os.environ["BUILDER_PASS_PHRASE"],
    )
)

relayer = RelayClient(
    os.environ["RELAYER_URL"],
    int(os.environ.get("CHAIN_ID", "137")),
    os.environ["PRIVATE_KEY"],
    builder_config,
)

# Derive and deploy deposit wallet
deposit_wallet = relayer.get_expected_deposit_wallet()
response = relayer.deploy_deposit_wallet()
confirmed = response.wait()

# Example: Execute a Wallet Batch (e.g., for Token Approval)
nonce_payload = relayer.get_nonce(
    relayer.signer.address(),
    TransactionType.WALLET.value,
)
wallet_nonce = str(nonce_payload["nonce"])

call = DepositWalletCall(
    target=os.environ["PUSD_ADDRESS"],
    value="0",
    data="0x..." # approve_calldata
)

response = relayer.execute_deposit_wallet_batch(
    calls=[call],
    wallet_address=deposit_wallet,
    nonce=wallet_nonce,
    deadline=str(int(time.time()) + 600),
)
confirmed = response.wait()
```

### 2. Initialize CLOB Client (V2)
```python
import os
from py_clob_client_v2 import (
    ApiCreds,
    AssetType,
    BalanceAllowanceParams,
    ClobClient,
    OrderArgs,
    OrderType,
    PartialCreateOrderOptions,
    Side,
    SignatureTypeV2,
)

class PolymarketService:
    def __init__(self, deposit_wallet: str):
        # API credentials must be passed during initialization for L2 methods
        creds = ApiCreds(
            api_key=os.environ["CLOB_API_KEY"],
            api_secret=os.environ["CLOB_SECRET"],
            api_passphrase=os.environ["CLOB_PASS_PHRASE"],
        )
        
        self.client = ClobClient(
            host=os.environ.get("CLOB_API_URL", "https://clob.polymarket.com"),
            chain_id=int(os.environ.get("CHAIN_ID", "137")),
            key=os.environ["PRIVATE_KEY"],
            creds=creds,
            signature_type=SignatureTypeV2.POLY_1271, # Required for Deposit Wallets
            funder=deposit_wallet,
        )
        
    async def sync_balance(self):
        """Sync CLOB balance after funding or changing allowances."""
        self.client.update_balance_allowance(
            BalanceAllowanceParams(
                asset_type=AssetType.COLLATERAL,
                signature_type=SignatureTypeV2.POLY_1271,
            )
        )
    
    async def get_market_data(self, token_id: str) -> dict:
        """Fetch comprehensive market data."""
        return {
            "price": self.client.get_price(token_id, Side.BUY),
            "midpoint": self.client.get_midpoint(token_id),
            "book": self.client.get_order_book(token_id),
            "spread": self.client.get_spread(token_id),
        }
    
    async def place_order(
        self,
        token_id: str,
        side: Side,
        price: float,
        size: float,
        tick_size: str = "0.01",
        neg_risk: bool = False,
        order_type: OrderType = OrderType.GTC
    ) -> dict:
        """Place a limit order using V2 API."""
        return self.client.create_and_post_order(
            order_args=OrderArgs(
                token_id=token_id,
                price=price,
                size=size,
                side=side,
            ),
            options=PartialCreateOrderOptions(
                tick_size=tick_size, 
                neg_risk=neg_risk
            ),
            order_type=order_type,
        )
```

### WebSocket Subscription
```python
import asyncio
import websockets
import json

async def subscribe_market_updates(token_ids: list[str]):
    """Subscribe to real-time market updates."""
    uri = "wss://ws-subscriptions-clob.polymarket.com/ws/market"
    
    async with websockets.connect(uri) as ws:
        await ws.send(json.dumps({
            "type": "subscribe",
            "markets": token_ids
        }))
        
        async for message in ws:
            data = json.loads(message)
            yield data
```

### Gamma API Client
```python
import httpx

class GammaClient:
    BASE_URL = "https://gamma-api.polymarket.com"
    
    def __init__(self):
        self.client = httpx.AsyncClient(base_url=self.BASE_URL)
    
    async def get_active_markets(self) -> list[dict]:
        """Fetch all active markets."""
        response = await self.client.get("/markets", params={"active": True})
        return response.json()
    
    async def get_event(self, slug: str) -> dict:
        """Fetch event with all markets."""
        response = await self.client.get(f"/events/{slug}")
        return response.json()
```

## Order Types

- **GTC** (Good Till Cancelled): Stays until filled or cancelled
- **GTD** (Good Till Date): Expires at specified time
- **FOK** (Fill or Kill): Must fill entirely or cancel
- **FAK** (Fill And Kill): Fills what's available, cancel rest

## Price Calculations

```python
def calculate_implied_probability(price: float) -> float:
    """Convert price to implied probability."""
    return price  # Prices ARE probabilities (0-1)

def calculate_cost(price: float, shares: float) -> float:
    """Calculate cost to buy shares."""
    return price * shares

def calculate_pnl(
    entry_price: float,
    current_price: float,
    shares: float,
    side: str
) -> float:
    """Calculate unrealized P&L."""
    if side == "BUY" or side == Side.BUY:
        return (current_price - entry_price) * shares
    return (entry_price - current_price) * shares
```

## Error Handling

```python
from py_clob_client_v2.exceptions import PolymarketException

try:
    result = client.post_order(order)
except PolymarketException as e:
    if "INSUFFICIENT_BALANCE" in str(e):
        # Handle insufficient funds (ensure pUSD is in Deposit Wallet, not EOA)
        pass
    elif "INVALID_PRICE" in str(e):
        # Handle price out of range
        pass
    elif "INVALID_SIGNATURE" in str(e):
        # Check if using POLY_1271 and Deposit Wallet funder
        pass
    raise
```

## Rate Limits

- Public endpoints: ~100 requests/minute
- Authenticated endpoints: ~1000 requests/minute
- WebSocket: Varies by subscription type

Always implement exponential backoff and request queuing.

## Key Contract Addresses (Polygon)

```python
CONTRACTS = {
    "CTF_EXCHANGE": "0x4bFb41d5B3570DeFd03C39a9A4D8dE6Bd8B8982E",
    "NEG_RISK_CTF_EXCHANGE": "0xC5d563A36AE78145C45a50134d48A1215220f80a",
    "CONDITIONAL_TOKENS": "0x4D97DCd97eC945f40cF65F87097ACe5EA0476045",
    "USDC": "0x2791Bca1f2de4661ED88A30C99A7a9449Aa84174",
}
```
