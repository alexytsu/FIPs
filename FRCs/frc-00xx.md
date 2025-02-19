---
fip: 00xx
title: Fungible token standard
author: Alex Su (@alexytsu), Andrew Bright (@abright), Alex North (@anorth)
discussions-to: https://github.com/filecoin-project/FIPs/discussions/492
status: Draft
type: FRC
created: 2022-14-11
---

# Non-Fungible Token Standard

## Simple Summary

A standard interface for native actor non-fungible tokens (NFTs).

## Abstract

This proposal provides a standard API for the implementation of non-fungible
tokens (NFTs) as FVM native actors. The proposal learns from NFT standards
developed for other blockchain ecosystems, being heavily inspired by
[ERC-721](https://eips.ethereum.org/EIPS/eip-721). However, as a design goal,
this proposal aims to complement the existing fungible token interface described
in [FRC-0046](https://github.com/filecoin-project/FIPs/pull/435/files). As such
it brings along equivalent specifications for:

- Standard token/name/symbol/supply/balance queries
- Standard allowance protocol for delegated control
- Mandatory universal receiver hooks for incoming tokens

The interface has been designed with gas-optimisations in mind and hence methods
support batch-operations where practical.

## Change Motivation

The concept of a non-fungible token is widely established in other blockchains.
As on other blockchains, a complementary NFT standard to FRC-0046 allows the
ownership of uniquely identifiable assets to be tracked on-chain. The Filecoin
ecosystem will benefit from a standard API implemented by these actors. A
standard permits easy building of UIs, wallets, tools, and higher level
applications on top of a variety of tokens representing different assets.

## Specification

Methods and types are described with a Rust-like pseudocode. All parameters and
return types are IPLD types in a CBOR tuple encoding.

Methods are to be dispatched according to the calling convention described in
[FRC-0042](https://github.com/filecoin-project/FIPs/blob/master/FRCs/frc-0042.md).

### Interface

An actor implementing a FRC-00XX token must provide the following methods.

```rust
type TokenID = u64;

/// Multiple token IDs are represented as a BitField encoded with RLE+, the index of each set bit
/// corresponds to a TokenID
type TokenList = BitField;

/// Multiple Actor IDs are represented as a BitField encoded with RLE+, the index of each set bit
/// corresponds to an f0 address expressed as an ActorID
type ActorList = BitField;

/// A descriptive name for the collection of NFTs in this actor
fn Name() -> String

/// An abbreviated (ticker) name for the NFT collection
fn Symbol() -> String

/// Returns the total number of tokens in this collection
/// Must be non-negative
/// Must equal the balance of all adresses
/// Should equal the sum of all minted tokens less tokens burnt
fn TotalSupply() -> u64

/// Returns a URI that resolves to the metadata for a particular token
fn MetadataID(tokenID: TokenID) -> String

/// Returns the balance of an address which is the number of unique NFTs held
/// Must be non-negative
fn BalanceOf(owner: Address) -> u64

/// Returns a parallel list of f0 ID-addresses that own the specified tokens
fn OwnerOf(tokenIDs: TokenList) -> Vec<ActorID>

/// Transfers tokens from caller to the specified address
/// Each token specified must exist and be owned by the caller
/// The entire batch of transfers aborts if any of the specified tokens does not meet the criteria
/// Transferring to the caller must be treated as a normal transfer
/// Returns the resulting balances for the from and to addresses
/// The operatorData is passed through to the receiver hook directly
///
/// Tokens that were successfully transferred are returned along with the new balances of the sending and receiving addresses
fn Transfer({to: Address, tokenIDs: TokenList, operatorData: Bytes})
  -> {fromBalance: u64, toBalance: u64}

/// Transfers tokens to the specified address with the caller acting as an operator
/// Each token specified must exist and the caller must be authorised to transfer it. The caller is considered authorised by being an approved operator on the token-id or by being an approved operator on the account that owns the token.
/// The entire batch of transfers aborts if any of the specified tokens does not meet the above criteria
/// Transferring to the owner must be treated as a normal transfer
/// Returns the resulting balance for the to address
/// The operatorData is passed through to the receiver hook directly
///
/// Returns tokens that were successfully transferred are returned along with the new balance of the receiving address
fn TransferFrom({to: Address, from: Address, tokenIDs: TokenList, operatorData: Bytes})
  -> {toBalance: u64}

/// Authorizes an address as the specified operator for a set of tokens
/// The caller must own all the specified tokens, or be an account-level operator for every token specified. If not, the entire batch will fail to be approved.
fn Approve({operator: Address, tokenIDs: TokenList}) -> ()

/// Revokes an address as an authorized operator for a set of tokens
/// The caller must own all the specified tokens, or be an account-level operator for every token specified. If not, the entire batch will fail to be revoked. If the specified operator is not currently an approved operator for some tokens in the list, the method can still succeed but will be a no-op on those tokens.
fn RevokeApproval({operator: Address, tokenIDs: TokenList}) -> ()

/// Returns whether an address is an approved operator for a particular set of tokens
/// The owner of a token is not implicitly considered as a valid operator of that token
/// The return value is encoded as a bitfield with the marked bits corresponding to the indices of approved tokens if tokenIDs were interpreted as a vector of TokenIDs
fn IsApprovedFor({operator: Address, tokenIDs: TokenList}) -> BitField

/// Authorizes the specified address as an operator for any token (including future tokens) that are owned by the caller's address
fn ApproveForAll({operator: Address}) -> ()

/// Returns whether an address is an authorized operator for another address
/// Addresses are not reflexive operators on themselves
fn IsApprovedForAll({operator: Address, owner: Address}) -> bool

/// Revokes an address as an authorized operator for the calling address
fn RevokeApprovalForAll({operator: Address}) -> ()

/// Burns the specified tokens where the caller is the owner
/// The entire burn operation aborts if any of the tokens is not owned by the caller
///
/// Returns the resulting balance of the caller's address
fn Burn({tokenIDs: TokenList}) -> u64;

/// Burns the specified tokens where the caller is an approved operator
/// The entire burn operation aborts if the caller is not an approved operator on any of the tokens
fn BurnFrom({from: Address, tokenIDs: TokenList}) -> ();
```

**Optional Extension - Enumerable TokenIDs**

An actor may choose to implement a method to retrieve the list of all
circulating NFTs.

```rust
/// Enumerates some number of circulating Token IDs
/// The method must return all the IDs for tokens in circulation in the range [rangeStart, next)
/// The method must return next = 0 if the entire range of TokenIDs has been exhausted
fn ListTokens(rangeStart: TokenID) -> {tokens: TokenList, next: TokenID}
```

When listing NFTs the actor must return the next `count` number of TokenIDs in
ascending order if exist. The returned IDs should start from and include
`tokenRangeStart` if it exists or the next lowest Token ID if it does not. When
a TokenList is returned with less than `count` IDs specified, the caller can
assume there are no more IDs to enumerate.

**Optional Extension - Enumerable Operators**

An actor may choose to implement any number of the following methods to support
enumeration of operators.

```rust
/// Returns all the token-level operators that are currently approved
fn TokenOperatorsFor(tokenID: TokenID) -> ActorList

/// Returns all the account-level operators that are currently approved
fn AccountOperatorsFor(actorID: ActorID) -> ActorList
```

### Receiver Interface

An actor must implement a receiver hook to receive NFTs. The receiver hook is
defined in FRC-0046 and must not abort when handling an incoming token. When
transferring batch of tokens, the receiver hook is invoked once meaning the
entire set of NFTs is either accepted or rejected (by aborting).

```rust
/// Type of the payload accompanying the receiver hook for a FRCXX NFT.
struct FRCXXTokensReceived {
    // The tokens being transferred
    tokenIDs: TokenList,
    // The address to which tokens were credited (which will match the hook receiver)
    to: Address,
    // The address from which tokens were debited
    from: Address,
    // The actor which initiated the mint or transfer
    operator: Address,
    // Arbitrary data provided by the operator when initiating the transfer
    operatorData: Bytes,
    // Arbitrary data provided by the token actor
    tokenData: Bytes,
}

/// Receiver hook type value for an FRC-00XX token
const FRCXXTokenType = frc42_hash("FRCXX")

// Invoked by a NFT actor after transfer of tokens to the receiver’s address.
// The NFT collection state must be persisted such that the receiver will observe the new balances.
// To reject the transfer, this hook must abort.
fn Receive({type: uint32, payload: byte[]})
```

### Behaviour

**Universal receiver hook**

The NFT collection must invoke the receiver hook method on the receiving address
whenever it credits tokens. The `type` parameter must be `FRCXXTokenType` and
the payload must be the IPLD-CBOR serialized `FRCXXTokensReceived` structure.

The attempted credit is only persisted if the receiver hook is implemented and
does not abort. A mint or transfer operation should abort if the receiver hook
does, or in any case must not credit tokens to that address.

**Minting**

API methods for minting are left unspecified. A newly minted token cannot have
the same ID as an existing token or a token that was previously burned. Minting
must invoke the receiver hook on the receiving address and fail if it aborts.

**Transfers**

Empty transfers are allowed, including when the `from` address has zero balance.
An empty transfer must invoke the receiver hook of the `to` address and abort if
the hook aborts. An empty transfer can thus be used to send messages between
actors in the context of a specific NFT collection.

**Operators**

Operators can be approved at two separate levels.

_Token level operators_ are approved by the owner of the token via the `Approve`
method. If an account is an operator on a token, it is permitted to debit (burn
or transfer) that token. An NFT can have many operators at a time, but
transferring the token will revoke approval on all its existing operators.

_Account level operators_ are approved via the `ApproveForAll` method. If an
owner approves an operator at the account level, that operator has permission to
debit any token belonging to the owner's account. This includes tokens that are
not yet owned by the account at the time of approval.

**Addresses**

Addresses for receivers and operators must be resolvable to an actor ID.
Balances must only be credited to an actor ID. All token methods must attempt to
resolve addresses provided in parameters to actor IDs. A token should attempt to
initialise an account for any address which cannot be resolved by sending a
zero-value transfer of the native token to the address.

Note that this means that an uninitialized actor-type (f2) address cannot
receive tokens or be authorized as an operator. Future changes to the FVM may
permit initialization of such addresses by sending a message to them, in which
case they should automatically become functional for this standard.

**Extensions**

An NFT collection may implement other methods for transferring tokens and
managing operators. These must maintain the invariants about supply and
balances, and invoke the receiver hook when crediting tokens.

An NFT collection may implement restrictions on allowances and transfer of
tokens.

## Design Rationale

### Batching

Methods on this interface accept a list of `TokenID`s. When a list of IDs is
specified, the method must succeed or fail atomically. This simplifies retry
logic for actors when methods fail and encourages further inspection of the
intended parameters when human errors may have been made.

### Synergy with fungible tokens

In order for higher synergy with the existing FRC-0046 fungible token standard,
this proposal aims for a conceptually similar interface in terms of balances,
supply and operators. For the same safety reasons described in FRC-0046, this
token standard requires a universal receiver on actors that wish to hold tokens.

This allows easier interactions between fungible tokens and NFTs and opens
possibilities for composition of the the two standards to represent different
structures of ownership such as fractionalized NFTs or semi-fungible tokens.
Instead of encoding such semantics directly into the standard, a minimal
interface is proposed instead to make the primary simple use cases more
efficient and straightforward.

### Transfers

There is no technical need to separate the `Transfer` and `TransferFrom`
methods. A transfer method could function given only the list of token ids that
the caller wishes to transfer and for each token asserts that:

- The caller is the owner of the token OR
- The caller is an approved operator on the token OR
- The caller is an approved operator on the account that owns the token

However, the authors judge that the benefit of aligning with existing
conventions (FRC-46, ERC-721 etc.) and having separate flows for operators and
token owners will reduce risk of mistakes and unexpected behaviour.

### Metadata

This proposal does not specify a structure or format of metadata. It is
encouraged that what is stored on chain is an `ipfs://` link that can be
resolved to the metadata.

## Backwards Compatability

There are no implementatons of NFT collections yet on Filecoin.

## Test Cases

Extensive test cases are present in the implementation of this proposal at
https://github.com/helix-onchain/filecoin/tree/main/frcxx_nft.

## Security Considerations

### Reentrancy

Receiver hooks introduce the possibility of complex call flows, the most
concerning of which might be a malicious receiver that calls back to a token
actor in an attempt to exploit a re-entrancy bug. We expect that one or more
high quality reference implementations of the token state and logic will keep
the vast majority of NFT actors safe. We judge the risk to more complex actors
as lesser than the aggregate risk of losses due to misdirected transfers.

## Incentive Considerations

N/A

## Product Considerations

??

## Implementation

An implementation of this standard is in development at
https://github.com/helix-onchain/filecoin/tree/main/frcxx_nft.

## Copyright

Copyright and related rights waived via
[CC0](https://creativecommons.org/publicdomain/zero/1.0/).
