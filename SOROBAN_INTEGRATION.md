# Soroban Smart Contract Integration (Optional)

## Current Architecture - No Smart Contracts Required

ChainPe currently uses **direct Stellar payments** without smart contracts:

```
Customer → USDC Payment → Merchant
              ↓
         ChainPe validates payment
         using memo + amount
```

**This works great for:**
- Simple merchant payments
- Low fees (just network fees)
- Fast settlement
- No custody of funds

## When to Add Soroban Smart Contracts

Consider adding Soroban for advanced features:

### 1. **Escrow Payments**
Hold funds until delivery confirmation

### 2. **Payment Splitting**
Automatically split payments to multiple recipients (e.g., marketplace fees)

### 3. **Subscription Payments**
Recurring charges from customer accounts

### 4. **Conditional Refunds**
Automated refund logic based on time/events

### 5. **Dispute Resolution**
Multi-signature escrow with arbitration

## Example: Simple Escrow Contract

### Escrow Contract (Rust + Soroban)

```rust
// contracts/escrow/src/lib.rs
#![no_std]
use soroban_sdk::{contract, contractimpl, Address, Env, token};

#[contract]
pub struct EscrowContract;

#[contractimpl]
impl EscrowContract {
    /// Create escrow payment
    /// Customer deposits USDC, held until merchant confirms delivery
    pub fn create_escrow(
        env: Env,
        customer: Address,
        merchant: Address,
        usdc_token: Address,
        amount: i128,
        session_id: String,
    ) -> Result<(), Error> {
        // Verify customer authorization
        customer.require_auth();
        
        // Transfer USDC from customer to escrow contract
        let client = token::Client::new(&env, &usdc_token);
        client.transfer(&customer, &env.current_contract_address(), &amount);
        
        // Store escrow details
        env.storage().instance().set(&session_id, &EscrowData {
            customer: customer.clone(),
            merchant: merchant.clone(),
            amount,
            status: EscrowStatus::Pending,
            created_at: env.ledger().timestamp(),
        });
        
        Ok(())
    }
    
    /// Merchant confirms delivery → Release funds
    pub fn release_payment(
        env: Env,
        session_id: String,
        usdc_token: Address,
    ) -> Result<(), Error> {
        let mut escrow: EscrowData = env.storage().instance().get(&session_id)?;
        
        // Only merchant can release
        escrow.merchant.require_auth();
        
        // Ensure still pending
        if escrow.status != EscrowStatus::Pending {
            return Err(Error::AlreadyCompleted);
        }
        
        // Transfer USDC to merchant
        let client = token::Client::new(&env, &usdc_token);
        client.transfer(
            &env.current_contract_address(),
            &escrow.merchant,
            &escrow.amount
        );
        
        // Mark as completed
        escrow.status = EscrowStatus::Completed;
        env.storage().instance().set(&session_id, &escrow);
        
        Ok(())
    }
    
    /// Customer requests refund (if timeout)
    pub fn refund_payment(
        env: Env,
        session_id: String,
        usdc_token: Address,
    ) -> Result<(), Error> {
        let mut escrow: EscrowData = env.storage().instance().get(&session_id)?;
        
        // Only customer can refund
        escrow.customer.require_auth();
        
        // Check timeout (e.g., 24 hours)
        let timeout = 24 * 60 * 60; // 24 hours in seconds
        if env.ledger().timestamp() < escrow.created_at + timeout {
            return Err(Error::TimeoutNotReached);
        }
        
        // Transfer USDC back to customer
        let client = token::Client::new(&env, &usdc_token);
        client.transfer(
            &env.current_contract_address(),
            &escrow.customer,
            &escrow.amount
        );
        
        // Mark as refunded
        escrow.status = EscrowStatus::Refunded;
        env.storage().instance().set(&session_id, &escrow);
        
        Ok(())
    }
}

#[derive(Clone)]
pub struct EscrowData {
    customer: Address,
    merchant: Address,
    amount: i128,
    status: EscrowStatus,
    created_at: u64,
}

#[derive(Clone, PartialEq)]
pub enum EscrowStatus {
    Pending,
    Completed,
    Refunded,
}

#[derive(Copy, Clone, Debug, Eq, PartialEq)]
#[repr(u32)]
pub enum Error {
    AlreadyCompleted = 1,
    TimeoutNotReached = 2,
}
```

