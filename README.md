# Private Order Book (Dark Pool) Demo

Standalone demo for:
- private order placement (commitment + encrypted payload)
- private execution (public tape hides participant identities)
- real-funds-only wallet sync (on-chain balances -> trading balances)
- real-funds-backed privacy notes (mint from synced balances, redeem for private trade funding)
- funded collateral locking on limit orders (lock on place; unlock on fill/cancel/expiry)
- wallet-derived blinded account IDs (`accountId = hash(salt + wallet)`)

## Run

```bash
cd /Users/evankereiakes/Documents/Codex/private-order-book
pnpm darkpool:serve
```

Open:
- http://127.0.0.1:8791/darkpool
- http://127.0.0.1:8791/partner (example independent frontend using SDK)

## Protocol Live Runbook (On-chain Settlement)

1. install deps once:

```bash
pnpm install
```

2. set env vars (copy from `.env.example`):

```bash
cp .env.example .env
```

3. deploy zkApp (one-time for a fresh contract):

```bash
source .env
pnpm zkapp:deploy
```

4. start protocol API:

```bash
source .env
pnpm darkpool:serve
```

5. in a second terminal, start DA relay (for `DA_MODE=zeko-relay`):

```bash
source .env
pnpm da:relay
```

6. in a third terminal, start the proof-precompute worker:

```bash
source .env
pnpm settlement:worker:proofs
```

7. in a fourth terminal, start settlement worker (on-chain mode):

```bash
source .env
pnpm settlement:worker:onchain
```

Worker note:
- the proof worker precomputes the next pending batch proof ahead of commit time
- the settlement worker now waits for that cached proof before payouts/commit
- both workers should run against prebuilt `dist-zkapp` artifacts in hosted mode, not `pnpm build` on every loop
- the live zkApp commit path returns `localBatchId`, `onchainBatchId`, and `txHash`
- the worker uses `localBatchId` to mark the local batch committed after the on-chain commit succeeds

7. sync funded testnet wallets into engine balances (real-funds mode):

```bash
curl -X POST http://127.0.0.1:8791/api/darkpool/accounts/sync-onchain \
  -H "content-type: application/json" \
  -d '{"wallet":"B62qbot_maker_wallet"}'
curl -X POST http://127.0.0.1:8791/api/darkpool/accounts/sync-onchain \
  -H "content-type: application/json" \
  -d '{"wallet":"B62qbot_taker_wallet"}'
curl -X POST http://127.0.0.1:8791/api/darkpool/accounts/sync-onchain \
  -H "content-type: application/json" \
  -d '{"wallet":"B62qqpyJPDGci2uxpapnXQmrFr77b47wRx1v2GDRnAHMUFJFjJv4YPb"}'
```

9. in a fifth terminal, run continuous live orderflow bot:

```bash
source .env

## Render Deployment

For the demo-scale target you described, you can host this as a single Render web service.

Use:
- Build command: `pnpm install --frozen-lockfile && pnpm build:zkapp`
- Start command: `pnpm render:start`

Set these env vars on Render:
- `DARKPOOL_HOST=0.0.0.0`
- `AUTO_RUN_BACKGROUND_WORKERS=true`
- `REQUIRE_CACHED_PRIVATE_STATE_PROOF=true`
- `ALLOW_INLINE_PRIVATE_STATE_PROVING=false`
- `PROOF_WORKER_API_KEY=<strong-random-secret>`

What that gives you:
- one public web service for UI + API + matcher
- one embedded proof-precompute loop supervised by the server
- one embedded settlement loop supervised by the server
- no separate Render worker services required for the demo

If you later need extra proving capacity:
- keep the Render web service as-is
- run `pnpm settlement:worker:proofs:remote` on another machine
- point it at the same `DARKPOOL_API`
- set the same `PROOF_WORKER_API_KEY`

The remote proof agent:
- fetches the next pending proof job snapshot from the market service
- builds the proof locally from that snapshot
- uploads the proof artifact back to the market service

That lets you scale proving horizontally while keeping settlement commit single-writer.

Recommended for Render:
- keep `DA_RELAY_*` disabled unless you are explicitly demoing relay mode
- keep the database/state on durable disk or move state into Postgres before higher-volume use
pnpm bot:arb
```

9. (optional) run one-shot replay flow:

```bash
pnpm replay:demo
```

