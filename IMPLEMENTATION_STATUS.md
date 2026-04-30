# Ignite DEX — Implementation Status

> **Generated:** 2026-04-30  
> **Sources:** Technical Documentation · Framework Guide · Overview & Tokenomics  
> **Overall Progress:** ~8% fully done · ~17% partial · ~75% not yet built (334 spec items total)

---

## Legend

| Symbol | Meaning |
|--------|---------|
| ✅ | Implemented & functional |
| ⚠️ | Partially implemented / stub / not wired up |
| ❌ | Not yet built |

---

## 0. Production Server Infrastructure

> From Framework Guide — these are the verified, assigned server roles.

| Server IP | Role | Status | Notes |
|-----------|------|--------|-------|
| `194.99.22.7` | PostgreSQL (primary state DB) | ❌ | DB schema exists locally; not deployed to this host |
| `89.38.131.38` | Matching Engine | ❌ | Engine runs in-process with API; not isolated to dedicated host |
| `72.61.158.30` | Risk / Liquidation Engine | ❌ | Risk logic lives in API service; not on dedicated host |
| `187.124.184.170` | Web + REST API + WebSocket | ❌ | No Dockerfile / deployment script targets this host |
| `95.169.204.96` | Redis + Message Bus (primary) | ❌ | Redis in Docker Compose; no app connection |
| `95.169.205.114` | Redis + Monitoring (secondary) | ❌ | Not provisioned |

**What needs to be built:**
- Dockerfiles for all services (`apps/backend`, `apps/web`, matching engine, risk service)
- Service isolation: matching engine and risk engine must run as independent processes
- Redis integration: publish/subscribe for event fan-out
- Deployment scripts targeting each server IP
- Reverse proxy / ingress config for `187.124.184.170`
- At least two independent RPC providers per chain with health-scored fallback

---

## 1. System Architecture

### 1.1 Core Layers

| Layer | Spec Requirement | Status | Notes |
|-------|-----------------|--------|-------|
| API Gateway | Auth, rate limiting, routing | ⚠️ | Fastify server exists (`apps/backend/src/server.ts`); signature check mocked (`signature.length > 10`) |
| Matching Engine | Off-chain, in-memory, deterministic, isolated process | ⚠️ | `packages/engine/src/index.ts` — class exists; runs in-process, not isolated; no fixed-point math |
| Risk Engine | Real-time margin + liquidation, independent process | ⚠️ | Formulas in `apps/backend/src/services/perps.service.ts`; not wired to live order flow; not isolated |
| Settlement Engine | Deterministic on-chain batch commits | ⚠️ | `IgniteSettlement.sol` exists; backend never calls `commitBatch()` |
| Indexer | Async data + analytics (Postgres + ClickHouse) | ❌ | ClickHouse in Docker Compose; no indexer service exists |
| AI Engine | Strategy execution (optional) | ❌ | Documented in `docs/ai/`; no code |
| Event Bus (NATS) | Event-driven backbone between all services | ❌ | NATS in Docker Compose; zero producers/consumers wired |
| Redis Cache | Hot reads, session state, pub/sub | ❌ | Redis in Docker Compose; app never connects |

### 1.2 Deterministic Design Requirements

| Requirement | Status | Notes |
|-------------|--------|-------|
| Fixed-point math (no floats) | ❌ | Engine uses JavaScript `number` — floating point |
| Strict timestamp ordering | ⚠️ | Timestamps stored; ordering not enforced in queue |
| Single-threaded matching per market | ⚠️ | Node.js is single-threaded but no per-market isolation |
| Ordered event queue | ❌ | No event log implemented |
| State replayability | ❌ | No snapshot + replay mechanism |
| Fraud proofs (future) | ❌ | Planned |
| zk-verification (future) | ❌ | Planned |

### 1.3 Core Event Types (required on NATS bus)

| Event | Status |
|-------|--------|
| `OrderPlaced` | ❌ |
| `OrderCancelled` | ❌ |
| `TradeExecuted` | ❌ |
| `MarginUpdated` | ❌ |
| `LiquidationTriggered` | ❌ |
| `FundingUpdated` | ❌ |

---

## 2. Matching Engine

### 2.1 Order Book

| Feature | Status | Notes |
|---------|--------|-------|
| Bid/ask price levels (sorted) | ✅ | `packages/engine/src/index.ts` |
| FIFO within price level | ✅ | Insertion order |
| In-memory state | ✅ | `Map<string, PriceLevel>` per market |
| Persistence / replication | ❌ | Lost on restart |

### 2.2 Order Types

| Order Type | Status | Notes |
|------------|--------|-------|
| Limit | ✅ | Inserted to book if unmatched |
| Market | ✅ | Consumes liquidity immediately |
| IOC (Immediate-Or-Cancel) | ⚠️ | Type field exists; cancel-remainder not enforced |
| FOK (Fill-Or-Kill) | ⚠️ | Type field exists; full-fill-or-revert not enforced |
| Stop-Market | ❌ | Not implemented |
| Stop-Limit | ❌ | Not implemented |
| Post-Only | ❌ | Not implemented |
| Reduce-Only | ❌ | Not implemented |
| Take-Profit | ❌ | Not implemented |

### 2.3 Time-In-Force

| TIF | Status |
|-----|--------|
| GTC (Good-Till-Cancel) | ⚠️ | Default behavior but not explicitly labeled |
| IOC | ⚠️ | Partially |
| FOK | ⚠️ | Partially |
| ALO (Post-Only) | ❌ | Not implemented |

### 2.4 Order Lifecycle States

