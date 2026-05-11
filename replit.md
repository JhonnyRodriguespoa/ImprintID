# ImprintID — Proof of Experience Protocol on Solana

A behavioral identity protocol that transforms emotional responses to visual stimuli into verifiable on-chain data. Responses are SHA-256 hashed, uploaded to IPFS, anchored on Solana Devnet (Anchor PDA or Memo Program fallback), **and minted as a Metaplex NFT** on Devnet carrying all ISS research attributes.

## Run & Operate

- `pnpm --filter @workspace/imprintid run dev` — run frontend (port 19933)
- `pnpm --filter @workspace/api-server run dev` — run API server (port 8080)
- `pnpm run typecheck` — full typecheck across all packages
- `pnpm run build` — typecheck + build all packages

**Env vars:**
- `VITE_PROGRAM_ID` — deployed Anchor program ID (enables custom program path; falls back to Memo Program when unset)
- `VITE_SOLANA_RPC` — custom Solana RPC endpoint (defaults to devnet public)
- `VITE_WEB3_STORAGE_KEY` — web3.storage API key for real IPFS (falls back to mock URI)
- `SESSION_SECRET` — session secret for API server

## NFT Minting (Metaplex — Devnet) — Server-side, no user SOL required

After experiment completion the app automatically:
1. Builds Metaplex-standard metadata JSON (name, symbol, image, 12 attributes)
2. Uploads it to IPFS (or mock URI if `VITE_WEB3_STORAGE_KEY` is unset)
3. Calls `POST /api/nft/mint` → API server mints NFT via Metaplex UMI with **server keypair** (no Phantom signing)
4. NFT token owner is set to the participant's wallet address — they receive the NFT without paying
5. API server also anchors the behavioral hash via Memo Program (server pays gas)
6. Confirms on Devnet (~0.015 SOL per mint, paid by server)

NFT minting and anchoring are **non-fatal**: if the server wallet is unfunded, behavioral data is still hashed/saved locally and the Dashboard shows a fallback message.

**Participants need zero SOL.** Only the server wallet needs Devnet SOL.

**NFT attributes on-chain**: ISS Score, Perfil de Apego, Intensidade Emocional, Taxa de Associação, Delta Emocional, Variância, Gatilho Dominante, Tipo de Resposta, Variante A/B, Protocolo, Rede.

## Server Wallet — Fund Once

The server wallet is derived deterministically from `SESSION_SECRET` (stable across restarts).

**Current server wallet address**: `4RnmsHgZPhKCzNEp9Gsd2Nmj98HGAdwZYnkntHviRuj9`

To enable on-chain minting, fund this address with 5+ SOL on Devnet:
1. Go to **https://faucet.solana.com**
2. Paste `4RnmsHgZPhKCzNEp9Gsd2Nmj98HGAdwZYnkntHviRuj9`
3. Request 5 SOL
4. That covers ~300+ NFT mints (no server restart needed)

Check balance at any time: `GET /api/nft/wallet`

Alternatively, set `SOLANA_SERVER_KEYPAIR` to a JSON secret-key array to use a pre-funded wallet.

## Solana Playground Deployment

The custom Anchor program lives at `artifacts/imprintid/src/lib/program.rs`.  
It stores one behavioral-identity PDA per wallet: `seeds = ["imprint", owner_pubkey]`.

### Step-by-step

1. Open **https://beta.solpg.io/** → "Create a new project" → select **Anchor** framework
2. Replace the contents of `lib.rs` with `artifacts/imprintid/src/lib/program.rs`
3. Click **Build** — wait for compilation to succeed
4. Click **Deploy** → confirm on Devnet → note the **Program ID** shown
5. Click **Export → IDL** → download the JSON file
6. **Replace** `artifacts/imprintid/src/lib/idl.json` with the downloaded IDL
7. In Replit **Secrets**, set `VITE_PROGRAM_ID = <your_program_id>`
8. Restart the `artifacts/imprintid: web` workflow

The app automatically switches from Memo Program to the Anchor program once `VITE_PROGRAM_ID` is detected.

### On-chain account structure

| Field               | Type     | Max     | Description                              |
|---------------------|----------|---------|------------------------------------------|
| `owner`             | Pubkey   | —       | Signer wallet                            |
| `hash`              | String   | 64 chars| SHA-256 of full behavioral payload       |
| `uri`               | String   | 100 chars| IPFS URI of the full JSON payload        |
| `iss`               | u8       | 0–100   | Imprint Sensitivity Score                |
| `attachment_profile`| String   | 16 chars| seguro / ansioso / evitativo / desorganizado |
| `variant`           | String   | 2 chars | A/B test variant shown                   |
| `timestamp`         | i64      | —       | Unix timestamp (seconds)                 |
| `bump`              | u8       | —       | PDA canonical bump                       |

