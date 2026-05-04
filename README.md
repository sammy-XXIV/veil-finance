# VEIL Finance

**Confidential lending protocol built on Zama FHEVM**

VEIL lets users deposit cWETH as encrypted collateral and borrow against it. Liquidation bots cannot compute health factors — all position data is encrypted onchain using Fully Homomorphic Encryption.

---

## Live Demo

**Frontend:** https://sammy-xxiv.github.io/veil-finance  
**Network:** Ethereum Sepolia Testnet

---

## How It Works

Traditional lending protocols store collateral and debt in plaintext. Any bot can monitor your health factor and liquidate the moment your position becomes eligible. VEIL changes this.

With VEIL:
- Your collateral amount is stored as an encrypted `euint64` onchain
- Your debt amount is stored as an encrypted `euint64` onchain
- Health factor computation happens inside the FHE coprocessor — the result is never exposed in plaintext
- Liquidation bots see only ciphertext handles — they cannot determine if your position is liquidatable

The only information visible onchain is what you explicitly reveal — the `plainAmount` parameter used for pool accounting.

---

## Protocol Parameters

| Parameter | Value |
|-----------|-------|
| Max LTV | 66% |
| Liquidation Threshold | 150% |
| Liquidation Bonus | 5% |
| Collateral Token | cWETH (ERC-7984) |
| Debt Token | cWETH (ERC-7984) |
| Network | Sepolia Testnet |

---

## Contracts

| Contract | Address |
|----------|---------|
| VeilLending | `0x1689b2e699bD28Dc21A8442Ec8e3D39F5d52dDCB` |
| cWETHMock | `0x46208622DA27d91db4f0393733C8BA082ed83158` |
| Underlying WETH | `0xff54739b16576FA5402F211D0b938469Ab9A5f3F` |

---

## How to Test

1. Connect MetaMask or any injected wallet to Sepolia Testnet
2. Get test ETH from a Sepolia faucet (e.g. `sepoliafaucet.com`)
3. Go to the Deposit page and click **Get 0.005 cWETH — Faucet**
4. Click **View Balance** — sign the EIP-712 message to decrypt your cWETH balance via FHE
5. Enter a deposit amount and click **Deposit & Open Position**
   - Sign the balance decryption (first popup)
   - Sign the deposit transaction (second popup)
6. View your encrypted position on the Dashboard
7. Borrow up to 66% of your collateral value on the Borrow page
8. Repay on the Repay page and close your position when debt is cleared

---

## Architecture

```
Frontend (GitHub Pages)
    ↓ encrypt request
Backend (Render — Node.js)
    ↓ @zama-fhe/relayer-sdk
Zama Relayer (Sepolia Testnet)
    ↓ FHE proof
VeilLending Contract (Sepolia)
    ↓ confidentialTransferFrom
cWETHMock ERC-7984 (Sepolia)
```

**Frontend** — Single HTML file, ethers.js v6, no framework  
**Backend** — Node.js on Render, handles FHE encryption and user decryption  
**Contracts** — Solidity 0.8.24, `@fhevm/solidity`, Zama FHEVM coprocessor  

---

## Key FHE Operations

| Operation | FHE Function |
|-----------|-------------|
| Store collateral | `FHE.fromExternal()` |
| Add collateral | `FHE.add()` |
| Compute health factor | `FHE.lt()`, `FHE.mul()` |
| Liquidation decision | `FHE.and()`, `FHE.select()` |
| Decrypt balance | `userDecrypt()` via Zama Relayer |

---

## Known Limitations

- `plainAmount` is user-provided and not cryptographically verified against the actual FHE transfer amount. This is a known limitation of the current Zama FHEVM design — encrypted transfer amounts cannot be read in plaintext by the contract. The frontend mitigates this by decrypting the user's real balance before accepting a deposit.
- Testnet only — cWETH and Sepolia ETH have no real monetary value.
- Faucet dispenses 0.005 cWETH per hour per address.

## Tech Stack

- Solidity 0.8.24
- Zama FHEVM (`@fhevm/solidity`)
- OpenZeppelin Confidential Contracts (ERC-7984)
- ethers.js v6
- Node.js + Express (backend)
- GitHub Pages (frontend)
- Render (backend hosting)


