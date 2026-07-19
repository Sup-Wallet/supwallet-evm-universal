# SupWallet (EVM) — Architecture, System Design, UX

An AI-agent wallet on **Arbitrum One**. Sup turns natural language into typed actions, but custody and
execution are bounded by the shipped onchain **SupVault + adaptor** model. Particle Universal Account
(EIP-7702) is the cross-chain funding and payout ramp; the vault itself is Arbitrum-only.

**Live:** app → https://supwallet-evm-universal.vercel.app · docs → https://supwallet-evm-universal.vercel.app/docs
· public repo → https://github.com/Sup-Wallet/supwallet-evm-universal

---

## 1. The one-paragraph mental model

Each owner has an isolated `SupVault` clone. The owner is the only party that can deposit or withdraw.
The agent can only call `execute(...)` or `executeBatch(...)` on that vault, and only through adaptors
that pass two gates: the adaptor address is curated in `AdaptorRegistry`, and the owner has granted that
adaptor a token allowance. The vault transfers **exactly** the authorized `amountIn`, calls the adaptor
with `call` (never `delegatecall`), then checks that the adaptor overspent nothing, retained nothing,
and returned the declared result to the vault. If any invariant fails, the transaction reverts.

The result: the agent gets real autonomy — natural-language DeFi, cross-chain funding, multi-step
strategies — while the principal is **structurally** unreachable to it. Custody safety is a property of
the contract, not a promise about the backend or the model.

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
| `AaveV3WithdrawAdaptor` | `0x32cC0133063C613A21C2246c208C43710f90c637` | Redeem-from-Aave adaptor |
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
  `IAdaptor`, `PayAdaptor`, `UniswapV3SwapAdaptor`, `AaveV3SupplyAdaptor`, `AaveV3WithdrawAdaptor`.
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

## 4. SupVault: execution invariants and why the flow is safe

### 4.1 The vault data model

A vault is an **EIP-1167 minimal-proxy clone** per user; `initialize(owner, agent, registry)` sets the
owner once. It stores one `Allowance` per `(adaptor, token)` pair:

```solidity
struct Allowance {
    uint256 perTxCap;      // max per single execute (token base units)
    uint256 periodCap;     // max total per rolling window
    uint256 periodLength;  // window length (seconds)
    uint256 validUntil;    // hard expiry (unix seconds)
    uint256 windowStart;   // rolling-window bookkeeping
    uint256 spentInWindow;
    bool    granted;       // owner opted this (adaptor, token) in
}
```

The caller also declares what the vault should *receive* so the vault can verify results generically:

```solidity
enum AssetKind { ERC20, ERC721, NONE }
struct Expectation { address asset; uint256 minOut; AssetKind kind; }
```

`kind = ERC20` → the vault's balance of `asset` must rise by `≥ minOut`; `ERC721` → the vault must own
`≥ minOut` more NFTs; `NONE` → a pure outflow (e.g. a payment) that credits nothing back, so the result
check is skipped — but the over-spend and adaptor-kept-nothing invariants below still apply.

### 4.2 `execute(...)` — the runtime "hot potato"

`execute(adaptor, inputToken, amountIn, Expectation, callData)` runs, in one atomic transaction:

1. **Registry gate** — `registry.isApproved(adaptor)` or revert `AdaptorNotApproved`.
2. **Debit the allowance** (`_debit`): revert `AdaptorNotGranted` if the owner never opted in, `Expired`
   past `validUntil`, `OverPerTxCap` if `amountIn > perTxCap`, `OverPeriodCap` if the rolling window
   would be exceeded (the window resets after `periodLength`).
3. **Hand over EXACTLY `amountIn`** — `safeTransfer(adaptor, amountIn)`. A transfer, not an open
   approval, so even a malicious approved adaptor can pull at most this one call's `amountIn`.
4. **Call the adaptor with `call`, never `delegatecall`** — the adaptor runs in its own storage and can
   never touch the vault's `owner` or `allowances`.
5. **Post-conditions** — any violation reverts the whole transaction:

   | Check | Error | What it prevents |
   |---|---|---|
   | no more than `amountIn` left the vault | `OverSpend` | the adaptor pulling more than the one authorized amount |
   | `balanceOf(adaptor) == 0` for `inputToken` after | `AdaptorRetainedFunds` | the adaptor skimming or parking funds |
   | result asset up by `≥ minOut` (unless `kind = NONE`) | `ResultNotCredited` | swap/supply output going anywhere but the vault |

