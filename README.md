# NEAR Storage Staking Guide

## Introduction

NEAR Protocol uses a unique storage staking mechanism to ensure sustainable blockchain growth. Unlike traditional models where storage is "rent-free," NEAR requires accounts to maintain a minimum balance to cover the storage they consume on-chain. This guide provides developers with comprehensive knowledge about storage staking, cost calculations, and optimization strategies.

## 1. Storage Staking Fundamentals

### What is Storage Staking?

Storage staking is NEAR's economic model for managing blockchain state growth. Every byte of data stored on-chain requires backing by a corresponding amount of NEAR tokens locked in the account. This approach incentivizes efficient data management and prevents blockchain bloat.

### Key Concepts

**Storage Measurement**: Storage is measured in bytes, encompassing:
- Account metadata (keys, nonce, code hash)
- Smart contract state
- Account balances
- Delegated validator stakes

**Account Balance Relationship**: An account's available balance = total balance - storage stake required

```
Available Balance = Total Balance - (Storage Used × Cost Per Byte)
```

**State Rent vs. Storage Staking**: 

NEAR originally planned state rent (charging rent on unused data), but switched to storage staking (locking capital for storage). This change reduced complexity and provided predictability—once you stake for storage, the cost never increases.

### Account Overhead

Every NEAR account has a base overhead of **182 bytes**, regardless of stored data. This covers:
- Account ID (variable length)
- Account metadata
- Access keys

## 2. Cost Calculation Methodology

### Storage Cost Formula

```
Storage Cost (NEAR) = Storage Used (bytes) × Cost Per Byte
```

### Current Cost Per Byte

As of 2024, NEAR's cost per byte is:
```
Cost Per Byte = 0.00001 NEAR = 10^-5 NEAR
```

To convert to yoctoNEAR (smallest NEAR unit):
```
Cost Per Byte = 10,000 yoctoNEAR
```

### Calculating Total Storage Usage

1. **Base account overhead**: 182 bytes
2. **Contract code size**: Serialized WASM binary
3. **State data**: Serialized JSON-like structures

### Worked Examples

#### Example 1: Simple Account with No Contract

```
Storage breakdown:
- Account ID: ~20 bytes (e.g., "alice.near")
- Account metadata: ~100 bytes
- One access key: ~62 bytes
- Total: ~182 bytes (base overhead)

Cost calculation:
Storage Cost = 182 bytes × 10^-5 NEAR
Storage Cost = 0.00182 NEAR
```

#### Example 2: Fungible Token Contract with 1,000 Holders

```
Storage breakdown:
- Contract WASM code: ~50 KB = 51,200 bytes
- Contract state (metadata): ~500 bytes
  * Token name, symbol, decimals
  * Owner account
- Token balances (1,000 holders): 1,000 × 48 bytes = 48,000 bytes
  * Each holder: account ID hash (32 bytes) + balance (16 bytes)
- Total: 51,200 + 500 + 48,000 = 99,700 bytes

Cost calculation:
Storage Cost = 99,700 bytes × 10^-5 NEAR
Storage Cost = 0.997 NEAR ≈ 1 NEAR
```

#### Example 3: NFT Minting Operation

```
Scenario: Minting an NFT with metadata

Storage added per NFT:
- Token ID (string): ~20 bytes
- Owner account reference: ~32 bytes
- Metadata (JSON): ~200 bytes
  * title, description, image URL, etc.
- Total per NFT: ~252 bytes

Cost per NFT minted:
Storage Cost = 252 bytes × 10^-5 NEAR
Storage Cost = 0.00252 NEAR per token

For 100 NFTs:
Total Cost = 0.252 NEAR

For 10,000 NFTs:
Total Cost = 25.2 NEAR
```

## 3. Storage Deposit Patterns

### Pattern 1: Storage Deposit Function

Most NEAR contracts implement a `storage_deposit()` function allowing users to pre-pay for storage:

```rust
#[payable]
pub fn storage_deposit(&mut self, account_id: Option<AccountId>) -> StorageBalance {
    let account_id = account_id.unwrap_or_else(env::predecessor_account_id);
    let account_storage = self.accounts.get(&account_id).unwrap_or_default();
    let storage_cost = account_storage.storage_usage as u128 * STORAGE_COST_PER_BYTE;
    
    let attached_deposit = env::attached_deposit();
    assert!(attached_deposit >= storage_cost, "Insufficient deposit");
    
    self.accounts.insert(&account_id, &account_storage);
    
    StorageBalance {
        total: storage_cost,
        available: attached_deposit - storage_cost,
    }
}
```

