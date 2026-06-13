# Pezkuwi Token Standards (PEZ-20 & PEZ-721)

Pezkuwi defines a small, familiar family of token standards so that anyone
coming from Ethereum (ERC-20 / ERC-721), BNB Chain (BEP-20), or Tron (TRC-20)
immediately understands how tokens work on Pezkuwi:

| Standard | What it is | Backed by |
|----------|------------|-----------|
| **PEZ-20** | Fungible tokens (currencies, stablecoins, LP tokens) | `pallet-assets` on Asset Hub |
| **PEZ-721** | Non-fungible tokens (NFTs, collectibles, certificates) | `pallet-nfts` on Asset Hub |

> **Important — these are not smart-contract standards.**
> Pezkuwi Asset Hub has **no EVM and no smart-contract layer**. PEZ-20 and
> PEZ-721 are *runtime-native* standards: a token is not a deployed contract
> with an address — it is a **registered asset with an on-chain ID** managed
> directly by the runtime. This makes tokens cheaper, faster, and impossible to
> brick with buggy contract code. The interface below is the conceptual
> equivalent of ERC-20/721, mapped to native runtime calls.

---

## PEZ-20 — Fungible Token Standard

A **PEZ-20 token** is a fungible asset registered in the `Assets` pallet on
Pezkuwi Asset Hub. Each token is identified by a numeric **Asset ID** (the
equivalent of an ERC-20 contract address).

### Identity

| Concept | Ethereum (ERC-20) | Pezkuwi (PEZ-20) |
|---------|-------------------|------------------|
| Token identity | Contract address `0x…` | **Asset ID** (`u32`) |
| Where it lives | A deployed contract | The `Assets` pallet registry |
| Native coin | ETH is *not* an ERC-20 | The native gas token (`Balances`) is *not* a PEZ-20 |

> Just as ETH itself is not an ERC-20 token, the Pezkuwi native/gas token is
> **not** a PEZ-20 token — it lives in the `Balances` pallet. PEZ-20 covers all
> *issued* fungible assets.

### Required metadata

Every compliant PEZ-20 token MUST set asset metadata via `assets.set_metadata`:

| Field | Type | ERC-20 equivalent |
|-------|------|-------------------|
| `name` | bytes | `name()` |
| `symbol` | bytes | `symbol()` |
| `decimals` | `u8` | `decimals()` |

### Interface

| ERC-20 method | PEZ-20 (extrinsic / storage) | Notes |
|---------------|------------------------------|-------|
| `totalSupply()` | `Assets.Asset(id).supply` | read |
| `balanceOf(owner)` | `Assets.Account(id, owner).balance` | read |
| `transfer(to, amount)` | `assets.transfer(id, to, amount)` | also `transfer_keep_alive` |
| `approve(spender, amount)` | `assets.approve_transfer(id, spender, amount)` | allowance |
| `allowance(owner, spender)` | `Assets.Approvals(id, owner, spender)` | read |
| `transferFrom(from, to, amount)` | `assets.transfer_approved(id, from, to, amount)` | spend allowance |
| `mint` (if mintable) | `assets.mint(id, to, amount)` | issuer/admin only |
| `burn` | `assets.burn(id, who, amount)` | admin, or `assets.transfer` to a burn flow |

### Roles

PEZ-20 inherits the `pallet-assets` role model, which is richer than ERC-20:

- **Owner** — can reassign the other roles and set metadata.
- **Issuer** — may `mint`.
- **Admin** — may `burn`, `freeze`, and force transfers.
- **Freezer** — may freeze individual accounts or the whole asset.

A token MAY renounce roles (e.g. set issuer to a burn address) to make supply
fixed — the equivalent of an ERC-20 with `mint` removed.

### Events

`Assets.Transferred`, `Assets.Issued`, `Assets.Burned`, `Assets.ApprovedTransfer`,
`Assets.MetadataSet` — the audit trail equivalent of ERC-20 `Transfer` /
`Approval` events.

### Variants