| State | Status | Notes |
|-------|--------|-------|
| Received | ⚠️ | Implicitly on API entry |
| Accepted | ⚠️ | No explicit state machine |
| Resting | ⚠️ | In-memory only |
| Partially Filled | ⚠️ | Tracked in engine; not persisted |
| Filled | ⚠️ | In engine; not persisted |
| Cancelled | ⚠️ | Route exists; not persisted |
| Rejected | ⚠️ | Returned as error; not stored |
| Expired | ❌ | No TTL / expiry logic |

### 2.5 Market Metadata (per market spec)

| Field | Status | Notes |
|-------|--------|-------|
| Symbol / canonical ID | ✅ | Seeded in DB |
| Quote / collateral asset | ✅ | In schema |
| Tick size | ❌ | Not defined per market |
| Lot size | ❌ | Not defined per market |
| Max leverage | ⚠️ | Hardcoded in engine, not per-market |
| IMR / MMR per market | ❌ | Flat rates only |
| Funding cadence per market | ❌ | No funding logic |
| Oracle source per market | ❌ | Single global price feed |
| Position limits per market | ❌ | Not implemented |

### 2.6 Matching Algorithm

| Feature | Status | Notes |
|---------|--------|-------|
| Price-time priority | ✅ | Best price, FIFO |
| Partial fills | ✅ | `Math.min(order.size, resting.remaining)` |
| Trade event generation | ⚠️ | `executeTrade()` called; not persisted or broadcast |
| Remove empty levels | ✅ | Implemented |
| Post-match book insert | ✅ | Remaining limit rests in book |

### 2.7 Engine Replication

| Feature | Status |
|---------|--------|
| Primary / follower model | ❌ |
| Event log | ❌ |
| Replay from log | ❌ |
| Snapshot model | ❌ |

---

## 3. Risk Engine

### 3.1 Core Calculations

| Feature | Status | Notes |
|---------|--------|-------|
| Equity = Wallet Balance + Unrealized PnL | ⚠️ | Formula present; not live per trade |
| Unrealized PnL (long + short formula) | ⚠️ | In `perps.service.ts`; fed hardcoded mark prices |
| Notional = |Size × Mark Price| | ⚠️ | Formula present |
| Margin ratio = Equity / Notional | ⚠️ | Formula exists; not enforced in order flow |
| Initial margin check | ⚠️ | At placement only; not live |
| Maintenance margin check | ⚠️ | Logic present; not continuously evaluated |
| Zero tolerance for negative equity | ❌ | Not enforced |
| Synchronous with matching engine | ❌ | Async only |
| Risk state streamed to clients | ❌ | Not implemented |

### 3.2 Dynamic Risk Tiers

| Feature | Status | Notes |
|---------|--------|-------|
| Tier table (notional caps, IM/MM rates) | ❌ | Flat rates only; example tiers: Tier1 <$100k IM1%/MM0.5%, Tier2 <$1M IM2%/MM1%, Tier3 >$1M IM5%/MM2.5% |
| Tier lookup on order sizing | ❌ | Not implemented |

### 3.3 Margin Modes

| Feature | Status | Notes |
|---------|--------|-------|
| Isolated margin (per-position) | ⚠️ | Data model supports it; enforcement not wired |
| Cross margin (shared balances) | ❌ | Not implemented |
| Mode switching per position | ❌ | Not implemented |
| Up to 100x leverage | ⚠️ | Max configurable; not enforced per tier |

### 3.4 Warning Threshold

| State | Condition | Status |
|-------|-----------|--------|
| Safe | MR > IM | ❌ Not surfaced to user |
| Warning | MM < MR ≤ IM | ❌ Not surfaced to user |
| Liquidation trigger | MR ≤ MM | ⚠️ Logic exists; not continuous |

---

## 4. Liquidation System

| Feature | Status | Notes |
|---------|--------|-------|
| Trigger: MR ≤ MM | ⚠️ | `isLiquidatable()` exists; not called continuously |
| Partial liquidation algorithm | ⚠️ | `partialLiquidation()` stubbed in perps service |
| Re-check after partial liq | ❌ | Not implemented |
| Full liquidation + collateral seizure | ❌ | Not implemented |
| Liquidation pricing (mark ± penalty) | ❌ | Not implemented |
| Insurance fund deficit routing | ❌ | `IgniteInsuranceFund.sol` exists; not connected |
| Keeper / liquidation bot network | ❌ | Not implemented |
| Liquidator reward (% of position) | ❌ | Not implemented |
| Protocol fee on liquidation | ❌ | Not implemented |
| User-visible liquidation event log | ❌ | Not implemented |

---

## 5. Perpetuals Engine

| Feature | Status | Notes |
|---------|--------|-------|
| Mark price = Index + EMA(Basis) | ❌ | Static mock price |
| Index price (Chainlink / Pyth / TWAP) | ❌ | `price-feed.service.ts` uses FreeCryptoAPI/CoinGecko — not production oracle |
| Open interest tracking | ❌ | Not tracked |
| Position tracking (entry price, size, margin) | ⚠️ | Prisma schema + contract exist; not written from engine fills |
| Funding rate calculation | ❌ | Formula: `(Mark - Index) / Index`; not implemented |
| Full formula: `Premium Index + Clamp(IR - Premium)` | ❌ | Not implemented |
| Funding payment flow (longs ↔ shorts) | ❌ | Not implemented |
| Periodic funding settlement (hourly) | ❌ | No scheduler/cron |
| Non-expiring contract model | ✅ | Correct product type defined |
| `POST /margin/update` endpoint | ❌ | Not implemented (required by spec) |

---

## 6. Auto-Deleveraging (ADL)

| Feature | Status | Notes |
|---------|--------|-------|
| ADL queue (score = PnL% × leverage) | ❌ | Not implemented |
| Ranking and sorting of top traders | ❌ | Not implemented |
| ADL execution (reduce opposing positions, no orderbook) | ❌ | Not implemented |
| ADL triggered on insurance depletion | ❌ | Not implemented |
| ADL indicator shown in UI | ❌ | Not shown |
| Queue position visible to user | ❌ | Not shown |

