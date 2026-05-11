[README.md](https://github.com/user-attachments/files/27603772/README.md)
# ImprintID — Proof of Experience Protocol on Solana

> *We don't verify who you are. We verify how you interact with reality.*

ImprintID is a behavioral identity protocol that transforms emotional responses to visual stimuli into verifiable on-chain data. Participants complete a 4-stimulus experiment; their responses are SHA-256 hashed, uploaded to IPFS, and minted as a **Metaplex NFT on Solana Devnet** — all without requiring the participant to hold any SOL.

---

## Features

- **Zero-SOL for participants** — the API server pays all transaction fees; participants only connect their Phantom wallet to receive the NFT
- **Behavioral fingerprinting** — ISS (Imprint Sensitivity Score) scoring based on Bowlby/Ainsworth attachment theory
- **A/B visual variants** — color photo vs. grayscale+data overlay, randomly assigned per session
- **Metaplex NFT minting** — server-side via UMI + keypairIdentity, token sent directly to participant's wallet
- **Dual-path anchoring** — custom Anchor PDA (when `VITE_PROGRAM_ID` is set) or Memo Program fallback
- **IPFS persistence** — full behavioral payload uploaded via web3.storage (or mock URI)
- **Non-fatal design** — every on-chain step is gracefully degraded; the experiment always completes

---

## Tech Stack

| Layer | Technology |
|---|---|
| Monorepo | pnpm workspaces, Node.js 24, TypeScript 5.9 |
| Frontend | React 19 + Vite 7, Tailwind v4, Framer Motion |
| Solana (client) | `@solana/wallet-adapter-react`, `@coral-xyz/anchor` ^0.32, Phantom |
| Solana (server) | `@metaplex-foundation/umi`, `mpl-token-metadata`, `@solana/web3.js` |
| On-chain primary | Custom Anchor program — PDA per wallet, `seeds = ["imprint", owner]` |
| On-chain fallback | Memo Program (`MemoSq4gqABAXKb96qnH8TysNcWxMyWCqXgDLGmfcHr`) |
| Data | SHA-256 (crypto-js) + web3.storage IPFS |
| API | Express 5, PostgreSQL + Drizzle ORM, Zod |

---

## Architecture

```
Participant (browser)
  ↓  connects Phantom (wallet gate — no SOL needed)
  ↓  completes 4-stimulus experiment
  ↓  POST /api/nft/mint  ─────────────────────────────────────┐
                                                               ↓
                                                    API Server (server wallet pays)
                                                      ├─ Mint Metaplex NFT → participant wallet
                                                      └─ Anchor hash via Memo Program tx
  ↓  receives NFT mint address + Solscan links
  ↓  Dashboard shows ISS profile + on-chain proof
```

**Server wallet** — derived deterministically from `SESSION_SECRET` via SHA-256 (`imprintid-solana-payer:<secret>`). Stable across restarts; fund it once.

---

## Project Structure

```
artifacts/
  imprintid/          # React + Vite frontend (port 19933)
    src/
      pages/          # Landing, Experiment, Dashboard
      components/     # WalletButton, StimulusVisual, ScaleSlider
      lib/
        anchor.ts     # Anchor client: PDA derivation, createImprintAnchor()
        solana.ts     # Memo Program fallback, airdrop helpers
        iss.ts        # ISS scoring + attachment profile classification
        hash.ts       # SHA-256 payload hashing
        ipfs.ts       # IPFS upload (web3.storage or mock)
        nft.ts        # NFT metadata builder (Metaplex standard)
        program.rs    # Anchor program source → paste into Solana Playground
        idl.json      # IDL template → replace after Playground deploy

  api-server/         # Express 5 API (port 8080)
    src/
      routes/
        nft.ts        # POST /nft/mint, GET /nft/wallet
        health.ts     # GET /healthz
      lib/
        solana-nft.ts # Server keypair, Metaplex UMI mint, Memo Program anchor

lib/
  api-spec/           # OpenAPI contract (codegen source)
  api-zod/            # Zod schemas (auto-generated)
  api-client-react/   # React Query hooks (auto-generated)
  db/                 # Drizzle ORM schema + migrations
```

---

## Getting Started

### Prerequisites

- **Node.js 24** (`node -v`)
- **pnpm 9+** (`npm i -g pnpm`)
- **Phantom wallet** browser extension (set to **Devnet**)

### 1. Clone & Install

```bash
git clone https://github.com/<your-org>/imprintid.git
cd imprintid
pnpm install
```

### 2. Configure Environment

```bash
cp .env.example .env
# Edit .env with your values
```

Required variables:

| Variable | Description |
|---|---|
| `SESSION_SECRET` | Secret string (min 32 chars) — also derives the server wallet keypair |
| `DATABASE_URL` | PostgreSQL connection string |

Optional variables:

| Variable | Description |
|---|---|
| `VITE_SOLANA_RPC` | Custom Devnet RPC endpoint (defaults to public) |
| `VITE_WEB3_STORAGE_KEY` | web3.storage API key for real IPFS (falls back to mock URI) |
| `VITE_PROGRAM_ID` | Deployed Anchor program ID (falls back to Memo Program) |
| `SOLANA_SERVER_KEYPAIR` | JSON array of secret key bytes — override the derived keypair |

### 3. Fund the Server Wallet

The server pays all Solana fees so participants never need SOL. Fund the server wallet **once**:

```bash
# Check the current server wallet address
curl http://localhost:8080/api/nft/wallet
```

Then visit **https://faucet.solana.com**, paste the address, and request **5 SOL** (covers ~300 NFT mints).

### 4. Run

```bash
# Terminal 1 — API server
pnpm --filter @workspace/api-server run dev

# Terminal 2 — Frontend
pnpm --filter @workspace/imprintid run dev
```

Frontend: http://localhost:19933  
API: http://localhost:8080/api/healthz

---

## ISS Scoring — Imprint Sensitivity Score

The ISS (0–100) is computed from 4 stimulus responses using Bowlby/Ainsworth attachment theory:

| Score | Profile | Description |
|---|---|---|
| 60–100 | **Seguro** (Secure) | Integrated emotional processing |
| 40–59 | **Ansioso** (Anxious) | Hyperactivated attachment system |
| 20–39 | **Evitativo** (Avoidant) | Deactivated emotional processing |
| 0–19 | **Desorganizado** (Disorganized) | Collapsed attachment strategies |

**Research variables recorded per participant:**

- `E` — Emotional Intensity (mean scale avg, 0–10)
- `A` — Association Rate (personal connection fraction, 0–1)
- `T` — Mean Observe Time (ms)
- `D` — Emotional Delta (max − min stimulus intensity)
- `V` — Emotional Variance (std deviation of intensities)

---

## Deploying the Anchor Program

The custom Anchor program stores one behavioral-identity PDA per wallet.

1. Open **https://beta.solpg.io/** → Create project → select **Anchor**
2. Replace `lib.rs` with `artifacts/imprintid/src/lib/program.rs`
3. Click **Build** → **Deploy** → note the **Program ID**
4. Click **Export → IDL** → download the JSON
5. Replace `artifacts/imprintid/src/lib/idl.json` with the download
6. Set env var: `VITE_PROGRAM_ID=<your_program_id>`
7. Restart the frontend

**On-chain account (256 bytes):**

| Field | Type | Description |
|---|---|---|
| `owner` | Pubkey | Signer wallet |
| `hash` | String (64) | SHA-256 of behavioral payload |
| `uri` | String (100) | IPFS URI |
| `iss` | u8 | Imprint Sensitivity Score (0–100) |
| `attachment_profile` | String (16) | seguro / ansioso / evitativo / desorganizado |
| `variant` | String (2) | A/B test variant |
| `timestamp` | i64 | Unix timestamp |
| `bump` | u8 | PDA canonical bump |

---

## API Reference

### `POST /api/nft/mint`

Mint a Metaplex NFT and anchor the behavioral hash. Server pays all fees.

**Request body:**
```json
{
  "walletAddress": "...",
  "metadataUri": "ipfs://...",
  "nftName": "ImprintID — Seguro",
  "hash": "sha256hex...",
  "uri": "ipfs://payload...",
  "iss": 72,
  "attachmentProfile": "seguro",
  "variant": "A",
  "timestamp": 1715000000000
}
```

**Response:**
```json
{
  "mint": "NFT mint address",
  "txSignature": "Metaplex tx signature",
  "memoSignature": "Memo Program anchor tx signature",
  "serverWallet": "server payer address",
  "serverBalanceSOL": 4.98
}
```

### `GET /api/nft/wallet`

Returns server wallet status.

```json
{
  "pubkey": "4RnmsHgZPhKCzNEp9Gsd2Nmj98HGAdwZYnkntHviRuj9",
  "balanceSOL": 4.98,
  "needsFunding": false,
  "faucetUrl": "https://faucet.solana.com"
}
```

### `GET /api/healthz`

```json
{ "status": "ok" }
```

---

## 4 Visual Stimuli

| ID | Event | What is measured |
|---|---|---|
| `flood` | Enchentes RS (2024) | Collective loss recognition |
| `covid` | COVID-19 Isolation | Pandemic attachment rupture |
| `displacement` | Forced Displacement | Migratory empathy response |
| `conflict` | Geopolitical Conflict | War/violence threat activation |

---

## Useful Links

- [Solana Playground](https://beta.solpg.io/)
- [Devnet Faucet](https://faucet.solana.com)
- [Solscan Devnet](https://solscan.io?cluster=devnet)
- [Solana Explorer Devnet](https://explorer.solana.com?cluster=devnet)
- [Metaplex Docs](https://developers.metaplex.com)
- [Memo Program](https://spl.solana.com/memo)

---

## License

MIT