| Variant | Pallet | Use case |
|---------|--------|----------|
| **PEZ-20** | `Assets` | Locally-issued fungibles (default) |
| **PEZ-20-F** (foreign) | `ForeignAssets` | Bridged / XCM assets, keyed by origin location instead of a `u32` ID |
| **PEZ-20-LP** (pool) | `PoolAssets` | Automatically-minted liquidity-provider tokens from on-chain pools |

All three share the PEZ-20 interface; they differ only in how the asset is
keyed and who can issue it.

### Compliance checklist

A token is **PEZ-20 compliant** if it:
1. is registered in `Assets` (or `ForeignAssets` / `PoolAssets`),
2. has `name`, `symbol`, and `decimals` metadata set,
3. exposes transfers and the approve / transfer-approved allowance flow,
4. publishes its role policy (mintable vs fixed supply).

---

## PEZ-721 — Non-Fungible Token Standard

A **PEZ-721 token** is a non-fungible item in the `Nfts` pallet on Pezkuwi
Asset Hub. NFTs are grouped into **collections**; each item is unique within
its collection.

### Identity

| Concept | Ethereum (ERC-721) | Pezkuwi (PEZ-721) |
|---------|--------------------|-------------------|
| Collection | Contract address | **Collection ID** (`u32`) |
| Token | `tokenId` (`uint256`) | **Item ID** (`u32`) within a collection |
| Full identity | `(contract, tokenId)` | `(collectionId, itemId)` |

### Interface

| ERC-721 method | PEZ-721 (extrinsic / storage) |
|----------------|-------------------------------|
| `ownerOf(tokenId)` | `Nfts.Item(collection, item).owner` |
| `balanceOf(owner)` | `Nfts.Account(owner, collection, item)` (iterated) |
| `mint(to, tokenId)` | `nfts.mint(collection, item, to, …)` |
| `transferFrom(from, to, tokenId)` | `nfts.transfer(collection, item, to)` |
| `approve(spender, tokenId)` | `nfts.approve_transfer(collection, item, spender, …)` |
| `tokenURI(tokenId)` | `nfts.set_metadata(collection, item, data)` → item metadata |
| collection metadata | `nfts.set_collection_metadata(collection, data)` |

### Capabilities beyond ERC-721

`pallet-nfts` gives PEZ-721 features ERC-721 needs extra contracts for:
- **Royalties / attributes** — native per-item and per-collection attributes.
- **Trading** — on-chain `set_price` / `buy_item` without a marketplace contract.
- **Locks & soulbound** — items or metadata can be made non-transferable.

### Legacy: `Uniques`

Older NFTs minted via the `Uniques` pallet are also considered PEZ-721 (legacy
profile). New collections SHOULD use `Nfts`.

---

## Naming & branding conventions

- Always written **hyphenated**: `PEZ-20`, `PEZ-721` (never `PEZ20`).
- In wallets/explorers, show a small **`PEZ-20`** / **`PEZ-721`** badge next to
  the asset, the same way wallets show `ERC-20`.
- The family is intentionally parallel to the ERC family so the mental model
  transfers for free: `ERC-20 → PEZ-20`, `ERC-721 → PEZ-721`.
- Reserved for future use: **PEZ-1155** (multi-token / semi-fungible) if a
  combined fungible+NFT pallet is adopted. Not active today.

## Worked example

A community token "Bereketli Points" issued on Asset Hub:
- Registered in `Assets` with Asset ID `N`, `symbol: BRP`, `decimals: 2`.
- Issuer set to the program treasury; admin can freeze on abuse.
- Wallets display it as **BRP · PEZ-20**.
- Transfers use `assets.transfer(N, …)`; allowances use
  `assets.approve_transfer` / `assets.transfer_approved`.

---

*This document defines a naming and interface convention only. It introduces no
new runtime code and requires no chain upgrade — PEZ-20 and PEZ-721 describe the
`pallet-assets` and `pallet-nfts` interfaces that already exist on Pezkuwi Asset
Hub.*
