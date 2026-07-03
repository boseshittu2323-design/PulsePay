# VeriDose

**Pharmaceutical supply chain verification and anti-counterfeit tracking, built on Stellar.**

VeriDose gives every drug batch a cryptographically verifiable identity as it moves from manufacturer to distributor to pharmacy to patient — making counterfeit drugs detectable in seconds and enabling instant, automated recalls across an entire supply chain.

> Other name options considered: PharmaTrail, ChainRx, TrueBatch, MedLedger, PharmaTrust, DoseChain, CustodyRx, AuthentiMed, StellarScript.

---

## Table of Contents

- [Why This Exists](#why-this-exists)
- [How It Works](#how-it-works)
- [Architecture](#architecture)
- [Core Components](#core-components)
- [Data Model](#data-model)
- [Smart Contracts (Soroban)](#smart-contracts-soroban)
- [Tech Stack](#tech-stack)
- [Getting Started](#getting-started)
- [Project Structure](#project-structure)
- [Roadmap](#roadmap)
- [Scalability Notes](#scalability-notes)
- [Security Considerations](#security-considerations)
- [Contributing](#contributing)
- [License](#license)

---

## Why This Exists

The World Health Organization estimates that **1 in 10 medical products in low- and middle-income countries is substandard or falsified**. Existing anti-counterfeit systems (holograms, serial numbers, centralized databases) are either easy to forge, siloed per-country, or too expensive for smaller manufacturers and pharmacies to adopt.

VeriDose solves this using Stellar's low-cost, fast-settlement ledger to create a **shared, tamper-evident custody trail** for every batch of medicine — without requiring every participant to trust a single central authority.

---

## How It Works

1. A **manufacturer** produces a batch of medicine and issues a corresponding **Stellar asset** representing that batch (not individual units — see [Scalability Notes](#scalability-notes)).
2. As the batch moves through the supply chain, each handoff (manufacturer → distributor → wholesaler → pharmacy) is recorded as a **custody transfer** on Stellar, signed by both parties.
3. Regulators hold a **co-signer role** and can flag or freeze a batch at any point if it's found to be counterfeit, contaminated, or expired.
4. **Patients or pharmacists** scan a QR code on the packaging, which queries a **Soroban smart contract** to instantly verify: authenticity, current custody status, expiry, and recall status.
5. If a batch is recalled, the **recall contract** automatically notifies every wallet currently holding that batch downstream — no manual outreach needed.

---

## Architecture

```
┌─────────────────┐        ┌──────────────────┐        ┌─────────────────┐
│   Manufacturer   │──────▶│    Distributor     │──────▶│    Pharmacy      │
│  (issues batch    │       │  (custody transfer  │       │ (custody transfer │
│   asset)          │       │   + multi-sig)       │       │  + multi-sig)     │
└─────────────────┘        └──────────────────┘        └────────┬────────┘
                                                                    │
                                                                    ▼
                                                          ┌──────────────────┐
                                                          │     Patient        │
                                                          │  (scans QR code,    │
                                                          │  calls Verify        │
                                                          │  contract)           │
                                                          └──────────────────┘

           ▲                         ▲                          ▲
           │                         │                          │
           └─────────────────────────┴──────────────────────────┘
                        Regulator (read/co-sign access)
                     Can trigger Recall Contract at any node

┌───────────────────────────── Off-chain layer ─────────────────────────────┐
│  IPFS / regulator database: lab certificates, manufacturing logs,          │
│  packaging images, full batch metadata                                     │
└──────────────────────────────────────────────────────────────────────────┘
```

**Core principle:** only cryptographic proofs and custody state live on-chain. Bulk data (certificates, images, logs) lives off-chain and is referenced by hash — this keeps the system cheap and fast even at national scale.

---

## Core Components

| Component | Purpose |
|---|---|
| **Batch Asset Issuer** | Issues one Stellar asset per production batch (not per unit) |
| **Custody Transfer Service** | Manages claimable balances representing handoffs between supply chain actors |
| **Verification Contract** | Soroban contract patients/pharmacists call to confirm authenticity |
| **Recall Contract** | Soroban contract that freezes a batch and pushes alerts downstream |
| **Regulator Multi-Sig Layer** | Grants regulators read + flagging rights without full custody control |
| **Off-chain Metadata Store** | IPFS-based storage for certificates, images, and manufacturing logs, referenced on-chain by hash |
| **QR Verification App** | Mobile/web app for scanning batch QR codes and querying contract state |

---

## Data Model

### Batch Asset
```
asset_code:      e.g. "PARA-2026-0912" (max 12 chars, Stellar asset code limit)
issuer:          Manufacturer's Stellar public key
metadata_hash:   IPFS hash pointing to full batch details
expiry_date:     Unix timestamp
unit_count:      Total units in this batch
status:          active | recalled | expired
```

### Custody Record (Claimable Balance)
```
batch_asset:     Reference to the batch asset
from:            Sending party's public key
to:              Receiving party's public key
timestamp:       Transfer time
signatures:      Multi-sig proof from both parties
location:        Optional geo/facility tag
```

### Off-chain Metadata (IPFS)
```json
{
  "batch_id": "PARA-2026-0912",
  "manufacturer": "Acme Pharmaceuticals Ltd.",
  "manufacturing_site": "Lagos, Nigeria",
  "lab_certificate": "ipfs://Qm...",
  "packaging_images": ["ipfs://Qm...", "ipfs://Qm..."],
  "active_ingredients": ["Paracetamol 500mg"],
  "regulatory_approval_id": "NAFDAC-2026-XXXX"
}
```

---

## Smart Contracts (Soroban)

### 1. `verification_contract`
- `verify_batch(batch_id) -> BatchStatus`
- `get_custody_history(batch_id) -> Vec<CustodyRecord>`
- Called by pharmacies and patients scanning a QR code.

### 2. `recall_contract`
- `flag_batch(batch_id, reason)` — regulator-only
- `freeze_batch(batch_id)` — halts further custody transfers
- `notify_holders(batch_id)` — emits an event to all wallets currently holding the batch

### 3. `expiry_contract`
- `check_expiry(batch_id) -> bool`
- `auto_flag_expired()` — scheduled job that flags batches past expiry, useful for donor-funded programs (WHO, Global Fund) requiring proof drugs weren't sold post-expiry.

### 4. `custody_transfer_contract`
- `initiate_transfer(batch_id, to)` — creates a claimable balance
- `accept_transfer(batch_id)` — receiving party signs and claims
- `reject_transfer(batch_id, reason)` — flags a disputed handoff

---

## Tech Stack

| Layer | Technology |
|---|---|
| Ledger | Stellar (mainnet/testnet) |
| Smart Contracts | Soroban (Rust) |
| Off-chain Storage | IPFS |
| Backend API | Node.js / Express or Rust (Axum) |
| Frontend (QR scanner app) | React Native (mobile) + React (web dashboard) |
| Regulator Dashboard | React + Stellar SDK |
| Oracles (for expiry/compliance triggers) | Custom scheduled job or Chainlink-style Soroban oracle integration |

---

## Getting Started

### Prerequisites
- [Rust](https://www.rust-lang.org/tools/install) + `soroban-cli`
- [Stellar SDK](https://developers.stellar.org/docs/tools/sdks) (JS, Python, or Go)
- Node.js 18+
- IPFS node or a pinning service (e.g., Pinata, Web3.Storage)

### Setup

```bash
# Clone the repo
git clone https://github.com/your-org/veridose.git
cd veridose

# Install dependencies
npm install

# Set up Soroban contracts
cd contracts
soroban contract build
soroban contract deploy --network testnet --source <your-key>

# Run the backend
cd ../backend
npm run dev

# Run the frontend
cd ../frontend
npm run dev
```

### Environment Variables

```env
STELLAR_NETWORK=testnet
SOROBAN_RPC_URL=https://soroban-testnet.stellar.org
ISSUER_SECRET_KEY=your_manufacturer_secret_key
IPFS_API_URL=https://your-ipfs-node-or-pinning-service
REGULATOR_PUBLIC_KEYS=GABCD...,GEFGH...
```

---

## Project Structure

```
veridose/
├── contracts/                 # Soroban smart contracts (Rust)
│   ├── verification/
│   ├── recall/
│   ├── expiry/
│   └── custody_transfer/
├── backend/                   # API layer connecting Stellar + IPFS
│   ├── src/
│   │   ├── routes/
│   │   ├── services/
│   │   └── stellar/
├── frontend/                  # QR scanner + verification app
│   └── src/
├── regulator-dashboard/       # Regulator-facing web app
│   └── src/
├── docs/
│   └── architecture.md
└── README.md
```

---

## Roadmap

- [x] Concept design and architecture
- [ ] MVP: single country, single drug category (e.g., malaria drugs, Nigeria)
- [ ] Manufacturer batch issuance + custody transfer flow
- [ ] QR verification app (patient/pharmacist facing)
- [ ] Regulator dashboard with flag/recall capability
- [ ] Automated recall notification system
- [ ] Multi-country anchor rollout
- [ ] Integration with invoice financing (pay distributors instantly on verified delivery)
- [ ] Mainnet launch + pilot partnership with a regulatory body or NGO

---

## Scalability Notes

- **Batch-level, not unit-level, tokenization.** One asset represents an entire production batch, not each pill — this keeps ledger writes proportional to supply chain *steps*, not physical units, so the system can handle millions of units without ledger bloat.
- **Off-chain/on-chain split.** Only hashes, custody signatures, and status flags go on-chain. Certificates, images, and logs live on IPFS.
- **Per-country anchors.** Each regulator can run its own compliant node/interface while remaining interoperable across borders through a shared issuer chain — critical for cross-border donor drug programs.
- **Event-driven recalls.** The recall contract pushes state changes rather than requiring every downstream holder to poll — this scales to large, deep supply chains without added load.

---

## Security Considerations

- Manufacturer issuer keys should be held in **multi-sig cold storage**, not a single hot wallet.
- Regulator co-signer keys should be **rotated periodically** and secured via HSM or similar.
- All off-chain metadata referenced on-chain must be **content-addressed** (IPFS hash) to prevent tampering after the fact.
- Rate-limit and authenticate QR verification calls to prevent scraping of the full custody graph by bad actors.
- Conduct a third-party audit of all Soroban contracts before mainnet deployment, given the compliance stakes involved.

---

## Contributing

Contributions are welcome. Please open an issue to discuss significant changes before submitting a pull request. See `CONTRIBUTING.md` (to be added) for coding standards and PR guidelines.

---

## License

MIT License — see `LICENSE` file for details.
