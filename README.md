# NEAR Storage Staking Guide

## Introduction

Storage staking is a fundamental mechanism in the NEAR Protocol that ensures network efficiency and prevents state bloat. Unlike traditional blockchains where storing data on-chain is "free" after transaction fees, NEAR implements an explicit storage cost model where developers and users must stake NEAR tokens to maintain data on the blockchain. This guide walks you through understanding storage staking mechanics, calculating costs accurately, and optimizing your smart contracts to minimize storage expenses.

## Storage Staking Fundamentals

Storage staking in NEAR operates on a simple principle: **storage bytes × price per byte = NEAR tokens required**. Currently, the storage price is **0.1 Ⓝ per 100KB** (or 0.000001 Ⓝ per byte). When you store data on NEAR, you're not burning tokens—you're staking them. This staked amount remains reserved in your account, earning the same yields as delegated tokens while locked.

The distinction between staking and burning is critical. Staked tokens for storage:
- Remain in your account as reserved balance
- Can be unstaked when you delete contract state or reduce data
- Earn staking rewards proportional to the network's validator commission
- Are separate from your liquid NEAR balance

This mechanism prevents malicious actors from creating unlimited state growth on validators' machines. Each byte of state requires a corresponding stake, making state bloat economically unfeasible. When you interact with a contract, the contract creator or transaction sender must cover storage expansion costs.

## Cost Calculation Examples

### Example 1: Simple Account Creation Storage Cost

**Input:** Creating a new NEAR account requires storing account metadata (~500 bytes)

**Calculation:**
- Storage bytes: 500
- Price per byte: 0.000001 Ⓝ
- Total: 500 × 0.000001 = **0.0005 Ⓝ**

### Example 2: Smart Contract Deployment Storage Cost

**Input:** Deploying a small smart contract with 150KB compiled WASM binary + metadata (~5KB)

**Calculation:**
- Total bytes: 150,000 + 5,000 = 155,000 bytes
- Price per byte: 0.000001 Ⓝ
- Total: 155,000 × 0.000001 = **0.155 Ⓝ**

### Example 3: Fungible Token Transfer Storage Cost

**Input:** Registering a new account for a fungible token (storage required for balance tracking: ~350 bytes)

**Calculation:**
- Storage bytes: 350
- Price per byte: 0.000001 Ⓝ
- Total: 350 × 0.000001 = **0.00035 Ⓝ**

### Example 4: NFT Minting Storage Cost

**Input:** Minting an NFT with metadata (IPFS hash + attributes: ~2KB total)

**Calculation:**
- Storage bytes: 2,000
- Price per byte: 0.000001 Ⓝ
- Total: 2,000 × 0.000001 = **0.002 Ⓝ**

### Example 5: Complex dApp State Example with Cumulative Costs

**Input:** A DeFi protocol interaction storing user position data
- User account balance: 200 bytes
- Liquidation queue entry: 150 bytes
- Oracle price cache: 500 bytes
- Transaction history entry: 300 bytes

**Calculation:**
- Total bytes: 200 + 150 + 500 + 300 = 1,150 bytes
- Price per byte: 0.000001 Ⓝ
- Total: 1,150 × 0.000001 = **0.00115 Ⓝ**

## Storage Deposit Patterns

### Pattern 1: Attaching Storage Deposit to Function Calls

```rust
use near_sdk::{near_bindgen, require, env};

#[near_bindgen]
pub fn create_user_profile(name: String, bio: String) -> Result<(), String> {
    let storage_required = (name.len() + bio.len()) as u128;
    let storage_cost = storage_required * 1_000_000_000_000_000; // Convert to yoctoNEAR
    
    require!(
        env::attached_deposit() >= storage_cost,
        format!("Storage deposit required: {} yoctoNEAR", storage_cost)
    );
    
    // Store profile...
    Ok(())
}
```

### Pattern 2: Validating Sufficient Attachment

```rust
#[near_bindgen]
pub fn register_token(&mut self, token_id: AccountId) -> Result<(), String> {
    const STORAGE_REQUIRED: u128 = 350_000_000_000_000; // 350 bytes in yoctoNEAR
    
    if env::attached_deposit() < STORAGE_REQUIRED {
        return Err(format!(
            "Insufficient storage deposit. Required: {} yoctoNEAR, Attached: {} yoctoNEAR",
            STORAGE_REQUIRED,
            env::attached_deposit()
        ));
    }
    
    // Register token...
    Ok(())
}
```

### Pattern 3: Storage Refund Handling