**Account space**: 248 bytes + 8 discriminator = 256 bytes total  
**PDA seeds**: `["imprint", owner_pubkey]` — one account per wallet, deterministically queryable

## Stack

- **Monorepo**: pnpm workspaces, Node.js 24, TypeScript 5.9
- **Frontend**: React + Vite, Tailwind v4, Framer Motion
- **Solana**: `@solana/wallet-adapter-react`, `@coral-xyz/anchor` ^0.32, Phantom-only
- **On-chain (primary)**: Custom Anchor program via Solana Playground — PDA per wallet, queryable without index
- **On-chain (fallback)**: Solana Memo Program (`MemoSq4gqABAXKb96qnH8TysNcWxMyWCqXgDLGmfcHr`) — no deploy needed
- **Data**: SHA-256 hash (crypto-js) + web3.storage IPFS upload
- **API**: Express 5, PostgreSQL + Drizzle ORM, Zod

## Where things live

```
artifacts/imprintid/src/
  pages/          Landing, Connect, Experiment, Dashboard
  components/     WalletButton, StimulusVisual, ScaleSlider
  lib/
    program.rs    Anchor program source → paste into Solana Playground
    idl.json      IDL template → replace with Playground export after deploy
    anchor.ts     Anchor client: hasProgramId(), getImprintPDA(), createImprintAnchor(), fetchImprint()
    solana.ts     Memo Program fallback + airdrop helpers
    iss.ts        Imprint Sensitivity Score + attachment profile classification
    hash.ts       SHA-256 hashing
    ipfs.ts       IPFS upload (web3.storage or mock)
```

## Architecture decisions

- **Dual-path anchoring**: when `VITE_PROGRAM_ID` is set the app calls `createImprintAnchor()` (Anchor PDA); otherwise it falls back to `createImprintTransaction()` (Memo Program). No code change needed to switch.
- **PDA design**: `seeds = ["imprint", owner_pubkey]` — each wallet has exactly one imprint, deterministically addressable. No off-chain index required to look up a user's record.
- **ISS (Imprint Sensitivity Score)**: derived from scale intensities (Ansiedade/Tristeza/Medo/Conexão), personal associations, reaction time, and variance. Range 0–100 mapped to 4 attachment profiles (Seguro/Ansioso/Evitativo/Desorganizado) via Bowlby/Ainsworth thresholds.
- **A/B testing**: variant (A = color photo, B = grayscale+data) randomly assigned per session and recorded in the payload and on-chain account.
- **Navigation**: state-based (no router) — "landing" | "connect" | "experiment" | "dashboard". Wallet gate triggers only when user attempts to operate.
- **Mock fallback**: app runs in mock mode (fake IPFS URI, Memo Program) when env vars are not set. No Phantom required for development.
- **Airdrop built-in**: auto-detects low balance before experiment submission; also available from WalletButton dropdown.
- **localStorage persistence**: last experiment result is saved to localStorage so the Dashboard survives page refreshes.

## Product

- **Landing**: public marketing page (no wallet required)
- **Experiment**: 4-stimulus protocol (Enchentes RS, COVID-19, Deslocamento Populacional, Conflito Geopolítico) with ISS scoring and A/B variant
- **Dashboard**: ISS gauge, attachment profile card, research variables grid (E/A/T/D/V), per-stimulus breakdown, on-chain tx links
- **Connect**: Phantom-only wallet gate shown only when user clicks an action

## User preferences

- Site must be publicly visible without wallet connection
- Wallet only requested when user tries to operate (experiment / dashboard)
- Devnet only; Phantom wallet only
- UI in Portuguese

## Gotchas

- Phantom must be set to **Devnet** in wallet settings for real transactions
- If balance is 0, the airdrop screen appears automatically before submission
- Anchor `createImprint` will fail if the PDA already exists (one imprint per wallet). To re-run, the program would need an `update_imprint` instruction or the account must be closed first.
- After deploying from Playground, you MUST replace `idl.json` with the Playground export — discriminators in the IDL must match the compiled program
- `web3.storage` v1 API is legacy; if IPFS upload fails the app uses a mock URI and continues

## Pointers

- Solana Playground: https://beta.solpg.io/
- Solana Memo Program: `MemoSq4gqABAXKb96qnH8TysNcWxMyWCqXgDLGmfcHr`
- Devnet faucet: https://faucet.solana.com
- Solscan Devnet: https://solscan.io?cluster=devnet
