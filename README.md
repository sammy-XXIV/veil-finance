# VEIL Finance

**Confidential lending protocol built on Zama FHEVM**

VEIL lets users deposit cWETH as encrypted collateral and borrow against it. Liquidation bots cannot compute health factors — all individual position data is encrypted onchain using Fully Homomorphic Encryption.

---

## Live Demo

**Frontend:** https://sammy-xxiv.github.io/veil-finance  
**Network:** Ethereum Sepolia Testnet

---

## How It Works

Traditional lending protocols store collateral and debt in plaintext. Any bot can call `getUserAccountData()`, compute your health factor, and liquidate you the moment your position is eligible.

VEIL changes this at the protocol level.

- Collateral is stored as an encrypted `euint64` — no plaintext ever written to storage
- Debt is stored as an encrypted `euint64` — same
- Health factor computation runs inside the FHE coprocessor — result is an encrypted `ebool`, never exposed
- Liquidation bots can only call `hasPosition(address)` — returns true/false, nothing else
- Even if a bot calls `liquidate()` on a healthy position — the contract silently returns 0 via `FHE.select`. The bot wastes gas and learns nothing.

**What bots see:** `hasPosition = true`. That's it.  
**What bots need:** collateral amount, debt amount, health factor. All encrypted.

---

## Protocol Parameters

| Parameter | Value |
|-----------|-------|
| Max LTV | 66% |
| Liquidation Threshold | 150% |
| Collateral Token | cWETH (ERC-7984) |
| Debt Token | cWETH (ERC-7984) |
| Network | Sepolia Testnet |

---

## Contracts

| Contract | Address |
|----------|---------|
| VeilLending | `0x366b34bdC8ca3477A9884E40F1E385aAe54941F2` |
| cWETHMock | `0x46208622DA27d91db4f0393733C8BA082ed83158` |
| Underlying WETH | `0xff54739b16576FA5402F211D0b938469Ab9A5f3F` |

---

## How to Test

1. Connect MetaMask to Sepolia Testnet
2. Get test ETH from a Sepolia faucet (e.g. `sepoliafaucet.com`)
3. Go to the Deposit page and click **Get cWETH — Faucet**
4. Click **View Balance** — sign the EIP-712 message to decrypt your cWETH balance
5. Enter a deposit amount and click **Deposit & Open Position**
6. View your encrypted position on the Dashboard — amounts show 🔒 Encrypted
7. Click **View Balance** to decrypt and see your collateral
8. Click **Check** next to Liq. Price to decrypt your debt and see your liquidation price
9. Borrow up to 66% of your collateral on the Borrow page
10. Repay on the Repay page before closing your position

---

## Architecture

```
Frontend (GitHub Pages)
    ↓ encrypt request
Backend (Render — Node.js)
    ↓ @zama-fhe/relayer-sdk
Zama Relayer (Sepolia Testnet)
    ↓ FHE proof + handle
VeilLending Contract (Sepolia)
    ↓ confidentialTransferFrom → returns euint64
cWETHMock ERC-7984 (Sepolia)
```

**Frontend** — Single HTML file, ethers.js v6, no framework  
**Backend** — Node.js on Render, handles FHE encryption and user decryption  
**Contracts** — Solidity 0.8.24, `@fhevm/solidity`, Zama FHEVM coprocessor  

---

## Key FHE Operations

| Operation | FHE Function |
|-----------|-------------|
| Store collateral | `confidentialTransferFrom` return value → `euint64` |
| Add collateral | `FHE.add()` |
| LTV check | `FHE.le(FHE.mul(debt,100), FHE.mul(collateral,66))` → `ebool` |
| Borrow enforcement | `FHE.select(withinLTV, amount, 0)` |
| Close position check | `FHE.eq(debt, 0)` → `FHE.select` |
| Liquidation decision | `FHE.lt()`, `FHE.and()`, `FHE.select()` |
| Repay cap | `FHE.min(received, debt)` |
| Decrypt balance | `userDecrypt()` via Zama Relayer + EIP-712 signature |

---

## Security Design

**Deposit** — Amount is cryptographically verified via `confidentialTransferFrom` return value. No user-supplied plaintext for collateral.

**Repay** — Same. Actual received amount is used to reduce debt. No user-supplied plaintext.

**Borrow** — LTV enforced in FHE via `FHE.select`. Exceeding 66% silently returns 0 tokens. Frontend enforces LTV before sending tx.

**Close Position** — FHE checks if debt == 0. If debt exists, collateral return is 0 via `FHE.select`. Frontend shows warning modal before close.

**Liquidation** — Full health factor computed in FHE. Liquidator gets collateral only if position is underwater. Healthy positions return 0 to liquidator silently.

---

## Known Limitations

**`plainAmount` in borrow** — When borrowing, the user supplies a plaintext amount for internal pool accounting. This value is trusted but cannot affect actual token transfers — the cWETH token enforces real balances. Lying about `plainAmount` only affects the internal counter, not real funds. This is a fundamental FHE limitation: the contract cannot read the plaintext result of `confidentialTransfer` to verify the amount sent. Once Zama ships encrypted return values from `confidentialTransfer`, this is fully fixable.

**FHE cannot revert on encrypted conditions** — LTV violations and debt-on-close return 0 silently instead of reverting. The frontend catches these cases with pre-flight checks and warnings.

**Testnet only** — cWETH and Sepolia ETH have no real monetary value.

---

## Roadmap Fix (Zama Dependent)

Once Zama ships:
1. **Encrypted return values from `confidentialTransfer`** — removes `plainAmount` trust in borrow
2. **FHE-based conditionals with revert** — allows onchain LTV enforcement with proper error messages

Both are on Zama's public roadmap. VEIL is designed to adopt these improvements with minimal contract changes.

---

## Tech Stack

- Solidity 0.8.24
- Zama FHEVM (`@fhevm/solidity`)
- OpenZeppelin Confidential Contracts (ERC-7984)
- `@zama-fhe/relayer-sdk` v0.4.0-5
- ethers.js v6
- Node.js + Express (backend)
- GitHub Pages (frontend)
- Render (backend hosting)

---
