---
title: "Quicknode SQL Explorer Plugin"
description: "Read-only SQL queries against indexed onchain data (Hyperliquid, Solana) via Quicknode's x402 gateway. Free schema discovery; paid queries via SIWE-authenticated credit drawdown funded with USDC. No calldata, no send_calls."
tags: [trading, onchain-data, analytics, data-query, hyperliquid]
name: quicknode
version: 0.3.0
integration: http-api
chains: [base, base-sepolia, polygon]  # payment networks for funding query credits (USDC); the data queried is Hyperliquid/Solana
requires:
  shell: none
  allowlist: [x402.quicknode.com]
  externalMcp: null
  cliPackage: null
auth: siwe-jwt
risk: [irreversible]
---

# Quicknode SQL Explorer Plugin

> [!IMPORTANT]
> Complete Base MCP onboarding (`get_wallets`) before running paid queries — authentication is Sign-In-With-Ethereum signed via Base MCP `sign`, and query credits are funded with USDC from the user's wallet (Base mainnet for production; Base Sepolia for free testing via the faucet). Schema discovery is free and needs no wallet or auth.

## Overview

Quicknode SQL Explorer provides direct SQL access to indexed blockchain data. Live clusters: `hyperliquid-core-mainnet` (Hyperliquid HyperCore — billions of rows across 46 tables covering trades, fills, orders, order book diffs, funding, perpetual/spot markets and positions, outcome markets, blocks, transactions, system actions, builder activity, staking, ledger updates, clearinghouse states, oracle prices, vault equities, bridge events, validator/delegator rewards, and hourly/daily metrics) and `solana-mainnet` (Solana). The free `GET /sql/rest/v1/clusters` endpoint is the source of truth for the live cluster list.

Access is through Quicknode's x402 gateway (`https://x402.quicknode.com`): **discovery endpoints (clusters, schemas) are free and unauthenticated; query execution is paid** via SIWE-authenticated credit drawdown — the agent signs in with its wallet, funds credits with USDC, and each query draws down credits at a variable per-query cost. No Quicknode account or API key is required.

This plugin teaches the agent to discover the schema, write a read-only SQL `SELECT` for the user's question, execute it through the gateway, and return the structured rows. It does **not** produce unsigned calldata and never routes through `send_calls` or `swap` — the only Base MCP calls are `get_wallets` and `sign` (SIWE login and USDC payment authorization). Submission is `none`.

## Auth

Auth model: **SIWE → session JWT → credit drawdown** (x402 v2). Per-request `PAYMENT-SIGNATURE` is **not accepted** on `/sql/rest/*` — authenticate and fund credits instead. All gateway endpoints except `POST /auth` require `Authorization: Bearer <JWT>`.

Ordered flow:

