# Architecture

SupWallet is a **chain‑agnostic, self‑custodial agent wallet**. The user’s own address — upgraded to a Universal Account via EIP‑7702 — is both their wallet and the surface an AI agent is allowed to spend from, bounded by a policy enforced in a TEE.

This document covers the accounts, the flows, the trust boundaries, and the design decisions behind the v1 architecture.

---

## Accounts

| Component | Role | Custody |
|---|---|---|
| **Privy embedded wallet** | The user’s key, created on social login. Its private key lives in Privy’s secure enclave (TEE). | User |
| **Particle Universal Account** | The same EOA, upgraded **in place via EIP‑7702** — same address, unified balance across chains. | User (it *is* the user’s address) |
| **Agent scoped signer** | A backend‑controlled authorization key added as an **additional signer** on the user’s wallet, restricted by a policy. | Backend (signer only — never owns funds) |

There is **no separate agent account** and **no custom smart contract**. The agent is a policy‑scoped *signer* on the user’s own Universal Account.

---

## The two spending paths

1. **Cross‑chain (Universal Account transfer).** Bringing value in, or paying across chains, goes through Particle’s Universal Account flow, which sources liquidity across chains and settles to the destination. Signed by the user.
2. **Autonomous agent spend (plain transfer).** The agent’s day‑to‑day payments are plain `USDC.transfer` transactions on Arbitrum, signed by the agent’s scoped signer. Because an EIP‑7702 account’s delegated code only runs when the account is *called* — not when it *sends* — a plain outbound transfer executes normally, and Privy’s policy engine can inspect the raw calldata (recipient, amount) to gate it.

---

## Flows

### 0 · Onboarding (one‑time)

```mermaid
sequenceDiagram
    actor U as User
    participant P as Privy
    participant PA as Particle UA SDK
    U->>P: Social login (email / Google)
    P-->>U: Embedded wallet (EOA) created
    U->>PA: Sign EIP-7702 authorization (once, per chain)
    PA-->>U: EOA is now a Universal Account (same address)
```

### 1 · Fund from any chain — the cross‑chain UA transfer

```mermaid
sequenceDiagram
    actor U as User
    participant UA as Universal Account
    participant X as Any source chain
    U->>UA: "Add 20 USDC"
    UA->>X: Universal Account transfer (sources liquidity cross-chain)
    X-->>UA: USDC settled on Arbitrum (same address)
```

### 2 · Give Sup an allowance (add scoped signer + policy)

```mermaid
sequenceDiagram
    actor U as User
    participant B as Backend
    participant P as Privy
    U->>B: "Let Sup pay up to 5 USDC/mo to X"
    B->>P: createPolicy(allowlist + per-tx cap + rolling cap)
    B->>P: addSigner(agent key, override_policy_ids=[policy])
    P-->>U: Consent (in-app) — no gas, no on-chain tx
```

### 3 · Agent spends (0 user signatures)

```mermaid
sequenceDiagram
    participant Agent as Agent (backend)
    participant P as Privy TEE
    participant C as Arbitrum
    Agent->>P: sign eth_sendTransaction(USDC.transfer → recipient, amount)
    P->>P: Evaluate signer policy in enclave
    alt within policy
        P->>C: broadcast (gas sponsored)
        C-->>Agent: tx hash ✓
    else off-allowlist / over-cap
        P-->>Agent: DENY (rejected before broadcast)
    end
```

### 4 · Revoke (instant, off‑chain)

```mermaid
sequenceDiagram
    actor U as User
    participant P as Privy
    U->>P: removeSigner(agent)
    P-->>U: Agent cut off immediately (no gas, no on-chain tx)
```

---

## Trust boundaries

```mermaid
flowchart LR
    subgraph safe["Bounded by policy (enforced in TEE)"]
        A["Agent scoped signer"]
    end
    subgraph root["Never used to move user funds"]
        R["App root secret"]
    end
    A -- "only allowlisted recipient, ≤ per-tx cap, ≤ rolling cap" --> U["User's USDC"]
    U -. "self-custody, can revoke signer anytime" .-> A
```

- **The agent is a signer, not an owner.** It can request signatures within its policy; it cannot change its own policy, add signers, or exceed its caps.
- **Policy is enforced at signing time in Privy’s secure enclave.** A violating transaction is never signed, so it never reaches the chain.
- **Learned the hard way (and designed for):** Privy’s *root app secret bypasses policies by design*. So the backend moves user funds **only** through the scoped, policy‑bound agent signer — never with the root secret. This separation is what makes “agent autonomy” safe.
- **Instant revocation.** Removing the signer cuts the agent off immediately; the principal never left the user’s wallet.

**Honest trade‑off:** v1 enforces the fine‑grained policy in Privy’s TEE rather than in an on‑chain contract. The absolute ceiling is the user’s own balance (self‑custody), and the fine limits are trusted to Privy’s enclave. A fully trustless on‑chain variant (see below) is kept as a future mode.

---

## Design decisions (how we got here)

1. **Funded agent “card” (rejected).** A separate ERC‑4337 account the user pre‑funds. Clean hard cap, but the user babysits balances and a new address on every chain — poor UX for the Universal Accounts vision.
2. **On‑chain `AllowanceRouter` (built, kept as reference).** The user `approve`s a custom contract; the agent spends via `transferFrom` under on‑chain allowlist/cap/rolling limits. Fully trustless (deployed + tested against real USDC). But it adds an approve step, gas for grant/revoke, and a contract to maintain.
3. **Contract‑less: Universal Account + TEE policy (shipped).** The agent is a scoped signer on the user’s own 7702 Universal Account; policy is enforced in Privy’s TEE. No approve, no `transferFrom`, no separate account, no contract — grant and revoke are gasless in‑app actions. This matches the Universal Accounts + 7702 model (“no new address, no contract deployment”) and optimizes for UX.

The result: **the account abstraction is the account**, and the agent is a bounded signer on it.

---

## Standards & building blocks

- **EIP‑7702** — upgrade an EOA in place; same address gains smart‑account behavior.
- **Particle Universal Accounts** — one address, one balance, cross‑chain execution & liquidity.
- **Privy** — embedded wallets, authorization‑key signers, TEE policy engine, gas sponsorship.
- **Arbitrum One** — settlement chain; **USDC** — unit of account.
