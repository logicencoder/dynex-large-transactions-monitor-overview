# Dynex Large Transactions — monitor service

Private Python worker for **large Dynex (DNX) on-chain transfers**. Polls a local Dynex daemon over JSON-RPC, filters transfers above a configurable threshold, stores history in SQLite, and POSTs batches to the WordPress plugin — so the public page never talks to the chain node.

**Operator service:** `dnx_large_txs.py` (multi-site edition, production on operator infrastructure)

**Public product (portfolio):** [dynex-large-transactions-plugin-overview](https://github.com/logicencoder/dynex-large-transactions-plugin-overview) — [logicencoder.com/dynex-large-transactions-monitor/](https://logicencoder.com/dynex-large-transactions-monitor/)

The plugin renders stats, searchable cards, pagination, and auto-refresh via **`[dynex_transactions]`**. This monitor is the chain indexer and ingest pump.

## Chain polling

Connects to local Dynex JSON-RPC (`getinfo`, `json_rpc` block fetches). Default large-tx threshold **9998 DNX** (overridable via `-t` or `dynex_config.ini` — production uses **4998 DNX**).

For each new block:

- Parses transaction outputs and coin transfers
- Keeps rows where amount exceeds threshold
- Deduplicates by `tx_hash` before push

Optional **lookback** mode rescans a block range (`--lookback`, `--start-block`) with parallel chunk processing (`--cores`, `--chunk-size`).

## Persistence

SQLite **`dynex_transactions.db`**:

| Table | Role |
|-------|------|
| `large_transactions` | Qualifying txs (timestamp, block, hash, from/to wallets, amount) |
| `wordpress_send_log` | Per-site push audit (success, response, block, tx count) |

Separate rotating logs under `logs/`, `large_tx_logs/`, and `address_logs/`.

## WordPress integration

Multi-site support — each target is `(url, api_key, site_name)` from `dynex_config.ini` or CLI (`-w`, `-k`, `-n`).

**`POST /wp-json/dynex/v1/transactions`**

- Header: `X-API-Key` (must match plugin option `dynex_api_key`)
- Body: `{ "transactions": [ { timestamp, block_number, tx_hash, from_wallet, to_wallet, amount } ] }`
- Expects `{ status, received, inserted }` — duplicates skipped via `INSERT IGNORE` on the plugin side

`auto_send = True` in config pushes each new large tx immediately after SQLite insert.

## CLI highlights

| Flag | Purpose |
|------|---------|
| `-t` / `--threshold` | DNX minimum for “large” classification |
| `--auto-send` | Push new txs to all configured WordPress sites |
| `--auto-run` | Start monitoring without interactive menu |
| `--send-initial` | Backfill recent SQLite rows to WordPress |
| `--lookback N` | Rescan last N blocks |
| `--track-address` / `--view-address` | Operator address tooling |
| `--test-api` | Verify Dynex RPC connectivity |

Interactive menu supports WordPress site configuration and connection tests without editing INI by hand.

## Operator extras

Beyond the public feed, the worker includes address directory collection, balance refresh, CSV export, mining-reward filtering, and log import — operator tooling that does not surface on the WordPress shortcode.

Private WordPress side: [dynex-large-transactions-plugin](https://github.com/logicencoder/dynex-large-transactions-plugin).

See [REPOS.md](REPOS.md).

---

**Made by [Logic Encoder](https://logicencoder.com)** · [GitHub](https://github.com/logicencoder) · [Contact](https://logicencoder.com/contact/)