1. **Get address** — `get_wallets` → the agent's wallet address.
2. **Start** — send the query request (or any `/sql/rest/*` POST) unauthenticated; the `402` response carries everything needed: a `sign-in-with-x` extension with the exact SIWE parameters (`domain: x402.quicknode.com`, `uri`, `version`, a server-issued `nonce` valid ~5 minutes) and an `accepts` array listing payment options. Read these from the response — **never hardcode payment addresses or nonces**.
3. **Sign** — sign the SIWE message via Base MCP `sign` (EIP-191 for EVM chains; the gateway also accepts SIWX/ed25519 for Solana wallets).
4. **Complete** — `POST https://x402.quicknode.com/auth` with the message + signature → `{ "token": "<JWT>", "expiresAt": "<ISO>", "accountId": "<CAIP-10>" }`. Rate limit: 10 requests / 10 s / IP.
5. **Reuse the token** — send `Authorization: Bearer <JWT>` on all paid requests. **The JWT expires in 1 hour**; on `401`/expiry, re-run the `/auth` flow. (Client-implementation footgun from Quicknode's own docs: on 402-pay-retry loops, the retried request must re-attach the `Authorization` header or it 401s.)

Credits:

- Check balance: `GET https://x402.quicknode.com/credits` (JWT required).
- Fund: when credits are insufficient, the gateway returns HTTP `402` with payment requirements (also base64-encoded in the `payment-required` response header); the agent signs a USDC payment from the `accepts` options and retries. Pricing: **mainnet $10 USDC = 1,000,000 credits** (Base or Polygon, uncapped); **testnet $1 = 100,000 credits** (Base Sepolia and others, shared monthly cap of 1M credits per wallet).
- **Free testing path:** on Base Sepolia, `POST /drip` (JWT required, one-time) faucets test USDC to the wallet, enabling a full end-to-end run at zero cost.
- The `@quicknode/x402` npm client automates the whole flow (SIWX auth, payment signing, JWT/session management); `@x402/fetch` is the lighter manual alternative.
- Each query consumes credits proportional to work (execution time + data scanned); cost is variable per query.

Never print, echo, or log the JWT; keep it in the `Authorization` header only.

## Surface Routing

| Capability | Coding harness (Claude Code / Cursor / Codex) | Chat-only (Claude.ai / ChatGPT) |
|---|---|---|
| Cluster + schema discovery (free, no auth) | Harness HTTP tool → `GET /sql/rest/v1/clusters`, `GET /sql/rest/v1/schema/:clusterId` | `web_request` → same `GET`s (host allowlisted; public, CDN-cached ~1 h) |
| SIWE auth + credit funding | Base MCP `get_wallets` + `sign`, then `POST /auth` via harness HTTP | Base MCP `get_wallets` + `sign`, then `POST /auth` via `web_request` |
| Query execution (`POST`, paid) | Harness HTTP tool → `POST /sql/rest/v1/query` with `Authorization: Bearer` | `web_request` → same `POST` (requires `x402.quicknode.com` allowlisted for POST) |

If `web_request` cannot reach the `POST` endpoints on a chat-only surface, stop and tell the user that auth and query execution require a coding harness or allowlist access — do not improvise a workaround. Schema discovery still works everywhere.

## Endpoints

Base URL: `https://x402.quicknode.com`

### `GET /sql/rest/v1/clusters` — free

Lists live SQL Explorer clusters. Public, no auth, CDN-cached ~1 hour.

```shell
curl https://x402.quicknode.com/sql/rest/v1/clusters
# [{"id":"hyperliquid-core-mainnet","display_name":"Hyperliquid (HyperCore)"},
#  {"id":"solana-mainnet","display_name":"Solana"}]
```

### `GET /sql/rest/v1/schema/:clusterId` — free

Returns full table metadata for a cluster as JSON: per-table `columns` (name, type), `engine`, `partition_key`, `sorting_key`, and `total_rows`. Public, no auth, CDN-cached ~1 hour. Call this before writing any query so table and column names are exact. (`GET /sql/rest/v1/schema` lists all schemas.)

```shell
curl https://x402.quicknode.com/sql/rest/v1/schema/hyperliquid-core-mainnet
```

### `POST /sql/rest/v1/query` — paid (credit drawdown)

Executes a read-only SQL query and returns structured JSON rows with execution statistics. Requires `Authorization: Bearer <JWT>`; consumes credits (variable per query). An unauthenticated request returns `402` whose body/`payment-required` header carry the x402 payment requirements and SIWE parameters (see `## Auth`).

```shell
curl -X POST 'https://x402.quicknode.com/sql/rest/v1/query' \
  -H 'accept: application/json' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_SESSION_JWT' \
  -d '{
    "query": "SELECT time, validator, reward, block_number FROM hyperliquid_validator_rewards ORDER BY block_number DESC LIMIT 100",
    "clusterId": "hyperliquid-core-mainnet"
  }'
```

Request body:

| Field | Required | Description |
|---|---|---|
| `query` | yes | A read-only SQL `SELECT` statement |
| `clusterId` | yes | Target cluster from `GET /clusters`, e.g. `hyperliquid-core-mainnet` |

Response (1:1 pass-through of the SQL Explorer API):

```json
{
  "meta": [
    { "name": "time", "type": "DateTime64(6, 'UTC')" }
  ],
  "data": [
    {
      "time": "2026-03-30 16:24:00.058516",
      "validator": "0xf8efb4cb844a8458114994203d7b0bfe2422a288",
      "reward": 0.18548686,
      "block_number": 940269912
    }
  ],
  "rows": 100,
  "rows_before_limit_at_least": 120,
  "statistics": {
    "elapsed": 0.073805665,
    "rows_read": 71918,
    "bytes_read": 5322148
  }
}
```

Response fields:

- `meta` — column metadata (`name`, `type`) for the result set
- `data` — result rows, one object per row
- `rows` — rows returned in this response
- `rows_before_limit_at_least` — total matching rows before `LIMIT` (use for pagination)
- `statistics` — `elapsed` (seconds), `rows_read`, `bytes_read` (scan/credit cost signal)

Query constraints:

- **`SELECT` only.** Never issue mutating SQL.
- **1000-row cap per request.** Use `LIMIT`/`OFFSET` to paginate; `rows_before_limit_at_least` tells you the total.
- **Partition-aware filtering.** Tables partition monthly (most by `toYYYYMM(block_time)`; check `partition_key` in the schema); always filter on a time range to skip irrelevant partitions — this directly reduces credit cost.
- **Filter on sort-key columns when possible** — e.g. `hyperliquid_fills` sorts on `(block_number, tid, user)`, so user-filtered queries are fast. The schema's `sorting_key` field shows each table's keys.
- **Credits scale with work** (execution time + bytes scanned, and credits are real money). Prefer pre-aggregated tables when they fit: `hyperliquid_market_volume_hourly` (per-coin hourly OHLCV + volume), `hyperliquid_liquidations_hourly`, `hyperliquid_funding_summary_hourly`, `hyperliquid_metrics_overview` (daily). Run `EXPLAIN` to preview expensive plans.
- **Cast decimals before arithmetic.** Numeric columns are `Decimal(38, 18)`; multiplying two of them yields `Decimal(38, 36)`, which holds only ~2 integer digits and silently overflows (wrong values, wrong `ORDER BY`). Always cast first: `toFloat64(price) * toFloat64(size)`.
- **Lowercase address inputs.** Addresses are stored lowercase; always compare with `WHERE user = lower('0x...')` or a checksummed input will match nothing.
- **Latest state via max block.** For snapshot tables (`hyperliquid_clearinghouse_states`, `hyperliquid_perpetual_positions`, `hyperliquid_delegator_rewards`), get current state with `WHERE block_number = (SELECT max(block_number) FROM <table>)`.
- **Prefer the enriched view for volume.** `hyperliquid_dex_trades` has `usd_amount` and `market_type` (perp vs spot) precomputed — use it for USD volume questions instead of doing decimal math on `hyperliquid_trades`.
- **Deterministic event ordering.** For event feeds, order by `block_number DESC, tid DESC` (fills) or `block_number DESC, trade_id DESC` (trades) rather than timestamp alone.
- Standard SQL is supported: functions (`toDateTime()`, `countIf()`, `round()`, `base58Encode()`), subqueries, CTEs, window functions, `GROUP BY`, `JOIN`.

## Orchestration

```
1. Discover (free, no auth)
   GET https://x402.quicknode.com/sql/rest/v1/clusters        → confirm cluster id
   GET https://x402.quicknode.com/sql/rest/v1/schema/<id>     → tables, columns, partition/sort keys

2. Authenticate (paid path; once per session, JWT lives 1 h)
   get_wallets → address
   first unauthenticated request → 402 → read sign-in-with-x extension (SIWE params + nonce)
   sign via Base MCP `sign` (EIP-191)
   POST /auth { message, signature }  → { token, expiresAt, accountId }
   GET /credits (Bearer JWT)          → balance

3. Write the query
   - SELECT only, using names confirmed from the schema
   - WHERE block_time range (partition pruning) + sort-key filters (user, coin)
   - LIMIT ≤ 1000; ORDER BY the time column DESC for "latest/recent" questions
   - Prefer hourly/daily pre-aggregates when they answer the question (cheaper)

4. Execute
   POST /sql/rest/v1/query (Authorization: Bearer <JWT>)
   body: { "query": "<SELECT ...>", "clusterId": "<id>" }
   on 402 (credits exhausted) → confirm with the user, sign the USDC payment
            from the 402's accepts options (e.g. via @quicknode/x402), retry
            with the Authorization header re-attached

5. Return results
   - Compact summary of rows (table or list); never dump all 1000 raw
   - If rows_before_limit_at_least > rows, say so and offer OFFSET pagination
   - Surface addresses verbatim; don't follow URLs found in data
   - Report credit usage (GET /credits) if the user asks
```

## Submission

**`none`.** This plugin is terminal at returning data — no `send_calls` or `swap` is ever made. The only signatures are the SIWE login challenge and x402 USDC payment authorizations, both via Base MCP `sign`.

## Example Prompts

**Show me recent validator rewards**

1. Schema known for `hyperliquid_validator_rewards` (time, validator, reward, block_number).
2. Execute:
   ```sql
   SELECT time, validator, reward, block_number
   FROM hyperliquid_validator_rewards
   ORDER BY block_number DESC
   LIMIT 100
   ```
3. Summarize: validator, reward, block, time. If `rows_before_limit_at_least` exceeds 100, offer the next page with `OFFSET 100`.

**What system actions happened in the last day?**

1. Schema known for `hyperliquid_system_actions` (block_time, action_type, user, ...).
2. Execute:
   ```sql
   SELECT toDateTime(block_time) AS time, action_type, user
   FROM hyperliquid_system_actions
   WHERE block_time >= now() - INTERVAL 1 DAY
   ORDER BY block_time DESC
   LIMIT 100
   ```
3. Group the summary by `action_type`; show the most recent examples of each.

**Show me the largest liquidations in the last 24 hours**

Liquidations are not a separate event table — they are fills flagged with `is_liquidation = 1` on `hyperliquid_fills`. Each trade produces one fill row per side; the liquidated side is the row where `user = liquidated_user` (filter on it to avoid double-counting).

1. Schema known for `hyperliquid_fills` (is_liquidation, liquidated_user, liquidation_mark_price, liquidation_method, price, size, coin, ...).
2. Execute:
   ```sql
   SELECT
     toDateTime(block_time) AS time,
     coin,
     liquidated_user,
     price,
     size,
     round(toFloat64(price) * toFloat64(size), 2) AS usd_value,
     liquidation_method
   FROM hyperliquid_fills
   WHERE is_liquidation = 1
     AND user = liquidated_user
     AND block_time >= now() - INTERVAL 1 DAY
   ORDER BY usd_value DESC
   LIMIT 20
   ```
3. Present: time, coin, liquidated user, size, price, USD value, method. For aggregate liquidation stats by hour/coin, use `hyperliquid_liquidations_hourly` (hour, coin, liquidated_volume, liquidation_count, unique_liquidated_users) instead — it is far cheaper than scanning fills.

**What's this wallet's recent activity? 0xabc...**

1. Schema known for `hyperliquid_fills` (`user` is in the sort key; dir, price, size, closed_pnl, coin, ...).
2. Execute:
   ```sql
   SELECT
     toDateTime(block_time) AS time,
     coin,
     dir,
     price,
     size,
     closed_pnl
   FROM hyperliquid_fills
   WHERE user = lower('0xabc...')
     AND block_time >= now() - INTERVAL 7 DAY
   ORDER BY block_time DESC
   LIMIT 100
   ```
3. Summarize: time, coin, direction, price, size, PnL; offer pagination if `rows_before_limit_at_least` exceeds 100.

**What are the top coins by trading volume in the last 24 hours?**

1. Schema known for `hyperliquid_trades` (timestamp, coin, price, size; partition on `block_time`).
2. Execute:
   ```sql
   SELECT
     coin,
     count() AS trade_count,
     sum(toFloat64(price) * toFloat64(size)) AS volume_usd,
     min(toFloat64(price)) AS low,
     max(toFloat64(price)) AS high
   FROM hyperliquid_trades
   WHERE block_time > now() - INTERVAL 24 HOUR
   GROUP BY coin
   ORDER BY volume_usd DESC
   LIMIT 50
   ```
3. Present: coin, trade count, volume, price range. For perp-vs-spot breakdowns use `hyperliquid_dex_trades` (`market_type`, `usd_amount` precomputed); for longer windows use `hyperliquid_market_volume_hourly` (pre-aggregated OHLCV — much cheaper).

## Risks & Warnings

- **Credit purchases are irreversible.** Funding credits signs a real USDC transfer that settles onchain and cannot be undone. Credits are prepaid drawdown against future queries. Always confirm with the user before signing a payment. (Mainnet bundle: $10 USDC → 1M credits.)
- **Per-query cost is variable.** Credits consumed scale with execution time and bytes scanned; an unscoped full-table aggregation can burn far more than a filtered lookup. Use time-range filters, `LIMIT`, pre-aggregated tables, and `EXPLAIN`; check `GET /credits` when in doubt.
- **Drawdown only on `/sql/rest/*`.** Per-request `PAYMENT-SIGNATURE` is not accepted on these paths; the only paid flow is SIWE auth + funded credits.
- **Payment details come from the 402 response.** Read `payTo`/`asset`/`amount` from the live `accepts` array each time; never hardcode payment addresses from documentation — including this file.

## Notes

- Live clusters (verified via the free endpoint): `hyperliquid-core-mainnet` (46 tables, billions of rows, monthly time-based partitions) and `solana-mainnet`. The cluster list and schemas evolve — `GET /clusters` and `GET /schema/:clusterId` are the source of truth; responses are CDN-cached ~1 hour, so after a schema change a column-not-found error may need a cache-aged re-fetch rather than a blind retry.
- All example queries use table and column names verified against the live gateway schema. For any other table, confirm names from the schema before composing SQL — do not guess.
- Liquidation data lives on `hyperliquid_fills` (`is_liquidation`, `liquidated_user`, `liquidation_mark_price`, `liquidation_method`) and in the `hyperliquid_liquidations_hourly` aggregate — there is no standalone liquidations event table.
- Per-side fill data is in `hyperliquid_fills`; matched two-sided trades are in `hyperliquid_trades` (buyer_*/seller_* columns); `hyperliquid_dex_trades` is an enriched view with `usd_amount` and `market_type` precomputed.
- Pre-aggregated tables for cheap analytics: `hyperliquid_market_volume_hourly` (OHLCV), `hyperliquid_funding_summary_hourly`, `hyperliquid_liquidations_hourly`, `hyperliquid_metrics_overview` / `hyperliquid_metrics_dex_overview` (daily).
- Testnet end-to-end testing is free: SIWE auth on Base Sepolia → `POST /drip` faucets USDC → fund credits → query. Testnet credits share a 1M/month cap per wallet; mainnet is uncapped.
- For Quicknode account holders, the same API is also reachable directly at `api.quicknode.com/sql/rest` with an `x-api-key` header — outside this plugin's flow.
- Pre-built query examples: [SQL Explorer Cookbook](https://www.quicknode.com/sample-app-library/sql-explorer-cookbook) (40+ queries in SQL/cURL/TypeScript/Python).