---

## 7. Insurance Fund

| Feature | Status | Notes |
|---------|--------|-------|
| `IgniteInsuranceFund.sol` contract | ✅ | Spec deployed; manages balance |
| Funded from liquidation penalties | ❌ | Flow not connected |
| Funded from trading fees (portion) | ❌ | FeeRouter exists; insurance split not wired |
| Protocol allocation | ❌ | Not wired |
| Bad debt absorption | ❌ | Not implemented |
| Depletion → ADL fallback | ❌ | Not implemented |
| Multi-asset insurance pools (future) | ❌ | Planned |
| On-chain transparency dashboard (future) | ❌ | Planned |

---

## 8. Settlement Engine

| Feature | Status | Notes |
|---------|--------|-------|
| State delta per trade (`StateDelta`) | ❌ | Not computed |
| Batch aggregation | ❌ | Not built |
| Merkle root computation | ❌ | Not built |
| `commitBatch(merkleRoot, batchId)` on-chain call | ❌ | Contract function exists; backend never calls it |
| Batch interval 1–5 seconds | ❌ | No scheduler |
| Confirmation + finalization flow | ❌ | Not built |
| State consistency reconciliation (Event Log → Replay → Compare → Correct) | ❌ | Not built |
| Conflict resolution (pause market → replay → restore) | ❌ | Not built |
| zk-proof per batch (future) | ❌ | Planned |
| Fraud-proof fallback (future) | ❌ | Planned |

---

## 9. State Consistency & Finality

| Feature | Status | Notes |
|---------|--------|-------|
| Strong consistency within engine (instant finality) | ⚠️ | In-memory consistent; not persisted |
| Eventual consistency across chains | ❌ | No cross-chain sync |
| Probabilistic settlement finality | ❌ | Batch not committed |
| Chain finality tracking (network dependent) | ❌ | No chain event listeners |
| Reconciliation process | ❌ | Not built |
| Conflict detection | ❌ | Not built |

---

## 10. Failure Handling & Recovery

| Feature | Status | Notes |
|---------|--------|-------|
| Engine recovery: Restart → Load Snapshot → Replay → Resume | ❌ | No snapshot; restart loses all state |
| Snapshot model (`{ state, timestamp }`) | ❌ | Not implemented |
| Oracle failure: switch to fallback | ❌ | Single oracle, no fallback |
| Oracle failure: use TWAP | ❌ | No TWAP logic |
| Oracle failure: trigger circuit breaker | ❌ | Not wired |
| Chain outage: pause settlement | ❌ | Not implemented |
| Chain outage: continue off-chain matching | ❌ | Not implemented |
| Chain outage: resume on recovery | ❌ | Not implemented |
| Circuit breakers (extreme volatility / oracle deviation) | ❌ | Not implemented |
| Emergency mode: trading paused, withdrawals enabled | ⚠️ | `pause()` in contract; no backend hook |
| Governance intervention in emergency | ❌ | Not implemented |

---

## 11. Vault Architecture