10. check health + fairness:

```bash
curl http://127.0.0.1:8791/api/darkpool/status
curl "http://127.0.0.1:8791/api/darkpool/fairness/audit?limit=200"
curl http://127.0.0.1:8787/health
```

## Core APIs

- `GET /api/darkpool/markets`
- `GET /api/darkpool/book?marketId=...&levels=20` (preferred)
- `GET /api/darkpool/book?pair=tETH/tZEKO&levels=20` (legacy)
- `GET /api/darkpool/book/hash?marketId=...` (book integrity hash)
- `POST /api/darkpool/accounts/sync-onchain` (real-funds mode)
- `GET /api/darkpool/accounts/balance?wallet=...`
- `GET /api/darkpool/accounts/onchain-diagnostics?wallet=...`
- `GET /api/darkpool/accounts/pretrade?wallet=...&marketId=...&side=BUY|SELL&orderType=LIMIT|MARKET&quantity=...&limitPrice=...`
- `GET /api/darkpool/activity?wallet=...&limit=150`
- `GET /api/darkpool/frontends/fees?frontendId=partner.alpha`
- `GET /api/darkpool/status`
- `GET /api/darkpool/fairness/audit?limit=200`
- `POST /api/darkpool/orders/place`
- `POST /api/darkpool/orders/:id/cancel`
- `GET /api/darkpool/orders/:id?token=...`
- `POST /api/darkpool/maker/quote` (`x-maker-key` required)
- `POST /api/darkpool/maker/cancel-all` (`x-maker-key` required)
- `GET /api/darkpool/trades`
- `GET /api/darkpool/settlement/batches`
- `POST /api/darkpool/settlement/mark-committed`
- `GET /api/darkpool/settlement/payout-requirements?batchId=...`
- `POST /api/darkpool/settlement/commit-next-local`
- `POST /api/darkpool/settlement/anchor-book` (enqueue book-hash anchor batch)
- `POST /api/darkpool/vault/deposit` (mint privacy note from synced available balance; requires `txHash` when `REQUIRE_ONCHAIN_DEPOSIT_TX=true`)
- `POST /api/darkpool/vault/deposit/find-latest` (find latest eligible deposit tx hash for wallet/token/amount)
- `POST /api/darkpool/vault/deposit-auto` (mint note by auto-selecting latest eligible deposit tx)
- `POST /api/darkpool/notes/redeem` (redeem privacy note to trading balance)
- `GET /api/darkpool/vault/pool`

`POST /api/darkpool/orders/place` accepts optional:
- `frontendId`: `3-40` chars `[a-z0-9._-]`
- `marketId` (preferred unique market identifier)
- `baseTokenId` + `quoteTokenId` (alternative unique selector)

Order placement response now includes a signed server receipt:
- `orderReceipt`
  - includes order digest fields + issued timestamp
  - `signature = sha256(ORDER_RECEIPT_SECRET | canonical-payload)`

## SDK + Multi-Frontend Liquidity

The backend is now a shared protocol endpoint with CORS enabled, so any frontend can connect and route into the same book/liquidity.

SDK path:
- `/sdk/shadowbook-sdk.js`

Example:

```html
<script src="https://your-shadowbook-node/sdk/shadowbook-sdk.js"></script>
<script>
  const sdk = window.ShadowBookSDK.createClient({
    baseUrl: 'https://your-shadowbook-node',
    frontendId: 'partner.alpha'
  });

  await sdk.placeOrder({
    wallet: 'B62q...',
    pair: 'tETH/tZEKO',
    side: 'BUY',
    timeInForce: 'GTC',
    limitPrice: 64000,
    quantity: 0.05,
    privateMemo: 'routed by partner'
  });

  const feeStats = await sdk.getFrontendFees();
</script>
```

Fee routing model in this demo:
- taker fee: `TAKER_FEE_BPS` (default `5` bps)
- frontend revenue share: `FRONTEND_FEE_SHARE_BPS` (default `3000` = `30%` of taker fee)
- remaining fee goes to protocol fee balances.

Operational SDK methods:
- `getStatus()`
- `getAudit(limit)`
- `getSettlementBatches(limit)`
- `markBatchCommitted({ batchId, txHash })`
- `commitNextLocal()`

## zkApp Wiring (On-chain Settlement Roots)