**Blast radius.** The agent chooses neither the recipient nor any retained funds; it only triggers an
owner-authorized adaptor within owner-set caps. An attacker holding the agent key can lose at most one
`perTxCap` of one granted `(adaptor, token)` — and for a swap/supply that value still lands **back in
the owner's vault**. Only a `NONE`-kind adaptor (a payment) is a true outflow, and it is still bounded to
one `amountIn`. The rest of the vault is unreachable.

### 4.3 `executeBatch(...)` — atomic multi-step (the EVM analog of a Sui PTB)

`executeBatch(Call[])` runs several adaptor calls in one transaction. Because every step forces its
result back into the vault before the next begins, a later step can **consume an earlier step's output**:

```
executeBatch([
  { swap  USDC → WETH  via UniswapV3SwapAdaptor, expect WETH ≥ minOut },
  { supply WETH → Aave via AaveV3SupplyAdaptor,  expect aWETH ≥ minOut },
])
```

Each step keeps the full per-step invariants from 4.2. If the Aave step reverts, the swap rolls back
too — the strategy is **all-or-nothing** and never lands half-done. A single `nonReentrant` guard wraps
the whole batch. Foundry coverage includes chained steps, mid-batch rollback, per-step allowance debit,
and `onlyAgent`.

### 4.4 What is trustless vs trusted here

- **Trustless (enforced by the contract):** owner-only deposit/withdraw, the registry check, the owner
  grant, the per-`(adaptor, token)` caps/expiry, "exactly `amountIn` leaves per call", "no
  `delegatecall`", "adaptor keeps nothing", and "the declared result lands in the vault".
- **Trusted but bounded to one `amountIn`/call:** the audited adaptor code and the curated registry.

---

## 5. Adaptor construction

An adaptor is the **only** thing the agent can point the vault at, so its shape is deliberately narrow.

### 5.1 The `IAdaptor` contract

```solidity
interface IAdaptor {
    // The vault has already sent EXACTLY `amountIn` of `inputToken` before this call.
    // Interact with ONE protocol; credit ALL results to `vault`; keep nothing.
    function execute(address inputToken, uint256 amountIn, address vault, bytes calldata callData) external;

    // The single protocol contract this adaptor is allowed to touch (informational, for inspection).
    function protocol() external view returns (address);
}
```

An adaptor is a **fixed, audited, stateless pass-through** to exactly one protocol. It is the EVM analog
of a Sup Sui adaptor: the vault's runtime post-conditions play the role of the Move *hot-potato* (results
forced back in-tx), and the curated registry plays the role of the unforgeable *witness type*.

### 5.2 Rules every adaptor must satisfy

The vault re-checks these at runtime, so a violation reverts the whole transaction — but a correct
adaptor is written to honor them by construction:

1. **Hold no funds across calls.** The vault sends `amountIn` immediately before `execute`; the adaptor
   consumes it and ends the call holding zero `inputToken` (`AdaptorRetainedFunds` otherwise).
2. **Credit all results to `vault`.** Set `recipient` / `onBehalfOf = vault` on the wrapped protocol
   call. Swapped tokens, LP/receipt tokens, and position NFTs all land in the vault, never in the adaptor
   or the agent.
3. **Never overspend** — use at most `amountIn` (`OverSpend` otherwise).
4. **`callData` is decoded parameters, never an arbitrary call target.** Pool tiers, `minOut`, tick
   ranges are decoded by the adaptor; an arbitrary target would defeat the allowlist.
5. **Exact, self-zeroing approvals.** Approve the protocol for exactly `amountIn` and reset to `0` after,
   so no standing allowance outlives the call.

### 5.3 Worked example — `UniswapV3SwapAdaptor`

