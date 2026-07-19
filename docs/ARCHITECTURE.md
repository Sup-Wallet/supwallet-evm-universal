# SupWallet (EVM) — Architecture, System Design, UX

An AI-agent wallet on **Arbitrum One**. Sup turns natural language into typed actions, but custody and
execution are bounded by the shipped onchain **SupVault + adaptor** model. Particle Universal Account
(EIP-7702) is the cross-chain funding and payout ramp; the vault itself is Arbitrum-only.

---

## 1. The one-paragraph mental model

Each owner has an isolated `SupVault` clone. The owner is the only party that can deposit or withdraw.
The agent can only call `execute(...)` or `executeBatch(...)` on that vault, and only through adaptors
that pass two gates: the adaptor address is curated in `AdaptorRegistry`, and the owner has granted that
adaptor a token allowance. The vault transfers **exactly** the authorized `amountIn`, calls the adaptor
with `call` (never `delegatecall`), then checks that the adaptor overspent nothing, retained nothing,
and returned the declared result to the vault. If any invariant fails, the transaction reverts.

---

## 2. Current custody model

| Rail | What it's for | Where funds live | Hard bound | Signature shape |
|---|---|---|---|---|
| **SupVault** | Agent payments and swaps through approved adaptors | Per-owner vault clone on Arbitrum | Per-`(adaptor, token)` cap + runtime post-conditions | Owner grants once; agent runs within caps |
| **Particle UA** | Cross-chain funding into the vault and payout from Arbitrum | User's UA / same EOA address | User-signed UA transaction + Particle route checks | User signs cross-chain actions |
| **Optional private float** | Private autonomous strategy research | Separate shielded float | Float size | Planned / advanced only |

Retired: the `AllowanceRouter` ERC-20 approve rail, Privy TEE policy / scoped signer /
`SessionPolicy`, ZeroDev, and "Tier-1 allowance card" language. `AllowanceRouter.sol` remains in the
contracts tree as unused reference-only code.

### Live Arbitrum One deployments

| Contract | Address | Role |
|---|---|---|
| `SupVaultFactory` | `0x7d75152E46048941E9B3a1e463f122B621833136` | EIP-1167 per-owner vault clones |
| `SupVault` implementation | `0xd9BeFB8160b1D10d6252Ba8c184D5e94584dbf38` | Vault logic |
| `AdaptorRegistry` | `0x741E7C55689c1B2B615ddB1520bA485B4Fb332d9` | Curated execution allowlist |
| `PayAdaptor` | `0x78c0a7bb3a666E83110cCFE275AE701BB684A2f5` | Payment adaptor |
| `UniswapV3SwapAdaptor` | `0x5Ee5725481AF6276dDCBB03EB4767460D2C7a2Df` | Swap adaptor |
| `AaveV3SupplyAdaptor` | `0x579f03527fcA38852a6caC47b947b30Db881021A` | Supply-to-Aave adaptor |
| `AdaptorListingRegistry` | `0xedCF749749a13BC00902C057055F2207bBdAbCf8` | Open marketplace listings |

---

## 3. System components

```
                         ┌─────────────────────────────────────────────┐
  SURFACES               │  Web chat (AgentChat)   Telegram bot + /tg   │
                         └───────────────┬─────────────────────────────┘
                                         │  natural language
                         ┌───────────────▼─────────────────────────────┐
  BRAIN                  │  runAgentTurn · system prompts · tools       │
                         └───────────────┬─────────────────────────────┘
                                         │  typed action envelope
                         ┌───────────────▼─────────────────────────────┐
  MONEY-PATH FIREWALL    │  recipient provenance · intent weld ·        │
                         │  preflight · receiver guard · failure guard  │
                         └───────────────┬─────────────────────────────┘
                                         │  bounded, verified action
                         ┌───────────────▼─────────────────────────────┐
  EXECUTION              │  SupVault.execute / executeBatch on Arbitrum │
                         │  Particle UA funding / payout                │
                         └───────────────┬─────────────────────────────┘
                                         │
  STATE                  │  grants · allowances · vaults · listings ·   │
                         │  activity · strategy · optional memory       │
```

### Key modules and contracts

- **Vault contracts**: `SupVaultFactory`, `SupVault`, `AdaptorRegistry`, `AdaptorListingRegistry`,
  `IAdaptor`, `PayAdaptor`, `UniswapV3SwapAdaptor`.
- **Vault UI**: deposit/withdraw supports USDC, WETH, and native ETH. ETH deposits wrap to WETH; WETH
  withdrawals can unwrap to native ETH.
- **Particle UA**: `createTransferTransaction` handles cross-chain transfer with
  `destinationChainId`; `createUniversalTransaction` handles arbitrary cross-chain contract calls.
- **Marketplace**: open listing registry stores discovery metadata only. Curated execution registry is
  the onchain gate that `SupVault` checks.