This repo now includes a settlement zkApp under `zkapp/`:
- `zkapp/contract.ts` (`ShadowBookSettlementZkApp`)
- rolling on-chain `settlementRoot`
- anchored `bookRoot`
- anchored `noteRoot`
- anchored `nullifierRoot`
- anchored `sequencingRoot`
- monotonic `lastBatchId`
- `batchCommitted` event

Local matching now writes settlement batches to:
- `data/settlement-batches.json`

When fills happen, a `pending` batch is enqueued with:
- `batchId`
- `batchHash`
- trades included
- aggregated payout obligations by wallet/token (`payouts[]`)

### Scripts

- `pnpm bot:arb` (continuous maker+taker bot for live chart/orderflow)
- `pnpm da:relay` (local DA relay for `DA_MODE=zeko-relay`)
- `pnpm settlement:worker` (poll + commit pending batches)
- `pnpm settlement:payout:template` (print payout tx proof template for next pending batch)
- `pnpm settlement:payout:auto` (auto-sign/send payout txs for next pending batch)
- `pnpm replay:demo` (seed/fund/place orders + print protocol summary)
- `pnpm build:zkapp`
- `pnpm zkapp:deploy`
- `pnpm zkapp:commit-next`
- `pnpm zkapp:get-state`
- `pnpm zkapp:inspect-private-state-merkle`
- `pnpm zkapp:prove-private-state`
- `pnpm settlement:worker:proofs`

`pnpm zkapp:commit-next` now follows the lean verified path:
- build next pending batch proof inputs
- load a cached private-state proof by default and refuse inline proving unless explicitly enabled
- verify that proof inside the settlement zkApp before anchoring the next roots
- prove the journaled note/nullifier delta against real Merkle witnesses, rather than only hashed aggregate summaries

`pnpm zkapp:inspect-private-state-merkle` inspects the live engine state and prints
the current Merkle-backed snapshot for:
- active notes
- spent nullifiers
- sequencing receipts

The private-state proof surface stays intentionally small:
- only touched note/nullifier entries are proven per batch
- matching stays off-chain
- order entry stays immediate
- proving is precomputed asynchronously before batch commit

Operator APIs:
- `GET /api/darkpool/operator/zkapp-state`
- `GET /api/darkpool/operator/private-state-merkle`
- `GET /api/darkpool/operator/private-state-witness`
- `POST /api/darkpool/operator/private-state-proof`

### Settlement worker modes

- `SETTLEMENT_MODE=zkapp` (default): runs `pnpm zkapp:commit-next` and then marks batch committed on API.
- `SETTLEMENT_MODE=local` (testing only): commits pending batches using local simulated tx hash.

Worker env vars:
- `DARKPOOL_API` (default `http://127.0.0.1:8791`)
- `SETTLEMENT_MODE` (`local` or `zkapp`)
- `SETTLEMENT_INTERVAL_MS` (default `6000`)
- `SETTLEMENT_REQUIRE_PAYOUT_PROOFS` (default `true`)
- `SETTLEMENT_ALLOW_UNVERIFIED_PAYOUTS` (default `false`)
- `SETTLEMENT_PAYOUT_COMMAND` (defaults to `node scripts/settlement-payout-executor.js`)
- `PAYOUT_OPERATOR_PRIVATE_KEY` (required for auto payout signing; must map to `VAULT_DEPOSIT_ADDRESS`)
- `PAYOUT_FEE_PAYER_PRIVATE_KEY` (optional separate fee payer)
- `PRIVATE_STATE_PROOF_INTERVAL_MS` (default `4000`)
- `PRIVATE_STATE_PROVER_MAX_OLD_SPACE_MB` (default `4096`)
- `PRIVATE_STATE_PROOF_COMMAND` (default `node --enable-source-maps dist-zkapp/prove-private-state-batch.js`)
- `REQUIRE_CACHED_PRIVATE_STATE_PROOF` (default `true`)
- `ALLOW_INLINE_PRIVATE_STATE_PROVING` (default `false`; demo default is to keep this off)
- `ZKAPP_COMMIT_COMMAND` (default `node --enable-source-maps dist-zkapp/commit-next-batch.js`)

### Required env vars (deploy/commit)