| Feature | Status | Notes |
|---------|--------|-------|
| `IgniteVault.sol` — deposit/withdraw | ✅ | ~90% complete |
| `lockMargin()` / `unlockMargin()` | ✅ | `onlyEngine` modifier |
| Spot Vault (token balances) | ⚠️ | Contract supports; type separation not enforced |
| Perps Vault (margin collateral) | ⚠️ | Contract supports; not separated |
| Cross Vault (shared margin) | ❌ | Not implemented |
| Deposit flow: User → Contract → Vault → Indexer → Balance | ⚠️ | Contract works; Indexer missing; backend not listening to events |
| Withdrawal risk check | ⚠️ | Contract enforces; backend pre-check not wired |
| Isolation guarantee (engine can't access funds directly) | ✅ | Enforced via `onlyEngine` |
| Backend listens to on-chain Vault events | ❌ | No event watchers |

---

## 12. Multi-Chain Settlement & Cross-Chain Margin

| Feature | Status | Notes |
|---------|--------|-------|
| Chain config (Litho/Base/Arbitrum/BNB) | ✅ | `packages/config/src/chains.ts` |
| Lithosphere Chain ID 700777, RPC `https://rpc.litho.ai` | ✅ | Config present |
| Cross-chain unified equity calculation | ❌ | Not implemented |
| Total equity = sum(all chain balances) + PnL | ❌ | Not implemented |
| Available margin = equity − used margin | ❌ | Per-chain only |
| State mirroring via relayers | ❌ | Not built |
| Use Base collateral for Arbitrum trade | ❌ | Not implemented |
| Cross-chain state computed off-chain | ❌ | Not implemented |

---

## 13. Bridge & Messaging Layer

| Feature | Status | Notes |
|---------|--------|-------|
| `BridgeMintBurn.sol` | ❌ | Not built |
| Mint/burn canonical bridge model | ❌ | Not built |
| Lock → Message → Relay → Mint flow | ❌ | Not built |
| Permissionless relayer network | ❌ | Not built |
| Relayer incentives (fee competition) | ❌ | Not built |
| Replay protection (`processed[hash] = true`) | ❌ | Not built |
| Trusted relayer + validation (current) | ❌ | Not built |
| zk-proof light client validation (future) | ❌ | Planned |

---

## 14. IGNT Token & Tokenomics

### 14.1 Token Contracts

| Item | Status | Notes |
|------|--------|-------|
| `IgniteToken.sol` (ERC-20) | ✅ | Deployed spec, bridge minting roles |
| LEP100 IGNT on Lithosphere | ❌ | Not deployed |
| ERC-20 IGNT on BNB Chain | ❌ | Not deployed (canonical address missing) |
| ERC-20 IGNT on Arbitrum | ❌ | Not deployed |
| ERC-20 IGNT on Base | ❌ | Not deployed |
| IGNT addresses in `packages/config/src/tokens.ts` | ❌ | Placeholder `0x...` only |

### 14.2 Fee Model & Distribution

| Feature | Status | Notes |
|---------|--------|-------|
| All fees payable in IGNT | ⚠️ | FeeRouter contract exists; not called from matching engine |
| Maker fee: 0% or rebate | ❌ | Not implemented |
| Taker fee: 0.01%–0.02% | ❌ | Not implemented |
| Fee split: Treasury 40% | ⚠️ | In `IgniteFeeRouter.sol`; never triggered |
| Fee split: Stakers 30% | ⚠️ | In contract; `Staking.sol` missing |
| Fee split: Liquidity Incentives 20% | ❌ | Not in current FeeRouter split (gap vs spec) |
| Fee split: Burn 10% | ⚠️ | In contract; never triggered |
| IGNT discount tiers for holders | ❌ | Not implemented |
| Deflationary burn mechanism | ⚠️ | `burnAddress` in FeeRouter; never called |
| Fee collection on each trade | ❌ | Not wired from matching engine |
| Multi-chain FeeRouter per chain | ❌ | Single contract; no per-chain routing |

### 14.3 Staking

| Feature | Status | Notes |
|---------|--------|-------|
| `Staking.sol` contract | ❌ | Not built |
| Stake IGNT → earn yield | ❌ | Not built |
| Stake → fee reduction | ❌ | Not built |
| Stake → governance weight | ❌ | Not built |
| Reward distribution (`rewardPerToken`) | ❌ | Not built |
| Staking UI (`/earn` page) | ⚠️ | UI renders; no contract calls |

### 14.4 Tokenomics Parameters

| Parameter | Spec Value | Status |
|-----------|-----------|--------|
| Total supply | 1,000,000,000 IGNT (fixed) | ❌ Not enforced on-chain |
| Ecosystem & Incentives | 35% | ❌ Allocation not configured |
| Liquidity & Market Making | 20% | ❌ Allocation not configured |
| Team & Contributors | 15% (4yr cliff 1yr) | ❌ Vesting not deployed |
| Treasury | 10% (controlled release) | ❌ Not configured |
| Strategic / Investors | 10% (2–3yr) | ❌ Not configured |
| Airdrop & Community | 10% | ❌ Not configured |
| Emissions (volume/liquidity/activity based) | Dynamic | ❌ Not implemented |

### 14.5 Governance

| Feature | Status | Notes |
|---------|--------|-------|
| `Governance.sol` (propose/vote/execute) | ❌ | Not built |
| Voting power = staked IGNT | ❌ | Not built |
| Timelock (24–72h) | ❌ | Not built |
| Multisig execution | ❌ | Not built |
| Governance scope: fees, listings, upgrades, risk params | ❌ | Not built |
| Proxy upgrade model (UUPS/Transparent) | ❌ | Contracts not using proxy pattern |

---

## 15. Smart Contracts

### 15.1 Deployed Contract Status

| Contract | Purpose | Status |
|----------|---------|--------|
| `IgniteToken.sol` | ERC-20 IGNT + bridge minting | ✅ ~100% |
| `IgniteVault.sol` | Collateral custody | ✅ ~90% |
| `IgniteSettlement.sol` | Batch PnL settlement | ⚠️ ~85% — never called by backend |
| `IgniteFeeRouter.sol` | Fee distribution | ✅ ~90% — never triggered |
| `IgniteInsuranceFund.sol` | Insurance reserve | ✅ ~90% — not wired to liquidations |
| `IgniteClearinghouse.sol` | Market config + trade execution | ⚠️ ~70% |
| `IgniteIncentives.sol` | Points + rewards | ⚠️ ~80% |
| `MockUSDC.sol` | Test token | ✅ 100% |

### 15.2 Missing Contracts

| Contract | Purpose | Status |
|----------|---------|--------|
| `Staking.sol` | IGNT staking, governance weight, fee discounts, rewards | ❌ |
| `BridgeMintBurn.sol` | Cross-chain mint/burn bridge | ❌ |
| `Governance.sol` | Proposal / vote / execute + timelock | ❌ |

### 15.3 Contract Integration with Backend

| Integration | Status | Notes |
|-------------|--------|-------|
| Backend listens to Vault deposit/withdraw events | ❌ | No event listeners |
| Backend calls `commitBatch()` on settlement | ❌ | Never wired |
| Backend calls `lockMargin()` on order placement | ❌ | In-memory check only |
| Backend calls `distribute()` on FeeRouter | ❌ | Never called |
| TypeChain bindings generated | ✅ | `packages/contracts/typechain-types/` |
| Contract audit before mainnet | ❌ | Not done |

---

## 16. Backend API

### 16.1 REST Endpoints

| Endpoint | Spec Requirement | Status | Notes |
|----------|-----------------|--------|-------|
| `POST /orders` | Place order + EIP-712 sig | ⚠️ | Mock sig check |
| `DELETE /orders/:id` | Cancel order | ⚠️ | Not persisted to DB |
| `GET /positions` | User positions | ⚠️ | Hardcoded/mock calculations |
| `GET /orderbook/:pair` | L2 snapshot | ✅ | `engine.snapshot()` |
| `GET /funding-rate/:pair` | Funding rate | ❌ | Not calculated |
| `GET /markets` | Market list | ✅ | Seeded markets |
| `GET /health` | Health check | ✅ | |
| `POST /auth/challenge` | Generate nonce challenge | ⚠️ | Exists; real nonce not persisted |
| `POST /auth/verify` | Verify EIP-712 + issue JWT | ⚠️ | Exists; real recovery not implemented |
| `POST /margin/update` | Update margin on position | ❌ | Not implemented |

### 16.2 API Auth Methods (all three required)

| Auth Method | Status | Notes |
|-------------|--------|-------|
| Wallet signature (EIP-712) | ⚠️ | Mocked |
| Session keys (low-latency trading) | ❌ | Not implemented |
| API keys (bots + institutions) | ❌ | Not implemented |

### 16.3 Order Lifecycle (per spec §5)

| Step | Status | Notes |
|------|--------|-------|
| 1. EIP-712 signature verification | ❌ | Mocked |
| 2. Balance check (vs vault contract) | ⚠️ | In-memory only |
| 3. Margin availability check | ⚠️ | In-memory only |
| 4. Engine queue entry | ✅ | `engine.placeOrder()` |
| 5. Trade generated + fills to DB | ⚠️ | In-memory; not written |
| 6. Risk engine update (synchronous) | ⚠️ | Partially wired |
| 7. Settlement batch creation | ❌ | Not built |
| 8. On-chain commit | ❌ | Not built |
| 9. WebSocket broadcast of fill | ⚠️ | Route exists; fills not broadcast |
| 10. Fee collection in IGNT | ❌ | Not wired |

### 16.4 Database Persistence

| Entity | Schema | Persisted | Notes |
|--------|--------|-----------|-------|
| Account | ✅ | ⚠️ | Created on auth; balances not live |
| Market | ✅ | ✅ | Seeded |
| Order | ✅ | ❌ | In-memory engine only |
| Fill | ✅ | ❌ | Not written |
| Position | ✅ | ❌ | Not updated from fills |
| Balance | ✅ | ❌ | Not updated from trades |

---

## 17. WebSocket Layer

| Feature | Status | Notes |
|---------|--------|-------|
| WS route (`/ws`) | ✅ | `apps/backend/src/routes/ws.routes.ts` |
| Channel: `orderbook` | ⚠️ | Full snapshot only; no delta streaming |
| Channel: `trades` | ❌ | No trade broadcast on fill |
| Channel: `positions` | ❌ | Not implemented |
| Channel: `funding` | ❌ | Not implemented |
| Channel: `ticker` | ⚠️ | Price feed polled; partial |
| Private auth channel (`op: auth`) | ❌ | Not implemented |
| `subscribe` / `unsubscribe` ops | ⚠️ | Basic subscribe exists |
| Delta updates (not full snapshots) | ❌ | Always full snapshot |
| Heartbeat / reconnect logic | ❌ | Not implemented |
| Subscription limits | ❌ | No enforcement |

---

## 18. Frontend (Web App)

### 18.1 Pages

| Page | Status | Notes |
|------|--------|-------|
| Trade terminal (`/trade`) | ⚠️ | UI renders; zero API wiring |
| Portfolio (`/portfolio`) | ⚠️ | UI renders; hardcoded values |
| Earn / Staking (`/earn`) | ⚠️ | UI renders; no contract calls |
| Leaderboard (`/leaderboard`) | ⚠️ | UI renders; no data |
| Dashboard (`/`) | ⚠️ | UI renders; no live data |

### 18.2 Wallet Integration

| Feature | Status | Notes |
|---------|--------|-------|
| wagmi + RainbowKit installed | ✅ | `package.json` |
| Providers wired in `layout.tsx` | ❌ | No `WagmiProvider` / `RainbowKitProvider` |
| `WalletButton.tsx` functional | ❌ | Exists; not wired |
| WalletConnect v2 | ❌ | Not configured |
| Hardware wallet support (Ledger/Trezor) | ❌ | Not configured |
| Chain switching | ❌ | Not implemented |
| EIP-712 signing from UI | ❌ | Not implemented |

### 18.3 Trading Components

| Component | Status | Notes |
|-----------|--------|-------|
| `OrderForm.tsx` — inputs render | ⚠️ | No submit handler / API call |
| `OrderBook.tsx` | ⚠️ | Renders; no live WS feed |
| `PriceChart.tsx` (Recharts) | ⚠️ | Static/mock data |
| `Portfolio.tsx` | ⚠️ | Hardcoded PnL and positions |
| `MarketPicker.tsx` | ⚠️ | No market list from API |
| Real-time orderbook delta updates | ❌ | Not wired |
| Trade history stream | ❌ | Not implemented |
| TradingView charts | ❌ | Recharts used; TradingView not integrated |
| Depth chart | ❌ | Not implemented |
| Multi-panel layout | ⚠️ | Basic layout exists; not dynamic |
| Keyboard-driven trading shortcuts | ❌ | Not implemented |

### 18.4 User Workflows (per Framework Guide §10)

| Workflow | Status | Notes |
|----------|--------|-------|
| Account onboarding (wallet connect + disclaimer) | ❌ | Not implemented |
| Deposit flow (choose network, asset, confirmations) | ❌ | Not implemented |
| Trading flow (select market, leverage, margin mode) | ⚠️ | UI only; no execution |
| Position management (modify leverage, TP/SL, add/reduce margin) | ❌ | Not implemented |
| Withdrawal flow (address validation, limits, queue) | ❌ | Not implemented |
| Referral / partner flow | ❌ | Not implemented |
| Status / incident banners (degraded mode) | ❌ | Not implemented |

---

## 19. Spot Trading

| Feature | Status | Notes |
|---------|--------|-------|
| Spot order placement | ❌ | Engine is perps-focused; no spot logic |
| Spot market data | ❌ | Not implemented |
| Permissionless token listings | ❌ | Not implemented |
| Listing requirement: min $20k liquidity | ❌ | No listing validation |
| Listing flow (deploy → pair → liquidity → submit → live) | ❌ | Not built |
| Auto-delisting (< threshold or 30 days inactive) | ❌ | Not built |

---

## 20. Liquidity Layer

| Feature | Status | Notes |
|---------|--------|-------|
| CLOB (Central Limit Order Book) | ✅ | Core matching engine |
| AMM fallback liquidity | ❌ | Not implemented |
| Cross-chain liquidity aggregation | ❌ | Not implemented |
| External routing (DEX aggregators) | ❌ | Not implemented |
| Market maker API (low-latency endpoints, broker IDs) | ❌ | No broker ID system |
| Market maker rebates | ❌ | Not implemented |
| Liquidity bootstrapping (bonding curves, launch pools) | ❌ | Not implemented |
| Incentivized LP rewards (paid in IGNT) | ❌ | Not implemented |

---

## 21. Referral & Incentives System

| Feature | Status | Notes |
|---------|--------|-------|
| Referral codes | ❌ | Not implemented |
| Referral earnings (% of referred user fees) | ❌ | Not implemented |
| Trading rewards (IGNT per volume) | ❌ | Not implemented |
| Liquidity mining rewards | ❌ | Not implemented |
| `IgniteIncentives.sol` — points tracking | ⚠️ | ~80% complete; not wired to trades |
| Dynamic emissions (volume/liquidity/activity) | ❌ | Not implemented |
| Referral UI / earnings dashboard | ❌ | Not implemented |

---

## 22. Binary Protocol (HFT Layer)

| Feature | Status | Notes |
|---------|--------|-------|
| Binary message format (34-byte header) | ❌ | JSON only |
| Opcodes: 0x01 place / 0x02 cancel / 0x03 modify | ❌ | Not built |
| Binary WebSocket transport | ❌ | Not built |
| Order modify endpoint | ❌ | No `PATCH /orders/:id` |

---

## 23. Order Signing & Anti-Replay

| Feature | Status | Notes |
|---------|--------|-------|
| EIP-712 typed data signing | ❌ | Mock check only |
| `keccak256` order hash | ❌ | Not computed |
| `recoverSigner` verification (`viem.recoverTypedDataAddress`) | ❌ | Not implemented |
| Per-user nonce (anti-replay) | ❌ | Not implemented |
| Idempotency for withdrawals / transfers | ❌ | Not implemented |

---

## 24. Rate Limiting & Throughput

| Feature | Status | Notes |
|---------|--------|-------|
| Request rate limiting middleware | ❌ | Fastify rate-limit not configured |
| Public tier: 50 req/sec | ❌ | Not implemented |
| Authenticated tier: 200 req/sec | ❌ | Not implemented |
| Institutional tier: custom | ❌ | Not implemented |
| Token bucket burst model | ❌ | Not implemented |
| WS subscription limits | ❌ | Not enforced |
| WebSocket backpressure | ❌ | Not implemented |
| Load balancing / geo-distributed gateways | ❌ | Not implemented |

---

## 25. Performance Targets

| Metric | Target | Current Status |
|--------|--------|----------------|
| Matching latency | < 1 ms | ⚠️ In-process Node.js; unmeasured |
| Order throughput | > 100k ops/sec | ❌ No benchmarks; no fixed-point math |
| Batch settlement interval | 1–5 seconds | ❌ No settlement |
| REST latency | < 50 ms | ⚠️ Unmeasured; likely acceptable for local |
| WebSocket | Real-time streaming | ⚠️ Partial |
| API throughput | 10k req/sec | ❌ No load testing |
| WS throughput | 1M msgs/sec | ❌ Not measured |

---

## 26. Security

### 26.1 Protocol Security

| Feature | Status | Notes |
|---------|--------|-------|
| MEV protection | ❌ | Not implemented |
| Circuit breakers | ❌ | Not implemented |
| Risk engine monitoring | ❌ | Not continuous |
| Emergency pause (contract) | ✅ | `pause()` onlyGuardian |
| Emergency pause (backend hook) | ❌ | Backend not wired to pause |

### 26.2 Smart Contract Security

| Feature | Status | Notes |
|---------|--------|-------|
| External audit | ❌ | Not done (mandatory before mainnet) |
| Multisig treasury | ❌ | Not configured |
| Timelock upgrades (24–72h) | ❌ | Not implemented |
| UUPS/Transparent proxy pattern | ❌ | Contracts not upgradeable |

### 26.3 User / API Security

| Feature | Status | Notes |
|---------|--------|-------|
| API key scopes | ❌ | Not implemented |
| IP allowlists | ❌ | Not implemented |
| Withdrawal approval / address screening | ❌ | Not implemented |
| Key rotation | ❌ | Not implemented |

### 26.4 AI Safety

| Feature | Status | Notes |
|---------|--------|-------|
| Agent permission scopes | ❌ | Not implemented |
| Trade limits for agents | ❌ | Not implemented |
| Kill-switch system | ❌ | Not implemented |
| Sandboxing / simulation mode | ❌ | Not implemented |

---

## 27. AI Trading Agents

| Feature | Status | Notes |
|---------|--------|-------|
| Agent runtime (execution sandbox) | ❌ | Not built |
| Agent data model (`id, owner, strategy, riskProfile, permissions`) | ❌ | Not built |
| Strategy execution engine (signal → decision → order) | ❌ | Not built |
| Signal types: momentum, OB imbalance, funding divergence, xchain arb, volatility | ❌ | Not built |
| Feedback loop (PnL evaluation → parameter adjustment) | ❌ | Not built |
| Agent coordination via message bus | ❌ | Not built |
| Agent types: market maker, arbitrageur, momentum, liquidation bot, portfolio mgr | ❌ | Not built |
| Risk-aware guardrails (max leverage, max drawdown, stop-loss) | ❌ | Not built |
| Kill switch (`halt(agent)` on drawdown breach) | ❌ | Not built |
| Capital pool allocation | ❌ | Not built |
| AI market making (dynamic spread = f(vol, inventory, risk)) | ❌ | Not built |
| Cross-chain arbitrage detection + execution | ❌ | Not built |
| On-chain agent identity + reputation (future) | ❌ | Planned |
| Verifiable strategies via zk-proof (future) | ❌ | Planned |

---

## 28. Future / Planned Products

| Product | Status | Notes |
|---------|--------|-------|
| Options (European, on-chain settlement) | ❌ | Planned — Phase 3 |
| Volatility primitives | ❌ | Planned |
| Ignite Launchpad (bonding curve + LBP) | ❌ | Planned |
| Alpha Terminal (smart wallet tracking, heatmaps, order flow analytics) | ❌ | Planned |
| AI Treasury (protocol fund allocation, yield optimization) | ❌ | Planned |
| Agent Economy (compete, collaborate, earn, reputation) | ❌ | Planned |
| DNNS identity layer integration (Lithosphere) | ❌ | Planned |
| Gas abstraction on Lithosphere | ❌ | Planned |
| Autonomous markets (self-balancing liquidity, AI-driven spreads) | ❌ | Planned |

---

## 29. Infrastructure & Operations

### 29.1 Deployment

| Component | Status | Notes |
|-----------|--------|-------|
| Docker Compose (local dev) | ✅ | PostgreSQL, Redis, NATS, ClickHouse, Prometheus, Grafana, Loki |
| `Dockerfile` for `apps/backend` | ❌ | Missing |
| `Dockerfile` for `apps/web` | ❌ | Missing |
| `Dockerfile` for matching engine | ❌ | Missing |
| `Dockerfile` for risk service | ❌ | Missing |
| Kubernetes manifests | ⚠️ | `infra/k8s/ignite-api-deployment.yaml` exists; incomplete |
| CI pipeline | ⚠️ | `ci.yml` runs build check; no tests, no deploy |
| Server-targeted deploy scripts (per Framework Guide IPs) | ❌ | Missing |

### 29.2 Observability

| Component | Status | Notes |
|-----------|--------|-------|
| Prometheus + Grafana | ⚠️ | Config files exist; app exports no metrics |
| Loki + Promtail log aggregation | ⚠️ | Config exists; app not emitting structured logs |
| Matching engine latency percentiles | ❌ | Not instrumented |
| Order reject rate | ❌ | Not tracked |
| WebSocket backlog metrics | ❌ | Not tracked |
| Risk engine health lag | ❌ | Not tracked |
| Oracle freshness check | ❌ | Not tracked |
| Chain RPC success % | ❌ | Not tracked |
| Public status page | ❌ | Not built |

### 29.3 SRE / Operations

| Requirement | Status | Notes |
|-------------|--------|-------|
| WAL archiving + PITR for PostgreSQL | ❌ | Not configured |
| Streaming replicas | ❌ | Not configured |
| At least 2 independent RPC providers per chain | ❌ | Single RPC |
| RPC health-scored fallback routing | ❌ | Not implemented |
| Redis sentinel / cluster with memory alarms | ❌ | Single instance |
| Incident severity matrix | ❌ | Not documented |
| Runbooks (engine restart, Redis failover, withdrawal pause) | ❌ | Not written |
| Point-in-time restore testing | ❌ | Not done |
| RTO / RPO defined per service | ❌ | Not defined |

---

## 30. Lithosphere (LITHO) Integration

| Feature | Status | Notes |
|---------|--------|-------|
| Chain config: ID 700777, RPC `https://rpc.litho.ai`, Explorer `https://makalu.litho.ai` | ✅ | Config present |
| LEP100 IGNT contract deployment | ❌ | Not deployed |
| Dual execution: Cosmos + EVM | ❌ | EVM side only considered |
| LITHO nodes in RPC pool | ❌ | No LITHO RPC wired to backend |
| DNNS identity layer | ❌ | Not integrated |
| LEP100 bridge to ERC-20 | ❌ | Bridge not built |

---

## 31. Mobile & Desktop Apps

| App | Status | Notes |
|-----|--------|-------|
| Mobile (Expo / React Native) | ❌ 5% | Shell only; zero screens, wallet connect, or API integration |
| Mobile: WalletConnect native | ❌ | Not implemented |
| Mobile: Push notifications | ❌ | Not implemented |
| Desktop (Tauri — macOS dmg + Windows msi) | ❌ 15% | Shell only; blocked on web app + code signing certs |
| Desktop: Auto-update channels | ❌ | Not configured |
| Desktop: Code signing (macOS notarization) | ❌ | Not configured |

---

## 32. SDK (`@ignite/dex-sdk`)

| Feature | Status | Notes |
|---------|--------|-------|
| `IgniteClient` class with chain config | ❌ | `packages/sdk/src/index.ts` is empty skeleton |
| `client.wallet.connect()` (WalletConnect v2) | ❌ | Not implemented |
| `client.trade.placeOrder()` | ❌ | Not implemented |
| `client.market.getAll()` | ❌ | Not implemented |
| Python SDK | ❌ | Not built |
| REST example docs | ❌ | Not written |
| OpenAPI spec | ❌ | Not generated |

---

## Summary: Prioritized Build Queue

### Priority 1 — Core Trading Loop (Blocks everything)

1. Wire `WagmiProvider` + `RainbowKitProvider` in `apps/web/app/layout.tsx`
2. Implement real EIP-712 `recoverTypedDataAddress` in auth service (viem)
3. Add per-user nonce storage and verification (anti-replay)
4. Persist orders, fills, positions, balances to Postgres after each match
5. Wire `OrderForm` submit handler → `POST /orders` API
6. Broadcast fill events over WebSocket on each match
7. Connect `OrderBook.tsx` to WS `orderbook` channel (delta updates)

### Priority 2 — Real Prices & Risk (Core safety)

8. Replace FreeCryptoAPI with Pyth / Chainlink oracle integration
9. Implement mark price = Index + EMA(Basis)
10. Run risk check (margin ratio) synchronously on every fill
11. Wire maintenance margin threshold → liquidation trigger
12. Implement partial liquidation market order execution
13. Wire liquidation penalties → `IgniteInsuranceFund.sol`
14. Implement dynamic risk tier table (3 tiers, notional-based)

### Priority 3 — Settlement & On-Chain Integrity

15. Compute `StateDelta` per fill
16. Batch deltas + compute Merkle root (1–5 second interval)
17. Backend calls `commitBatch()` on `IgniteSettlement.sol`
18. Add Vault event listeners (deposit/withdraw → update DB balances)
19. Backend calls `lockMargin()` on `IgniteVault.sol` on order placement

### Priority 4 — Perpetuals & Funding

20. Implement funding rate: `Premium Index + Clamp(IR - Premium)`
21. Hourly cron: apply funding payments (direct balance adjustments)
22. Track open interest per market
23. Implement `POST /margin/update` endpoint

### Priority 5 — IGNT & Fee System

24. Collect fees on every trade (send to FeeRouter)
25. Implement fee split: Treasury 40% / Stakers 30% / Liquidity 20% / Burn 10%
26. Build `Staking.sol` (stake, distribute rewards, governance weight)
27. Wire IGNT discount tiers to fee calculation
28. Deploy IGNT contracts on all 4 chains; populate `packages/config/src/tokens.ts`

### Priority 6 — Missing Contracts & Governance

29. Build `BridgeMintBurn.sol` with replay protection
30. Build `Governance.sol` (propose/vote/execute + timelock)
31. Add UUPS proxy pattern to core contracts for upgradeability

### Priority 7 — Infrastructure & Deployment

32. Write Dockerfiles for backend, web, engine, risk services
33. Write deployment scripts targeting Framework Guide server IPs
34. Isolate matching engine to `89.38.131.38`, risk engine to `72.61.158.30`
35. Wire Redis pub/sub for NATS event fan-out
36. Configure 2+ RPC providers per chain with fallback routing
37. Add Prometheus metrics to backend (order latency, reject rate, etc.)

### Priority 8 — Cross-Chain Margin & Bridge

38. Unified equity aggregation across all chains
39. Relayer network for cross-chain state mirroring
40. Use Base collateral for Arbitrum trade flow

### Priority 9 — Performance & HFT Protocol

41. Replace `number` with `bigint` fixed-point math in engine
42. Implement append-only event log for deterministic replay
43. Snapshot + replay mechanism (state recovery)
44. Rate limiting middleware (token bucket, per-tier)
45. Binary WebSocket protocol (HFT layer)
46. Session keys and API keys for bots / institutions

### Priority 10 — ADL

47. ADL queue (score = PnL% × leverage)
48. ADL execution on insurance depletion
49. ADL UI indicator + queue position display

### Priority 11 — AI Agents (Future)

50. Agent runtime with sandbox, permissions, risk profile
51. Strategy execution engine (signal → decision → order → feedback)
52. Agent coordination message bus
53. Capital pool allocation + dynamic rebalancing

### Priority 12 — Mobile & Desktop

54. Mobile app: all trading screens, wallet connect, API integration, push notifications
55. Desktop app: reuse web app in Tauri, code signing, auto-updates

---

## Quick Reference: Gap Count by System

| System | Spec Items | ✅ Done | ⚠️ Partial | ❌ Missing |
|--------|-----------|--------|-----------|-----------|
| Infrastructure / Servers | 10 | 0 | 0 | 10 |
| Matching Engine | 18 | 5 | 7 | 6 |
| Risk Engine | 12 | 0 | 7 | 5 |
| Liquidation | 10 | 0 | 2 | 8 |
| Perpetuals + Funding | 10 | 1 | 1 | 8 |
| ADL | 6 | 0 | 0 | 6 |
| Insurance Fund | 7 | 1 | 0 | 6 |
| Settlement Engine | 9 | 0 | 1 | 8 |
| State Consistency + Failure | 12 | 0 | 1 | 11 |
| Vault | 9 | 3 | 3 | 3 |
| Cross-Chain Margin | 8 | 1 | 0 | 7 |
| Bridge | 7 | 0 | 0 | 7 |
| IGNT Token + Tokenomics | 22 | 1 | 3 | 18 |
| Smart Contracts | 12 | 5 | 3 | 4 |
| Backend API | 18 | 3 | 9 | 6 |
| WebSocket | 11 | 1 | 4 | 6 |
| Frontend (Web) | 22 | 1 | 12 | 9 |
| User Workflows | 7 | 0 | 0 | 7 |
| Spot Trading | 6 | 0 | 0 | 6 |
| Liquidity Layer | 8 | 1 | 0 | 7 |
| Referral + Incentives | 7 | 0 | 1 | 6 |
| Binary Protocol | 4 | 0 | 0 | 4 |
| Order Signing + Auth | 7 | 0 | 0 | 7 |
| Rate Limiting | 8 | 0 | 0 | 8 |
| Security | 14 | 1 | 0 | 13 |
| AI Agents | 13 | 0 | 0 | 13 |
| Planned Products | 9 | 0 | 0 | 9 |
| Observability + Ops | 18 | 1 | 2 | 15 |
| Lithosphere Integration | 6 | 1 | 0 | 5 |
| Mobile + Desktop | 8 | 0 | 0 | 8 |
| SDK | 7 | 0 | 0 | 7 |
| **TOTAL** | **334** | **26 (8%)** | **56 (17%)** | **252 (75%)** |
