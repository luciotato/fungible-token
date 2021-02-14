# Fungible Token

The new, official Fungible Token Standard (nep-141).

The Fungible Token is composed of the following standards: Core Fungible Token, Fungible Token Metadata and Contract Storage. We also define an optional extension: Account Registration.

```rust
trait FungibleToken: CoreFungibleToken + FTMetadata + ContractStorage {}

// Extensions
trait AccountRegistration
trait MintableFungibleToken
```

The design goals:

+ mot trading off simplicity - the contract must be easy to implement and use;
+ completely remove allowance: removes UX flaws and optimize contract storage space;
+ simplify interaction with other smart-contracts;
+ simplify flow in NEP-122 (Allowance-free vault-based token standard);
+ remove frictions related to different decimals;
+ enable smart contract composability through token transfers.

Our work is mostly influenced by the aforementioned ERC-223 and NEP-122.

Below we provide only the highlights. Follow the links for detailed description.

## Core Fungible Token

+ The design [discussion](https://github.com/near/NEPs/issues/141).
+ [Details](./core_fungible_token.md)

The Core Fungible Token abstracts the account and meta-information management and focuses on

```rust
trait CoreFungibleToken {
    /// Simple transfer without callback.
    #[payable]
    fn ft_transfer(
        &mut self,
        receiver_id: ValidAccountId,
        amount: U128,
        memo: Option<String>
    );

    /// Transfer to a contract with a callback.
    #[payable]
    fn ft_transfer_call(
        &mut self,
        receiver_id: ValidAccountId,
        amount: U128,
        memo: Option<String>,
        msg: String,
    ) -> Promise;

    /// Callback to resolve transfer.
    /// Private method (`env::predecessor_account_id == env::current_account_id`).
    /// after the receiver handles the transfer call and returns value of used amount in `U128`.
    #[private]
    fn ft_resolve_transfer(
        &mut self,
        sender_id: ValidAccountId,
        receiver_id: ValidAccountId,
        amount: U128,
        #[callback_result] used_amount: CallbackResult<U128>,  // NOTE: this interface is not supported yet and has to
                                                               // be handled using lower level interface.
    ) -> U128;
}
```

The receiver contract should implement the following Trait:

```rust
/// Receiver of the Fungible Token for `transfer_call` calls.
pub trait FungibleTokenReceiver {
    /// Callback to receive tokens.
    /// Called by fungible token contract `env::predecessor_account_id` after `transfer_call` was initiated by
    /// `sender_id` of the given `amount` with the transfer message given in `msg` field.
    fn ft_on_transfer(
        &mut self,
        sender_id: ValidAccountId,
        amount: U128,
        msg: String,
    ) -> PromiseOrValue<U128>;
}
```

## FT Metadata

+ The design [discussion](https://github.com/near/NEPs/discussions/148).
+ [Details](./ft_metadata.md)


Many smart-contracts need to carry extra information data. Notably, token smart contract should provide minimum information to a user about it's usage. Fungible Token Metadata is a set of minimum attributes a Fungible Token has to provide to assure correct operations.

```rust

trait FTMetadata {
    fn metadata(&self) -> Metadata
}

// recommended function
trait UpdateableFTMetadata {
    fn metadata_update(&mut self, reference: String, reference_hash: String)
}

struct Metadata {
    /// Canonical token name.
    name: String,

    /// A JSON document or an URL to a JSON document with resources about the token.
    reference: String,

    /// A blake2b hash of the resource pointed by `reference`. It's used to validation that
    /// the reference content didn't change.
    reference_hash: vec[u8],

    /// Returns the number of decimals the token uses - e.g. 8, means to divide the token
    /// amount by 100000000 to get its user representation.
    decimals: uint8,
}
```


## Contract Storage

+ The design [discussion](https://github.com/near/NEPs/discussions/145).
+ [Details](./contract_storage.md)


Many smart contract will store new data for every user. Fungible Tokens will need to store balance information per user. Hence we need a way to manage a storage costs per user.

```rust
trait ContractStorage {
    /// Deposit funds to pay for account storage for the specified `account_id`.
    /// If the `account_id` is not specified, then it defaults to the `predecessor_id`.
    ///
    /// Returns the account's updated balance
    #[payable]
    fn storage_deposit(account_id: string|null): AccountStorageBalance;

    /// Withdraw the specified amount from the `predecessor_id` available storage balance.
    /// If no `amount` is specified, then all of the account's available storage balance is withdrawn.
    ///
    /// Returns the account's updated balance
    #[payable]
    fn storage_withdraw(amount: U128|null): AccountStorageBalance;

    /// Returns a minimum balance required for handling a user.
    fn storage_minimum_balance(): U128;

    fn storage_balance_of(account_id: string): AccountStorageBalance;

    struct AccountStorageBalance {
        total: U128;
        available: U128;
    }
}
```