```rust
#[near_bindgen]
pub fn delete_profile(&mut self) -> Promise {
    let profile = self.profiles.remove(&env::signer_account_id()).unwrap();
    let storage_freed = (profile.name.len() + profile.bio.len()) as u128;
    let refund_amount = storage_freed * 1_000_000_000_000_000;
    
    Promise::new(env::signer_account_id())
        .transfer(refund_amount)
}
```

### Pattern 4: Cross-Contract Storage Transfer

```rust
use near_sdk::promise_batch_action_transfer;

#[near_bindgen]
pub fn transfer_with_storage(&mut self, receiver: AccountId, amount: U128) -> Promise {
    const STORAGE_BUFFER: u128 = 500_000_000_000_000; // 500 bytes buffer
    
    Promise::new(receiver.clone())
        .add_access_key_with_full_access()
        .transfer(amount.0 + STORAGE_BUFFER)
}
```

## Contract Optimization Strategies

### Lazy Initialization of State

Avoid initializing large collections at contract deployment. Instead, create them on first access:

```rust
#[near_bindgen]
pub fn get_or_create_user_data(&mut self, user: AccountId) -> UserData {
    if !self.user_data.contains_key(&user) {
        self.user_data.insert(user.clone(), UserData::default());
    }
    self.user_data.get(&user).unwrap()
}
```

This saves storage costs for unused accounts from day one.

### Strategic Collection Selection

- **Vector**: Use for ordered, indexed data (minimal overhead)
- **UnorderedMap**: Use for key-value lookups (slightly higher overhead than hashmap but serializable)
- **LookupMap**: Use for large datasets (minimal serialization cost, only touched keys are loaded)

### Data Serialization Format Impact

**JSON serialization** produces ~30% more bytes than **Borsh binary serialization**. For storage-sensitive contracts, use Borsh:

```rust
#[derive(BorshSerialize, BorshDeserialize, Debug)]
pub struct CompactData {
    pub balance: u128,
    pub timestamp: u64,
}

// Estimated sizes:
// JSON: ~50 bytes
// Borsh: ~24 bytes (52% reduction)
```

### State Versioning for Migrations

Design contracts with versioning to allow efficient migrations:

```rust
#[near_bindgen]
pub struct Contract {
    version: u8,
    state_v1: Option<StateV1>,
    state_v2: Option<StateV2>,
}

pub fn migrate_v1_to_v2(&mut self) {
    if let Some(v1) = self.state_v1.take() {
        // Convert V1 to V2, accounting for storage differences
        self.state_v2 = Some(StateV2::from(v1));
    }
}
```

### Trie Node Packing Efficiency

Store related data contiguously to maximize trie node utilization. Instead of:

```rust
self.user_balances.insert(user.clone(), balance);
self.user_stakes.insert(user.clone(), stake);
```

Consider a combined struct:

```rust
#[derive(BorshSerialize, BorshDeserialize)]
pub struct UserPosition {
    balance: u128,
    stake: u128,
}
self.user_positions.insert(user, UserPosition { balance, stake });
```

## Common Mistakes and How to Avoid Them

**Mistake 1: Forgetting Storage Calculations**
- ❌ Assuming storage is "free" like traditional blockchains
- ✅ Always calculate expected storage and attach appropriate deposits

**Mistake 2: Exceeding Attached Deposit**
- ❌ Allowing unlimited data writes without deposit validation
- ✅ Check remaining attached deposit before operations: `env::attached_deposit() - used`

**Mistake 3: Not Accounting for Serialization Overhead**
- ❌ Estimating 100 bytes but serializing to 130 bytes
- ✅ Test actual serialization with `to_vec()` before deployment

**Mistake 4: Ignoring Collection Overhead**
- ❌ Using UnorderedMap for every small lookup
- ✅ Profile collection sizes; use LookupMap for large datasets

**Mistake 5: Leaving Orphaned Data**
- ❌ Never clearing unused state, accumulating locked deposits
- ✅ Implement cleanup functions and refund mechanisms

## Best Practices Summary

1. **Always calculate storage needs** before writing to state
2. **Use Borsh serialization** to minimize byte overhead
3. **Implement deposit validation** in state-modifying functions
4. **Provide refund mechanisms** when users delete data
5. **Test storage costs** in testnet with actual contract code
6. **Monitor account balance** to ensure sufficient storage reserves
7. **Profile your contracts** to identify optimization opportunities

## Conclusion

Storage staking in NEAR creates a sustainable, economically-aligned system for managing blockchain state. By understanding cost calculations, implementing proper deposit patterns, and optimizing contract design, you can build efficient dApps that minimize user costs while maintaining state integrity. Start with careful calculations, test thoroughly on testnet, and progressively optimize based on real usage patterns.