- `ZEKO_GRAPHQL`
- `ZEKO_TX_GRAPHQL` (optional; used for tx-hash verification if `ZEKO_GRAPHQL` does not expose tx query fields)
- `ZEKO_ARCHIVE_GRAPHQL` (optional alias; used as tx verification fallback source)
- `ZEKO_ARCHIVE_RELAY_GRAPHQL` (optional alias; highest-priority tx verification fallback source)
- `ZEKO_NETWORK_ID` (optional; currently not used by scripts)
- tx verification endpoint priority:
  - `ZEKO_TX_GRAPHQL`
  - `ZEKO_ARCHIVE_RELAY_GRAPHQL`
  - `ZEKO_ARCHIVE_GRAPHQL`
  - `ZEKO_GRAPHQL`
- `DEPLOYER_PRIVATE_KEY`
- `ZKAPP_PRIVATE_KEY` (deploy)
- `ZKAPP_PUBLIC_KEY` (commit/get-state)
- `TX_FEE` (optional, default `100000000`)
- `SETTLEMENT_BATCHES_FILE` (optional override for batch file path)

### Runtime env vars (server)

- `ORDER_RECEIPT_SECRET` (receipt signing secret)
- `TAKER_FEE_BPS`
- `FRONTEND_FEE_SHARE_BPS`
- `AUTO_SETTLEMENT` (`true`/`false`)
- `AUTO_SETTLEMENT_INTERVAL_MS`
- `MAKER_API_KEY`
- `ENABLE_LOCAL_SETTLEMENT` (`true`/`false`, default `false`)
- `ONCHAIN_SYNC_TTL_MS` (default `60000`)
- `REQUIRE_ONCHAIN_DEPOSIT_TX` (`true` by default; enforces signed on-chain deposit tx hash for note mint)
- `VAULT_DEPOSIT_ADDRESS` (required when `REQUIRE_ONCHAIN_DEPOSIT_TX=true`)
- `SETTLEMENT_REQUIRE_ONCHAIN_PAYOUTS` (`true` by default; requires payout tx proofs before batch commit)
- `GTC_ORDER_EXPIRY_MS` (`0` disables automatic expiry; >0 auto-cancels stale GTC and unlocks collateral)
- `ASSET_DECIMALS_JSON` (for converting on-chain integer balances)
- `ORDER_STATE_ENCRYPTION_KEY` (encrypt persisted engine state at rest)
- `BOOK_ANCHOR_INTERVAL_MS` (`0` disables; >0 enqueues periodic book-hash anchor batches)
- `DA_MODE` (`http-json` or `zeko-relay`)
- `DA_ENDPOINT`, `DA_BEARER_TOKEN` (optional DA publish hook for encrypted book anchor payloads)
- `DA_REQUIRE_ENCRYPTION` (default `true`)
- `DA_INCLUDE_ORDER_SNAPSHOT` (default `false`, keep false for tighter privacy)
- `DA_ENCRYPTION_KEY` (encryption key for DA payloads)
- `ZEKO_DA_NETWORK`, `ZEKO_DA_APP_ID`, `ZEKO_DA_SCHEMA` (metadata used in `zeko-relay` mode)
- `DA_RELAY_PORT` (default `8787`)
- `DA_RELAY_FORWARD_MODE` (`none`, `zeko-bridge`, `command`)
- `ZEKO_DA_BRIDGE_URL`, `ZEKO_DA_BRIDGE_TOKEN` (used when `DA_RELAY_FORWARD_MODE=zeko-bridge`)
- `ZEKO_DA_NAMESPACE`, `ZEKO_DA_REQUIRE_FORWARD`, `DA_RELAY_FORWARD_RETRIES`, `DA_RELAY_FORWARD_TIMEOUT_MS`
- `DA_RELAY_COMMAND` (used when `DA_RELAY_FORWARD_MODE=command`)
- `DA_RELAY_SECRET` (relay receipt signing key)

## Real Funds Only (Testnet)

ShadowBook now runs in real-funds-only mode:

- `POST /api/darkpool/accounts/fund` is disabled
- `POST /api/darkpool/vault/withdraw` is disabled (direct on-chain note withdrawal not wired yet)
- place order requires a recent on-chain sync (`/api/darkpool/accounts/sync-onchain`)

Privacy note mint flow in real funds mode:
1. Connect wallet and sync balances (`/api/darkpool/accounts/sync-onchain`).
2. Send an on-chain token transfer from your wallet to `VAULT_DEPOSIT_ADDRESS` (wallet signs this tx).
3. Mint note with `POST /api/darkpool/vault/deposit` including:
   - `wallet`
   - `asset` or `tokenId`
   - `amount`
   - `txHash` (signed deposit transfer hash)
