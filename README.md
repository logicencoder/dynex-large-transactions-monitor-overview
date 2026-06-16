# Dynex Large Transactions — monitor service

Private Python worker that watches a **local Dynex node**, classifies **large DNX transfers**, buffers them in SQLite, and **pushes batches to WordPress** so the public monitor page never opens an RPC connection.

Visitors use the WordPress shortcode on [logicencoder.com/dynex-large-transactions-monitor/](https://logicencoder.com/dynex-large-transactions-monitor/) — see [dynex-large-transactions-plugin-overview](https://github.com/logicencoder/dynex-large-transactions-plugin-overview) for the live UI, stats panels, and wp-admin screens. This service is the chain-side indexer and ingest pump behind that page.

## Tech stack

| Layer | Technologies |
|-------|--------------|
| Worker | Python 3, `requests`, `sqlite3`, `configparser`, `argparse`, `colorama` |
| Optional async | `aiohttp` when installed (address-range tooling) |
| Parallel scans | `multiprocessing` pools for lookback and address tracking |
| Chain access | Dynex JSON-RPC (`getinfo`, block-by-height fetches on local daemon port) |
| Persistence | SQLite local buffer — large transfers, push audit trail, address tooling |
| WordPress handoff | Authenticated REST batch push (`X-API-Key`) to the plugin ingest endpoint |
| Configuration | `dynex_config.ini` and/or CLI flags (`-w`, `-k`, `-n`, `-t`) |
| Operations | Rotating file logs (`logs/`, `large_tx_logs/`, `address_logs/`), USM-managed service on operator Linux host |
| Downstream | [dynex-large-transactions-plugin](https://github.com/logicencoder/dynex-large-transactions-plugin) — MySQL storage and `[dynex_transactions]` UI |

## Real-time block monitoring

The default production loop polls the node once per second, reads the latest block height from `getinfo`, and processes each new block exactly once.

For every transaction in the block, the worker parses outputs, converts raw amounts with a fixed **coin divisor** (1e9), and compares against **`LARGE_TX_THRESHOLD`**. Qualifying rows are written to the local SQLite buffer with hash-based deduplication so replays and duplicate blocks do not create double entries.

When **auto-send** is enabled and at least one WordPress target is configured, each batch of new large transfers from that block is POSTed immediately after SQLite insert. The plugin responds with how many rows were received versus inserted; duplicates are skipped server-side.

Production threshold on logicencoder.com is **4998 DNX** (`threshold` in `dynex_config.ini`). The script default when no config exists is **9998 DNX** — operators set the live value in INI or with `-t` / `--threshold`.

## WordPress multi-site push

**Multi-site edition** keeps a list of `(url, api_key, site_name)` tuples. Each ingest call sends the same transaction batch to every selected site in sequence and records success or failure per site in the **push audit log** (timestamp, site name, URL, HTTP outcome, transaction count, block number).

Configuration paths:

- **`dynex_config.ini`** — `[WordPress]` sections (repeatable) plus `[Settings]` for `auto_send` and `threshold`
- **CLI** — `-w` / `-k` / `-n` repeated for multiple targets; `--auto-send`, `--send-initial`, `--save-config`

The interactive menu (**option 4**) adds, lists, edits, or removes WordPress targets without hand-editing INI. **Option 6** sends a chosen slice of SQLite history (last 100, last 1000, all, or custom count) to one site or all configured sites — useful after plugin redeploy or API key rotation.

Ingest uses the plugin’s authenticated transactions endpoint; the API key must match the value in wp-admin **Dynex Transactions → Settings**. Details of the HTTP contract live in the private plugin `ARCHITECTURE.md`, not duplicated here.

## Historical lookback and backfill

**Lookback mode** (`--lookback`, menu option 8) rescans a window of past blocks from a starting height. Chunk size, sleep delay between chunks, and optional multiprocessing (`--cores`, `--chunk-size`, `--use-ram-cache`) tune how aggressively the worker walks history without overloading the node.

Derived timestamps during lookback approximate block age from a fixed seconds-per-block estimate so WordPress receives sensible `timestamp` fields even when replaying old chain data.

**`--send-initial`** pushes recent SQLite rows to WordPress on startup — handy when the public database was empty but the local buffer already held qualifying transfers.

## Interactive operator menu

When not started with `--auto-run` or `--no-menu`, a twenty-option terminal menu covers everything beyond the public feed:

| Area | Menu options | What operators do |
|------|----------------|-------------------|
| **Monitor** | 1 | Start the real-time block loop |
| **Inspect DB** | 2 | Print recent large transactions from SQLite |
| **Import** | 3 | Parse a legacy log file into the database |
| **WordPress** | 4–6, 5 toggle | Configure sites, flip auto-send, manual push |
| **Config** | 7 | Persist current threshold and WordPress list to INI |
| **History** | 8 | Interactive lookback wizard |
| **Address tools** | 9–11 | Track, view, or CSV-export a single wallet |
| **Directory** | 12–15 | Collect all chain addresses, stats, CSV export, balance refresh |
| **Maintenance** | 16, 18, 19 | Miner/balance fixes (slow, fast, or customizable batches) |
| **Diagnostics** | 17 | RPC smoke test |

Address tracking (options 9–10 and CLI `--track-address`) supports block ranges, mining-only vs regular filters, optional multiprocessing, RAM cache sizing, and per-chunk sleep — the same performance knobs as lookback.

## Logging and audit trail

Three log families rotate under the worker directory:

- **General trace** — block processing and RPC errors (`logs/dynex_log_*.log`)
- **Large-transaction highlight** — one line per qualifying transfer (`large_tx_logs/`)
- **Per-address traces** — optional deep logs when tracking wallets (`address_logs/`)

Terminal output uses colour when attached to a TTY; `--show-messages` controls verbose WordPress send banners.

## Headless and service deployment

For systemd or Universal Service Manager deployments, operators typically run with **`--auto-run`** (and often **`--auto-send`**) so the process skips the menu and enters the block loop immediately. Configuration is read from `dynex_config.ini` beside the script.

The worker assumes a **co-located Dynex daemon** reachable on the local RPC URL baked into the script. It does not expose an HTTP API of its own — WordPress is the only public consumer of indexed large transfers.

Private code: [dynex-large-transactions-monitor](https://github.com/logicencoder/dynex-large-transactions-monitor) · WordPress UI [dynex-large-transactions-plugin-overview](https://github.com/logicencoder/dynex-large-transactions-plugin-overview)

See [REPOS.md](REPOS.md).

---

**Made by [Logic Encoder](https://logicencoder.com)** · [GitHub](https://github.com/logicencoder) · [Contact](https://logicencoder.com/contact/)