```solidity
function execute(address inputToken, uint256 amountIn, address vault, bytes calldata callData) external {
    (address tokenOut, uint24 fee, uint256 minOut) = abi.decode(callData, (address, uint24, uint256));

    IERC20(inputToken).forceApprove(address(router), amountIn);   // exact approval
    router.exactInputSingle(ISwapRouter02.ExactInputSingleParams({
        tokenIn: inputToken, tokenOut: tokenOut, fee: fee,
        recipient: vault,                 // result lands in the VAULT, never here, never the agent
        amountIn: amountIn,
        amountOutMinimum: minOut,         // slippage floor at the router …
        sqrtPriceLimitX96: 0
    }));
    IERC20(inputToken).forceApprove(address(router), 0);          // no standing allowance survives
}
```

Slippage is enforced **twice** — `amountOutMinimum` at the router *and* the vault's own `expect.minOut`
post-condition. The immutable `router` address is the one protocol this adaptor may touch.

### 5.4 Two independent gates before an adaptor can ever run

1. **Curation** — the adaptor address must be `isApproved` in `AdaptorRegistry` (a curator-controlled,
   protocol-level allowlist).
2. **Owner grant** — the vault owner must `setAllowance(adaptor, token, perTxCap, periodCap,
   periodLength, validUntil)`.

Both are required; neither alone is enough. Publishing to the marketplace (`AdaptorListingRegistry`) is
discovery metadata only and grants nothing.

### 5.5 AI-authored adaptors

`/api/adaptors/author` drafts an `IAdaptor` implementation from a natural-language protocol description,
pre-bound to the invariants in 5.2. **Drafting or publishing never grants execution rights** — a human
still curates the contract into `AdaptorRegistry`, and the owner still grants the allowance in their
vault. The AI shortens authoring; it does not shortcut the trust gates.

---

## 6. Shipped adaptors

| Adaptor | Purpose | Result expectation |
|---|---|---|
| `PayAdaptor` | Pay a recipient from the vault | `AssetKind.NONE` (bounded outflow) |
| `UniswapV3SwapAdaptor` | Swap through Uniswap V3 | output token credited to vault, `minOut` enforced |
| `AaveV3SupplyAdaptor` | Supply to Aave V3 | `aToken` credited to vault, `minOut` enforced |
| `AaveV3WithdrawAdaptor` | Redeem vault `aToken` back to the underlying | underlying credited to vault, `minOut` enforced |

Uniswap LP, perps, and other protocol adaptors are planned marketplace additions, not shipped contracts.

---

## 7. UX flows

### A. Fund the vault

1. **Sign in** with email/Google (Privy embedded EOA), upgraded in place via EIP-7702 into a Particle
   Universal Account — same address, one balance across chains.
2. **Pick asset + amount** (USDC / WETH / native ETH). Native ETH is wrapped to WETH on deposit.
3. **Same-chain:** `vault.deposit(token, amount)` pulls from the UA — one signature.
4. **Cross-chain:** if the USDC is on another chain, Particle UA sources it
   (`createTransferTransaction(destinationChainId)`, or `createUniversalTransaction` for a contract-call
   route) and delivers value to Arbitrum, then the deposit completes. The UI shows the source chain(s)
   and **blocks on a source-chain mismatch** — funds respect their origin chain.

### B. Authorize an adaptor (grant)

- The first time Sup needs an adaptor, an **inline grant card** appears in chat — contextual, not an
  onboarding ceremony.
- The owner sets caps (`perTxCap`, `periodCap` over a `periodLength` window, `validUntil`); one signature
  calls `setAllowance(...)`.
- **Revoke anytime:** `revokeAllowance(adaptor, token)` flips `granted = false` instantly. Funds never
  left the vault, so there is nothing to unwind.

### C. Agent executes through an adaptor — 0 owner signatures

1. Natural language → typed action envelope.
2. **Money-path firewall:** recipient provenance, intent weld, preflight, receiver guard, failure guard.
3. `SupVault.execute` / `executeBatch` runs only if registry + grant + allowance + post-conditions all
   pass. The agent server wallet sends the tx and pays gas; the owner signs nothing.
4. A policy failure reverts and surfaces as a **"blocked"** card — a feature, not an error. Results stay
   in the vault.

### D. Withdraw

Owner-only. USDC/WETH transfer from the vault; WETH can be unwrapped to native ETH on withdrawal. The
agent can never withdraw or move a position out — it can only *operate* positions via `execute`.

### E. Marketplace

Anyone can publish adaptor discovery metadata to `AdaptorListingRegistry`. That is **not** executable
permission — execution still requires curation into `AdaptorRegistry` plus an owner grant.