### Deploy Escrow Contract

```bash
# Install Soroban CLI
cargo install --locked soroban-cli

# Build contract
cd contracts/escrow
soroban contract build

# Deploy to testnet
soroban contract deploy \
  --wasm target/wasm32-unknown-unknown/release/escrow.wasm \
  --source YOUR_SECRET_KEY \
  --network testnet

# Output: Contract ID (e.g., CXXXXXXXXX...)
```

### Integrate with ChainPe Backend

```python
# app/services/soroban_escrow.py
from stellar_sdk import SorobanServer, Keypair, TransactionBuilder, Network
from stellar_sdk.soroban_rpc import GetTransactionStatus
from stellar_sdk import xdr

class SorobanEscrowService:
    def __init__(self):
        self.server = SorobanServer("https://soroban-testnet.stellar.org")
        self.contract_id = "YOUR_ESCROW_CONTRACT_ID"
        self.usdc_contract = "USDC_TOKEN_CONTRACT_ID"
    
    async def create_escrow_payment(
        self,
        customer_address: str,
        merchant_address: str,
        amount: str,
        session_id: str
    ):
        """Create escrow payment via smart contract."""
        
        # Build Soroban transaction
        source = Keypair.from_secret("CUSTOMER_SECRET")
        
        # Invoke create_escrow function
        tx = (
            TransactionBuilder(
                source_account=self.server.load_account(source.public_key),
                network_passphrase=Network.TESTNET_NETWORK_PASSPHRASE,
            )
            .append_invoke_contract_function_op(
                contract_id=self.contract_id,
                function_name="create_escrow",
                parameters=[
                    scval.to_address(customer_address),
                    scval.to_address(merchant_address),
                    scval.to_address(self.usdc_contract),
                    scval.to_int128(int(float(amount) * 10**7)),  # USDC has 7 decimals
                    scval.to_string(session_id),
                ]
            )
            .build()
        )
        
        # Sign and submit
        tx.sign(source)
        response = self.server.send_transaction(tx)
        
        return response.hash
    
    async def release_escrow(
        self,
        merchant_keypair: Keypair,
        session_id: str
    ):
        """Release escrowed funds to merchant."""
        
        tx = (
            TransactionBuilder(
                source_account=self.server.load_account(merchant_keypair.public_key),
                network_passphrase=Network.TESTNET_NETWORK_PASSPHRASE,
            )
            .append_invoke_contract_function_op(
                contract_id=self.contract_id,
                function_name="release_payment",
                parameters=[
                    scval.to_string(session_id),
                    scval.to_address(self.usdc_contract),
                ]
            )
            .build()
        )
        
        tx.sign(merchant_keypair)
        response = self.server.send_transaction(tx)
        
        return response.hash
    
    async def refund_escrow(
        self,
        customer_keypair: Keypair,
        session_id: str
    ):
        """Refund payment to customer after timeout."""
        
        tx = (
            TransactionBuilder(
                source_account=self.server.load_account(customer_keypair.public_key),
                network_passphrase=Network.TESTNET_NETWORK_PASSPHRASE,
            )
            .append_invoke_contract_function_op(
                contract_id=self.contract_id,
                function_name="refund_payment",
                parameters=[
                    scval.to_string(session_id),
                    scval.to_address(self.usdc_contract),
                ]
            )
            .build()
        )
        
        tx.sign(customer_keypair)
        response = self.server.send_transaction(tx)
        
        return response.hash
```

### Add Escrow Endpoints