### Pattern 2: Checking Storage Balance

Query function to determine storage requirements:

```rust
pub fn storage_balance_of(&self, account_id: AccountId) -> Option<StorageBalance> {
    self.accounts.get(&account_id).map(|account| {
        let storage_cost = account.storage_usage as u128 * STORAGE_COST_PER_BYTE;
        StorageBalance {
            total: storage_cost,
            available: account.balance - storage_cost,
        }
    })
}
```

### Pattern 3: Storage Unregister and Refunds

Return unused storage deposits:

```rust
pub fn storage_unregister(&mut self, force: Option<bool>) -> bool {
    let account_id = env::predecessor_account_id();
    if let Some(account) = self.accounts.remove(&account_id) {
        let storage_cost = account.storage_usage as u128 * STORAGE_COST_PER_BYTE;
        Promise::new(account_id).transfer(account.balance);
        true
    } else {
        false
    }
}
```

## 4. Contract Optimization Strategies

### Strategy 1: State Compression and Packing

**Use compact data structures**:

```rust
// ❌ Inefficient: Wastes space with enum overhead
#[derive(BorshSerialize, BorshDeserialize)]
pub enum TokenType {
    Fungible(String),
    NonFungible(String),
    Semi(String),
}

// ✅ Efficient: Use discriminant-only enum
#[derive(BorshSerialize, BorshDeserialize)]
pub enum TokenType {
    Fungible = 0,
    NonFungible = 1,
    Semi = 2,
}
```

### Strategy 2: Lazy Initialization

Defer state creation until needed:

```rust
pub fn lazy_init_metadata(&mut self) {
    // Only create metadata when first accessed
    if !self.metadata_initialized {
        self.metadata = Metadata::new();
        self.metadata_initialized = true;
    }
}
```

### Strategy 3: Efficient Data Structure Selection

**HashMap vs. Vector Trade-offs**:

- Use **HashMap** when: Frequent lookups by key, sparse data
- Use **Vector** when: Sequential iteration, dense data

```rust
// ❌ Bad for 10,000 items with sparse IDs
pub fn get_balance_vector(balances: &Vec<(AccountId, Balance)>, id: &AccountId) -> Balance {
    balances.iter().find(|(a, _)| a == id).map(|(_, b)| *b).unwrap_or(0)
    // O(n) lookup!
}

// ✅ Good for 10,000 items with sparse IDs
pub fn get_balance_hashmap(balances: &HashMap<AccountId, Balance>, id: &AccountId) -> Balance {
    balances.get(id).copied().unwrap_or(0)
    // O(1) lookup
}
```

### Strategy 4: Pagination for Large Datasets

```rust
pub fn get_token_holders(&self, from_index: u64, limit: u64) -> Vec<(AccountId, Balance)> {
    self.balances
        .iter()
        .skip(from_index as usize)
        .take(limit as usize)
        .map(|(id, balance)| (id.clone(), balance.clone()))
        .collect()
}
```

### Strategy 5: Archival Patterns

Move historical data off-chain:

```rust
pub fn archive_completed_orders(&mut self) {
    let current_block = env::block_index();
    let cutoff_block = current_block - 1_000_000;
    
    // Delete old orders from state
    self.orders.retain(|order| order.completion_block > cutoff_block);
}
```

## 5. Practical Optimization Checklist

- ✅ Profile storage usage with `env::storage_usage()`
- ✅ Use `UnorderedMap` for sparse collections instead of `Vector`
- ✅ Implement `storage_deposit()` pattern for user-funded features
- ✅ Archive or compress old data regularly
- ✅ Test storage costs in testnet before mainnet deployment
- ✅ Document storage overhead in contract specifications
- ✅ Monitor state growth with NEAR Explorer or custom indexing

## 6. Best Practices Summary

1. **Always calculate storage costs** before deploying contracts
2. **Use NEAR CLI** to estimate costs: `near deploy`
3. **Implement storage deposit patterns** for user interactions
4. **Monitor contract state growth** over time
5. **Optimize data structures** during development, not after
6. **Test thoroughly** on testnet with realistic data volumes
7. **Document storage assumptions** in your contract README

## Conclusion

Understanding NEAR's storage staking mechanism is essential for building efficient, cost-effective smart contracts. By mastering cost calculations, implementing proper deposit patterns, and applying optimization strategies, developers can build scalable applications that maximize value while minimizing storage expenses. Start with simple calculations, move to complex scenarios, and always test on testnet before mainnet deployment.