---

## 8. Particle Universal Account — why it matters and how SupWallet uses it

The Universal Account is what makes the whole product approachable: it removes the parts of crypto that
scare a normal user (seed phrases, gas, bridging, "which chain is my money on") *before* the agent ever
does anything.

### 8.1 What Particle UA gives us

- **One address on every chain, upgraded in place (EIP-7702).** A social login mints a Privy embedded
  EOA; EIP-7702 upgrades *that same address* into a Universal Account. There is no second "smart wallet"
  address to teach users about — the EOA they see is the account that works everywhere.
- **A single unified balance.** Particle aggregates the user's holdings across chains into one spendable
  balance. The user thinks in dollars, not in "USDC on Base vs USDC on Arbitrum".
- **Cross-chain sourcing with no bridge UI.** A payment or deposit can be *sourced* from whatever chain
  the funds happen to sit on and *delivered* to Arbitrum, in one user action — no bridge selection, no
  wrapped-asset juggling, no chain picker.
- **One signature for multi-step actions (EIP-7702 batching).** Several userOps are covered by a single
  `personal_sign` over the batch root hash, and the 7702 authorization is a one-time-per-chain step — so
  "approve + deposit", or a multi-leg route, is one tap instead of several.
- **No separate account to pre-fund.** Because the UA *is* the user's account, there is never an "agent
  account" to top up on each chain — the classic funded-card UX problem simply doesn't exist here.

### 8.2 How SupWallet applies it

- **Onboarding.** Email/Google → Privy embedded EOA → EIP-7702 upgrade to a Particle UA. Same address,
  usable immediately; the user never handles a key.
- **Fund the vault from anywhere.** `createTransferTransaction(destinationChainId)` moves same-chain
  balance and delivers cross-chain to Arbitrum; `createUniversalTransaction` (with `expectTokens`)
  cross-*sources* from other chains for an arbitrary contract-call route. The deposit UX surfaces the
  actual **source chain(s)** and blocks on a source-chain mismatch, so funds "respect their origin chain"
  rather than silently moving balance the user didn't intend.
- **Payout ramp.** The same primitives move value back out of Arbitrum to wherever the user wants it.
- **Signature economy.** 7702 batching means a fund-and-deposit, or a route with several legs, asks the
  user for exactly one signature.

### 8.3 The deliberate boundary

The UA is the **funding and payout ramp**, not the vault. Cross-chain execution cannot offer the same
all-or-nothing invariant as an Arbitrum `executeBatch` (there is no single-transaction atomicity across
chains), so the vault/adaptor safety model is intentionally **single-chain on Arbitrum**: UA brings value
*to* the vault and takes it *out*, and every agent action that touches the principal happens inside the
one chain where the post-conditions can be enforced atomically.

---

## 9. Trust boundaries

- **Onchain enforced:** vault custody, owner-only deposit/withdraw, curated registry check, owner grant,
  per-`(adaptor, token)` allowance, exact `amountIn`, no `delegatecall`, adaptor residual check, and
  `minOut` result credit.
- **Trusted but bounded:** curated adaptor code. A bad curated adaptor is still capped by one authorized
  `amountIn` per call and cannot directly drain the rest of the vault.
- **Discovery only:** marketplace listings and AI-authored drafts.
- **Offchain convenience:** agent brain, indexing, KV manifests, optional Walrus / memory providers.
- **Never trusted:** the LLM is never the source of custody authority.

---

## 10. Status

- **Deployed on Arbitrum One:** factory, vault implementation, curated registry, listing registry,
  `PayAdaptor`, `UniswapV3SwapAdaptor`, `AaveV3SupplyAdaptor`, `AaveV3WithdrawAdaptor`.
- **Built/tested:** `execute`, `executeBatch`, chained batch steps, mid-batch rollback, per-step
  allowance, and `onlyAgent`; Aave supply/withdraw exercised against a mainnet fork.
- **UI shipped:** vault deposit/withdraw for USDC/WETH/native ETH, Particle UA cross-chain funding, and
  autonomous agent execution (swap + Aave supply/withdraw) end-to-end with real USDC.
- **Planned:** Uniswap LP, perps adaptors, expanded marketplace analytics and review flows.
</content>
</invoke>