```python
# app/routes/escrow.py
from fastapi import APIRouter, Depends, HTTPException
from app.services.soroban_escrow import SorobanEscrowService
from app.dependencies import require_merchant

router = APIRouter(prefix="/api/escrow", tags=["Escrow"])
escrow_service = SorobanEscrowService()

@router.post("/create")
async def create_escrow_payment(
    amount: float,
    session_id: str,
    merchant = Depends(require_merchant)
):
    """Create escrow payment using Soroban smart contract."""
    
    try:
        tx_hash = await escrow_service.create_escrow_payment(
            customer_address="CUSTOMER_ADDRESS",  # From request
            merchant_address=merchant.stellar_address,
            amount=str(amount),
            session_id=session_id
        )
        
        return {
            "status": "escrow_created",
            "tx_hash": tx_hash,
            "message": "Funds locked in escrow"
        }
    except Exception as e:
        raise HTTPException(status_code=400, detail=str(e))

@router.post("/release/{session_id}")
async def release_escrow(
    session_id: str,
    merchant = Depends(require_merchant)
):
    """Release escrow funds to merchant."""
    
    # Get merchant keypair (in production, use secure key management)
    merchant_keypair = Keypair.from_secret(merchant.stellar_secret)
    
    tx_hash = await escrow_service.release_escrow(
        merchant_keypair,
        session_id
    )
    
    return {
        "status": "released",
        "tx_hash": tx_hash,
        "message": "Funds released to merchant"
    }

@router.post("/refund/{session_id}")
async def refund_escrow(session_id: str):
    """Customer requests refund after timeout."""
    
    # Validate timeout passed
    # Get customer keypair
    
    tx_hash = await escrow_service.refund_escrow(
        customer_keypair,
        session_id
    )
    
    return {
        "status": "refunded",
        "tx_hash": tx_hash,
        "message": "Funds refunded to customer"
    }
```

## Example: Payment Splitting Contract

For marketplace scenarios where you need to split payments:

```rust
// Split payment between merchant + platform fee
pub fn split_payment(
    env: Env,
    customer: Address,
    merchant: Address,
    platform: Address,
    usdc_token: Address,
    total_amount: i128,
    platform_fee_percent: u32, // e.g., 250 = 2.5%
) -> Result<(), Error> {
    customer.require_auth();
    
    // Calculate split
    let platform_fee = (total_amount * platform_fee_percent as i128) / 10000;
    let merchant_amount = total_amount - platform_fee;
    
    let client = token::Client::new(&env, &usdc_token);
    
    // Transfer to merchant
    client.transfer(&customer, &merchant, &merchant_amount);
    
    // Transfer platform fee
    client.transfer(&customer, &platform, &platform_fee);
    
    Ok(())
}
```

## Comparison: Direct Payment vs Escrow

| Feature | Current (Direct) | With Soroban Escrow |
|---------|------------------|---------------------|
| **Fees** | ~0.00001 XLM | ~0.1 XLM + contract fees |
| **Speed** | 3-5 seconds | 3-5 seconds + contract execution |
| **Custody** | No (direct to merchant) | Yes (contract holds funds) |
| **Refunds** | Manual | Automated (timeout-based) |
| **Trust** | Required | Smart contract enforced |
| **Complexity** | Low | Medium-High |
| **Best For** | B2C payments | Marketplaces, P2P, high-value |

## Recommendation

**For most use cases, keep the current direct payment approach!**

Only add Soroban if you specifically need:
- ✅ Escrow/holding funds
- ✅ Automated payment splitting
- ✅ Trustless transactions
- ✅ Complex conditional logic

The current implementation is simpler, cheaper, and works perfectly for merchant checkout flows.

## Resources

- **Soroban Docs:** https://soroban.stellar.org/docs
- **Soroban Examples:** https://github.com/stellar/soroban-examples
- **Soroban CLI:** https://soroban.stellar.org/docs/getting-started/setup
- **Smart Contract Tutorial:** https://soroban.stellar.org/docs/tutorials

## Quick Start with Soroban (If You Want to Experiment)

```bash
# 1. Install Soroban CLI
cargo install --locked soroban-cli

# 2. Create new contract
soroban contract init escrow

# 3. Build
cd escrow
soroban contract build

# 4. Deploy to testnet
soroban contract deploy \
  --wasm target/wasm32-unknown-unknown/release/escrow.wasm \
  --source YOUR_SECRET \
  --network testnet

# 5. Invoke function
soroban contract invoke \
  --id CONTRACT_ID \
  --source YOUR_SECRET \
  --network testnet \
  -- \
  create_escrow \
  --customer GXXX... \
  --merchant GYYY... \
  --amount 1000000
```

---

**Bottom Line:** Your current payment gateway is production-ready without Soroban. Add smart contracts only if you need escrow or advanced features!