- **AI authoring**: `/api/adaptors/author` drafts Solidity `IAdaptor` contracts bound to vault
  invariants. Drafting or publishing does not grant execution rights.

Optional, off by default:
- Walrus manifest storage behind `SUP_WALRUS_ENABLED`; manifests currently live in KV.
- Walrus Memory / MemWal behind `SUP_MEMORY_ENABLED`; the default provider is `NullProvider`.

---

## 4. SupVault execution invariants

### `execute(...)`

`execute(adaptor, inputToken, amountIn, Expectation, callData)`:

1. Requires the adaptor to be curated in `AdaptorRegistry`.
2. Requires the owner to have granted that adaptor for the input token.
3. Debits the per-`(adaptor, token)` allowance: `perTxCap`, `periodCap` rolling window, and `expiry`.
4. Transfers **exactly** `amountIn` to the adaptor.
5. Calls the adaptor with normal `call`, never `delegatecall`.
6. Verifies post-conditions: no over-spend, adaptor kept no residual input token, and the declared
   result credited to the vault by at least `minOut`.

The blast radius of a malicious approved adaptor is one authorized `amountIn`, not the vault balance.

### `executeBatch(...)`

`executeBatch(Call[])` is the EVM analogue of a Sui PTB: several adaptor calls run in one atomic
transaction. Each step keeps the same checks as `execute`, and every step's outputs are forced back to
the vault before the next step starts. That lets a later step consume an earlier step's output, while any
mid-batch revert rolls back the whole batch.

Foundry coverage includes chained steps, mid-batch rollback, per-step allowance debit, and `onlyAgent`.

---

## 5. Shipped adaptors

Only two EVM adaptors exist today:

| Adaptor | Purpose | Result expectation |
|---|---|---|
| `PayAdaptor` | Pay a recipient from the vault | `AssetKind.NONE` |
| `UniswapV3SwapAdaptor` | Swap through Uniswap V3 | Output token credited to vault, `minOut` enforced |

Uniswap LP, lending, perps, and other protocol adaptors are planned marketplace additions,
not shipped contracts.

---

## 6. UX flows

### A. Fund the vault

1. User chooses USDC, WETH, or native ETH.
2. Same-chain deposits transfer into the user's vault. Native ETH is wrapped to WETH on deposit.
3. Cross-chain funding uses Particle UA to source funds from another chain and deliver value to
   Arbitrum, then completes the vault deposit flow.

### B. Agent executes through an adaptor

1. Owner grants an adaptor/token allowance with per-tx cap, period cap, and expiry.
2. Sup prepares an adaptor call from user intent.
3. The vault executes it only if registry, owner grant, allowance, and post-conditions all pass.
4. Results stay in the vault.

### C. Withdraw

Only the owner can withdraw. USDC/WETH withdrawals transfer from the vault; WETH can be unwrapped to
native ETH on withdrawal.

### D. Marketplace

Anyone can publish adaptor discovery metadata to `AdaptorListingRegistry`. That is not executable
permission. Execution requires curation into `AdaptorRegistry` plus an owner grant in the vault.

---

## 7. Particle UA cross-chain role

Particle UA remains important, but it is not the vault:

- Same address / unified balance for the owner.
- Cross-chain transfer via `createTransferTransaction(destinationChainId)`.
- Arbitrary cross-chain contract call via `createUniversalTransaction`.
- Used as the funding and payout ramp around the Arbitrum vault.

Cross-chain work cannot give the same all-or-nothing invariant as an Arbitrum `executeBatch`, so the
vault/adaptor safety model is intentionally single-chain.

---

## 8. Trust boundaries

- **Onchain enforced:** vault custody, owner-only deposit/withdraw, curated registry check, owner grant,
  per-`(adaptor, token)` allowance, exact `amountIn`, no `delegatecall`, adaptor residual check, and
  `minOut` result credit.
- **Trusted but bounded:** curated adaptor code. A bad curated adaptor is still capped by one authorized
  `amountIn` per call and cannot directly drain the rest of the vault.
- **Discovery only:** marketplace listings and AI-authored drafts.
- **Offchain convenience:** agent brain, indexing, KV manifests, optional Walrus / memory providers.
- **Never trusted:** the LLM is never the source of custody authority.

---

## 9. Status

- **Deployed on Arbitrum One:** factory, vault implementation, curated registry, listing registry,
  `PayAdaptor`, `UniswapV3SwapAdaptor`.
- **Built/tested:** `execute`, `executeBatch`, chained batch steps, mid-batch rollback, per-step
  allowance, and `onlyAgent`.
- **UI shipped:** vault deposit/withdraw for USDC/WETH/native ETH and Particle UA cross-chain funding.
- **Planned:** Uniswap LP, lending/perps adaptors, expanded marketplace analytics and
  review flows.
