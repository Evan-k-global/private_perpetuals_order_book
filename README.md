# Private Perpetuals Order Book

ShadowBook is a private perpetuals / dark pool demo built on Zeko with:
- note-backed private collateral
- public and private order visibility modes
- off-chain matching
- on-chain settlement anchoring through a zkApp
- proof-precompute support for keeping settlement off the hot path

## What Works

- wallet sync from on-chain balances
- deposit -> private note mint
- limit and market orders
- public book and private off-book orders
- withdrawals back to wallet
- settlement batching and zkApp root commits
- local operator tooling and proof inspection

## Quick Start

```bash
cd /Users/evankereiakes/Documents/Codex/private-order-book
pnpm install
cp .env.example .env
pnpm build:zkapp
pnpm darkpool:serve
```

Open:
- [Trading UI](http://127.0.0.1:8791/darkpool)
- [Partner Frontend Example](http://127.0.0.1:8791/partner)

## Recommended Runtime Modes

### Local dev

Run the API/UI only:

```bash
pnpm darkpool:serve
```

### Demo / hosted single service

Run the server with embedded proof + settlement background loops:

```bash
pnpm render:start
```

### Split proving later

If you want extra proof capacity later, keep the main service running and add:

```bash
pnpm settlement:worker:proofs:remote
```

That remote proof agent fetches the next pending proof job snapshot, builds the proof locally, and uploads the proof artifact back.

## Key Docs

- [Render deployment and runtime topology](./docs/deployment.md)
- [Protocol runbook and local operations](./docs/runbook.md)
- [API and SDK reference](./docs/api.md)
- [Private-state proof roadmap](./docs/private-state-proof-plan.md)

## System Model

- client:
  - wallet connect
  - deposit signing
  - off-chain order authorization
  - UI rendering
- market server:
  - notes
  - matching
  - batching
  - APIs
- proof path:
  - precompute proofs ahead of commit
  - optionally on a separate machine
- settlement path:
  - payout handling
  - zkApp commit

Short version:
- matching stays fast and off-chain
- proving stays off the request path
- chain is used for settlement verification and root anchoring

## Notes

- This repo is demo-ready, not final trust-minimized production infrastructure.
- The top-level README stays intentionally high level; detailed ops live in `docs/`.
