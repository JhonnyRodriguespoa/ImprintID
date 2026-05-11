# Contributing to ImprintID

First of all, thank you for your interest in contributing to **ImprintID**.

ImprintID is building a new primitive for Web3: **Proof of Experience** — a protocol that transforms human interactions into verifiable behavioral identity on Solana.

We welcome contributions from:

- Solana developers
- Frontend engineers
- Backend engineers
- Researchers
- Designers
- Documentation contributors
- DeSci collaborators

---

# Ways to Contribute

You can help by:

- fixing bugs
- improving documentation
- refining protocol logic
- building new integrations
- improving UX/UI
- expanding test coverage
- proposing new use cases
- reviewing pull requests

---

# Development Setup

## 1. Fork the repository

Click **Fork** on GitHub.

---

## 2. Clone your fork

```bash
git clone https://github.com/YOUR_USERNAME/ImprintID.git
cd ImprintID
```

---

## 3. Install dependencies

Using pnpm:

```bash
pnpm install
```

Or npm:

```bash
npm install
```

---

## 4. Configure environment variables

Copy:

```bash
cp .env.example .env
```

Set required values:

```env
SOLANA_RPC_URL=
WALLET_PRIVATE_KEY=
PINATA_JWT=
```

---

## 5. Run locally

```bash
pnpm dev
```

---

# Branch Naming Convention

Please create descriptive branches:

```bash
feature/add-wallet-adapter
fix/nft-metadata-bug
docs/update-readme
refactor/api-validation
```

---

# Commit Guidelines

Use clear commit messages:

```bash
feat: add compressed NFT minting
fix: resolve wallet connection issue
docs: improve setup instructions
refactor: simplify behavioral hash engine
```

Recommended prefixes:

- `feat`
- `fix`
- `docs`
- `refactor`
- `test`
- `chore`

---

# Pull Request Process

1. Create your branch
2. Make changes
3. Run tests
4. Update documentation if needed
5. Commit changes
6. Push branch
7. Open Pull Request

---

# Coding Standards

## Frontend

- TypeScript preferred
- Reusable components
- Clean state management
- Mobile responsive

---

## Backend

- Clear API contracts
- Strong validation
- Error handling
- Minimal dependencies

---

## Solana Program

- Secure account validation
- Deterministic PDA usage
- Efficient compute usage
- Clear instruction documentation

---

# Documentation Standards

Good documentation matters.

Please keep:

- concise explanations
- reproducible steps
- architecture diagrams when useful
- code comments where necessary

---

# Security

Do **not** submit:

- private keys
- seed phrases
- `.env` files
- production credentials

If you discover a security issue, please contact the maintainers privately.

---

# Code of Conduct

Be respectful.

We value:

- openness
- collaboration
- curiosity
- scientific rigor
- human-centered innovation

---

# Project Vision

ImprintID is more than an application.

We are building infrastructure for:

- behavioral identity
- decentralized trust
- human-centered AI
- Proof of Experience

Thank you for helping build the future of Web3 identity.

---

# Questions?

Open an issue or start a discussion in the repository.

We welcome your contribution.