4. Server verifies sender, recipient, token, and amount against chain data before issuing note.

Token deposit handling notes:
- `tMINA` uses the native wallet payment flow.
- `tETH` / `tZEKO` use a Mina fungible-token zkApp transfer flow.
- For Auro compatibility, ShadowBook asks the wallet to `onlySign` FT deposit transactions as stringified zkApp JSON, then the backend submits the signed zkApp command with the Zeko `sendZkapp` GraphQL mutation.
- ShadowBook follows the same practical Auro guard used by Lumina: if the built transaction fee payer body fee is `"0"`, it is bumped to `"1"` before wallet submission to avoid a wallet-side zero-fee rejection edge case.
- Contract addresses for supported FT assets are configured through `TOKEN_CONTRACT_ADDRESSES_JSON`.

Settlement flow in real funds mode (on-chain tied):
1. Matching engine creates a pending settlement batch with payout obligations per wallet/token.
2. Settlement worker runs payout command (`SETTLEMENT_PAYOUT_COMMAND`) to sign/send payout txs and collect hashes.
3. Settlement worker commits batch hash on zkApp.
4. `POST /api/darkpool/settlement/mark-committed` includes `payoutTxs` proofs when required.
5. Server verifies each payout tx hash on-chain:
   - sender is `VAULT_DEPOSIT_ADDRESS`
   - recipient wallet matches payout
   - token ID matches payout token
   - amount is sufficient
6. Fill proceeds are treated as on-chain payouts (not instant internal note credits).

Secure-by-default recommendation:
1. Use a dedicated payout vault key for `PAYOUT_OPERATOR_PRIVATE_KEY` (not deployer key).

## Private State Proof Roadmap

Fast-path design target:
- order entry stays off-chain
- matching stays off-chain
- note selection stays off-chain
- proofs run asynchronously per batch
- on-chain writes remain settlement/state-root commits only

Proof-layer artifacts:
- typed private-state skeleton:
  - `/Users/evankereiakes/Documents/Codex/private-order-book/zkapp/private-state.ts`
- prover entrypoint:
  - `/Users/evankereiakes/Documents/Codex/private-order-book/zkapp/private-state-prover.ts`
- witness/proof builders:
  - `/Users/evankereiakes/Documents/Codex/private-order-book/zkapp/build-private-state-witness.ts`
  - `/Users/evankereiakes/Documents/Codex/private-order-book/zkapp/prove-private-state-batch.ts`
- design notes:
  - `/Users/evankereiakes/Documents/Codex/private-order-book/docs/private-state-proof-plan.md`

Near-term proving goal:
- prove note spends / nullifier updates / output notes per settlement batch
- anchor sequencing receipts and note-root transitions without slowing order entry
- keep the current fast matcher and current public/private order visibility model
2. Keep `SETTLEMENT_REQUIRE_PAYOUT_PROOFS=true` and `SETTLEMENT_ALLOW_UNVERIFIED_PAYOUTS=false`.
3. Use a separate fee payer key in `PAYOUT_FEE_PAYER_PRIVATE_KEY`.
4. Keep signer process isolated from public frontend hosts.
- sync reads wallet balances by token ID from `ZEKO_GRAPHQL`
- sync applies on-chain balance deltas and enforces locked protocol collateral constraints (open orders + outstanding notes)
- limit order collateral is locked at order placement and only unlocked on fill, cancel, or expiry
- note minting is real-funds-backed: minting a note deducts synced available balance

Example:

```bash
curl -X POST http://127.0.0.1:8791/api/darkpool/accounts/sync-onchain \
  -H "content-type: application/json" \
  -d '{"wallet":"B62q..."}'
```

## Persistence + Integrity

- Open orders now persist across server restarts in `data/engine-state.json`
- If `ORDER_STATE_ENCRYPTION_KEY` is set, persisted engine state is encrypted (AES-256-GCM)
- Book hash endpoint:
  - `GET /api/darkpool/book/hash?marketId=...`
- Book hash anchoring:
  - Manual: `POST /api/darkpool/settlement/anchor-book`
  - Automatic: set `BOOK_ANCHOR_INTERVAL_MS` and run settlement worker to commit anchor batches on-chain

Anchor flow:
1. Engine computes canonical book hash
2. Hash is enqueued as pending settlement batch (`batchType=book_anchor`)
3. Existing worker commits it on-chain via `zkapp:commit-next`

## Zeko DA Integration

ShadowBook supports a Zeko-oriented DA publishing mode:

- `DA_MODE=zeko-relay`
- `DA_ENDPOINT=http://127.0.0.1:8787/publish` (local relay endpoint)

Why relay mode:
- Zeko DA nodes expose Async RPC (`Post_diff`, etc.), not plain HTTP JSON.
- ShadowBook publishes encrypted anchor payloads to the local relay.
- Relay can forward to an external Zeko DA HTTP bridge or command adapter.
- In strict chain mode (`DA_RELAY_FORWARD_MODE=zeko-bridge` + `ZEKO_DA_REQUIRE_FORWARD=true`) publish is rejected if upstream DA forwarding fails.

Recommended config:

```bash
DA_MODE=zeko-relay
DA_ENDPOINT=http://127.0.0.1:8787/publish
DA_REQUIRE_ENCRYPTION=true
DA_INCLUDE_ORDER_SNAPSHOT=false
DA_ENCRYPTION_KEY=<strong-random-key>
ZEKO_DA_NETWORK=testnet
ZEKO_DA_APP_ID=shadowbook
ZEKO_DA_SCHEMA=shadowbook.da.v1
# relay forwarding mode
DA_RELAY_FORWARD_MODE=zeko-bridge
ZEKO_DA_BRIDGE_URL=https://<your-zeko-da-bridge>/publish
ZEKO_DA_REQUIRE_FORWARD=true
DA_RELAY_FORWARD_RETRIES=3
DA_RELAY_FORWARD_TIMEOUT_MS=15000
```

Relay endpoints:
- `POST /publish`
- `GET /health`
- `GET /records?limit=50`

### Arbitrage/Orderflow bot env vars

- `DARKPOOL_API` (default `http://127.0.0.1:8791`)
- `BOT_PAIR` (default `tETH/tZEKO`)
- `BOT_MARKET_ID` (preferred, if set)
- `BOT_BASE_TOKEN_ID`, `BOT_QUOTE_TOKEN_ID` (token-id market resolution)
- `BOT_MAKER_WALLET`, `BOT_TAKER_WALLET`
- `BOT_LOOP_MS` (default `2200`)
- `BOT_QUOTE_SPREAD_BPS`, `BOT_QUOTE_SIZE`
- `BOT_TAKER_PROB` (probability of IOC taker each tick)
- `BOT_TAKER_SIZE_MIN`, `BOT_TAKER_SIZE_MAX`
- `BOT_MISPRICE_BPS`, `BOT_EXTERNAL_VOL_BPS`
- `BOT_AUTO_FUND` (`true`/`false`, default `false`)
- `BOT_REAL_FUNDS` (`true`/`false`, default `true`; bot syncs balances from chain via `/accounts/sync-onchain`)
- `BOT_AUTO_FUND_*` / `BOT_MIN_*` (maker+taker pool management thresholds)

## Privacy Model

- Default public surface: market commitments, batch hashes, and audit/settlement metadata.
- DA payload privacy: payloads are encrypted before publish when `DA_REQUIRE_ENCRYPTION=true`.
- Best-practice default is commitment-only DA (`DA_INCLUDE_ORDER_SNAPSHOT=false`), so no order details are published in cleartext.
- This still leaks timing/volume metadata at the system level (as with most DA systems), even with encrypted payloads.
- User balances are not fully private in this app-layer design because settlement and token balances are observable at L1 interfaces.
- Additional balance privacy requires an encrypted rollup / encrypted state model where balances, transfers, and matching state are hidden in-circuit and only validity proofs + minimal commitments are public.

## Notes

- This is application-layer privacy, not a fully encrypted sovereign rollup.
- Fairness audit chain is written to `data/fairness-audit.jsonl` (append-only hash-linked records).
- `indicativeMid` is computed from live best bid/ask.
- `referencePrice` auto-follows `indicativeMid` when the book has liquidity and falls back to the last anchor otherwise.
- Set `MAKER_API_KEY` in env for maker endpoint auth (defaults to `demo-maker-key` for local demo only).
