# Octra Chain → BigQuery + Stats Dashboard

Indexes the Octra blockchain into Google BigQuery and publishes a pre-aggregated
stats JSON to Google Cloud Storage, which powers the `stats.html` dashboard in
the Octra wallet (webcli).

---

## Architecture

```
Octra RPC  ──▶  backfill.py  ──▶  BigQuery (octra-data.devnet)
                     │
                stats_export.py  ──▶  gs://octra-data/devnet/stats.json  (public)
                                              │
                                       stats.html fetches this
                                       (no auth, no BigQuery access from browser)
```

The browser never touches BigQuery. `stats.json` is a static public file served
by Google Cloud Storage. BigQuery is only accessed by the Cloud Run Function.

---

## GCP Resources

| Resource | Name |
|---|---|
| Project | `octra-data` |
| BigQuery dataset | `devnet` |
| GCS bucket | `octra-data` (public read) |
| Stats JSON | `gs://octra-data/devnet/stats.json` |
| Public URL | `https://storage.googleapis.com/octra-data/devnet/stats.json` |
| Cloud Run Function | `devnet-backfill` |
| Cloud Scheduler | triggers function every 5 min |
| RPC node | `https://devnet.octra.com/rpc` |

---

## BigQuery schema

**Project:** `octra-data`  **Dataset:** `devnet`

### `epochs`
One row per finalized epoch.

| Column | Type | Description |
|---|---|---|
| `epoch_id` | INT64 | Epoch number |
| `finalized_at` | TIMESTAMP | When the epoch was finalized |
| `state_root` | STRING | Parent commit hash (`parent_commit` from RPC) |
| `tx_count` | INT64 | Confirmed transactions |
| `rejected_count` | INT64 | Rejected transactions |
| `validator` | STRING | Address that finalized the epoch (`finalized_by`) |
| `roots` | INT64 | Merkle roots count |
| `ingested_at` | TIMESTAMP | When this row was written |

Partitioned by `DATE(finalized_at)`, clustered by `epoch_id`.

---

### `transactions`
One row per transaction (confirmed and rejected).

| Column | Type | Description |
|---|---|---|
| `tx_hash` | STRING | 64-char hex hash |
| `epoch_id` | INT64 | Epoch the tx was included in |
| `tx_timestamp` | TIMESTAMP | Timestamp from the transaction itself |
| `from_address` | STRING | Sender address |
| `to_address` | STRING | Recipient address |
| `amount_raw` | INT64 | Amount in raw units (divide by 1,000,000 for OCTR) |
| `nonce` | INT64 | Sender nonce at submission |
| `ou` | INT64 | Operation Units paid (fee proxy) |
| `op_type` | STRING | `Transfer`, `Deploy`, `Call`, etc. |
| `status` | STRING | `confirmed` or `rejected` |
| `message` | STRING | Optional memo |
| `has_encrypted_data` | BOOL | Whether transaction carried encrypted payload |
| `has_stealth` | BOOL | Whether transaction used a stealth address |
| `public_key` | STRING | Ed25519 public key used in signing |
| `ingested_at` | TIMESTAMP | When this row was written |

Partitioned by `DATE(tx_timestamp)`, clustered by `(op_type, from_address)`.

---

### `network_snapshots`
Periodic snapshots of global chain state, written once every 500 epochs.

| Column | Type | Description |
|---|---|---|
| `snapshot_at` | TIMESTAMP | When the snapshot was taken |
| `epoch_id` | INT64 | Chain tip at snapshot time |
| `total_accounts` | INT64 | All-time registered accounts |
| `active_accounts` | INT64 | Accounts with activity |
| `total_supply_raw` | INT64 | Circulating supply in raw units |
| `max_supply_raw` | INT64 | Max supply cap in raw units |
| `burned_raw` | INT64 | Burned supply in raw units |
| `total_transactions` | INT64 | Cumulative confirmed transactions |
| `staging_size` | INT64 | Pending transactions in mempool |
| `recent_tx_count` | INT64 | Transactions in recent epochs |
| `ingested_at` | TIMESTAMP | When this row was written |

Partitioned by `DATE(snapshot_at)`.

---

### `checkpoint`
Single-row state table used for resumable backfill.

| Column | Type | Description |
|---|---|---|
| `id` | STRING | Always `'main'` |
| `last_epoch` | INT64 | Last fully processed epoch |
| `updated_at` | TIMESTAMP | When the checkpoint was last updated |

---

## Files

```
backfill.py        Backfill script — fetches epochs from RPC, writes to BigQuery.
                   Also serves as the Cloud Run Function entry point (main(request=None)).
                   Calls stats_export.main() automatically on completion.

stats_export.py    Reads aggregated stats from BigQuery, writes stats.json to GCS.
                   Runs at the end of every backfill.

control_panel.py   Local utility to inspect/manage the BigQuery tables.

requirements.txt   Python dependencies.

Dockerfile         Container image (used for Cloud Run Job deploys, not the function).

deploy.bat         Build, push, create/update Cloud Run Job, run a test execution.

run_full.bat       Switch to full backfill (24h timeout) and execute.
```

---

## stats.json format

```json
{
  "generated_at": "2026-03-15T20:51:07+00:00",
  "stats": {
    "total_transactions": 364219,
    "total_accounts": 130309,
    "supply": 984501446.0,
    "epoch_id": 190500
  },
  "charts": {
    "MONTH": [{"bucket": "2025-06", "count": 223511, "volume": 68628898.19}, ...],
    "WEEK":  [{"bucket": "2025-06-01", "count": 12400, "volume": 1200000.0}, ...],
    "DAY":   [{"bucket": "2025-06-01", "count": 1800, "volume": 180000.0}, ...]
  }
}
```

Chart timeframe granularities:

| Key | Bucket size | Window |
|---|---|---|
| `MONTH` | monthly | all time |
| `WEEK` | weekly (Sun) | all time |
| `DAY` | daily | all time |

---

## Cloud Run Function deployment

The backfill runs as a Cloud Run Function triggered by Cloud Scheduler every 5 minutes.

- **Entry point:** `main`
- **Runtime:** Python 3.12
- **Trigger:** HTTP (Cloud Scheduler POST)
- **Schedule:** `*/5 * * * *`

Service account needs these IAM roles on `octra-data`:
- `roles/bigquery.dataEditor`
- `roles/bigquery.jobUser`
- `roles/storage.objectAdmin` (to write stats.json)

---

## Local run

```powershell
pip install google-cloud-bigquery google-cloud-storage requests
gcloud auth application-default login

$env:GCP_PROJECT="octra-data"
$env:BQ_DATASET="devnet"
python backfill.py
```

To run only the stats export:

```powershell
python stats_export.py
```

---

## GCS bucket setup (one-time)

```bash
gcloud storage buckets create gs://octra-data --location=US --uniform-bucket-level-access
gcloud storage buckets add-iam-policy-binding gs://octra-data --member=allUsers --role=roles/storage.objectViewer
cat > cors.json << 'EOF'
[{"origin":["*"],"method":["GET"],"responseHeader":["Content-Type"],"maxAgeSeconds":3600}]
EOF
gcloud storage buckets update gs://octra-data --cors-file=cors.json
```

---

## Cloud Run Job deployment (alternative to function)

```bat
deploy.bat
```

For a full backfill from genesis:

```bat
run_full.bat
```

The full run sets `task-timeout=86400` (24 hours). Resumable — re-running is always safe.

---

## RPC methods used

| Method | Purpose |
|---|---|
| `epoch_current` | Get current chain tip epoch ID |
| `epoch_get` | Metadata for a specific epoch |
| `epoch_list` | Paginated list of all epoch IDs |
| `octra_transactionsByEpoch` | All transactions in a specific epoch |
| `node_stats` | Network-wide stats (accounts, supply, tx count) |
| `node_metrics` | Performance metrics (volume, fees, TPS) |

**Nodes:**
- Devnet: `https://devnet.octra.com/rpc`
- Mainnet: `http://46.101.86.250:8080/rpc`

---

## Useful queries

**Transactions per day:**
```sql
SELECT
  DATE(tx_timestamp)       AS day,
  COUNT(*)                 AS tx_count,
  SUM(amount_raw) / 1e6    AS volume_octr
FROM `octra-data.devnet.transactions`
WHERE status = 'confirmed'
GROUP BY 1
ORDER BY 1;
```

**Account growth over time:**
```sql
SELECT
  DATE(snapshot_at)        AS day,
  MAX(total_accounts)      AS total_accounts
FROM `octra-data.devnet.network_snapshots`
GROUP BY 1
ORDER BY 1;
```

**Top senders all-time:**
```sql
SELECT
  from_address,
  COUNT(*)              AS tx_count,
  SUM(amount_raw) / 1e6 AS total_sent_octr
FROM `octra-data.devnet.transactions`
WHERE status = 'confirmed'
GROUP BY 1
ORDER BY 2 DESC
LIMIT 20;
```
***For webcli it will require this in main.cpp:**
```cpp
int main(int argc, char** argv) {
#ifdef _WIN32
    SetErrorMode(SEM_FAILCRITICALERRORS | SEM_NOGPFAULTERRORBOX);
    SetConsoleCtrlHandler([](DWORD) -> BOOL {
        handle_signal(0);
        return TRUE;
    }, TRUE);
#else
    struct rlimit rl = {0, 0};
    setrlimit(RLIMIT_CORE, &rl);
#ifdef __linux__
    prctl(PR_SET_DUMPABLE, 0);
#endif
    signal(SIGTERM, handle_signal);
    signal(SIGINT, handle_signal);
#endif

    int port = 8420;
    if (argc > 1) port = atoi(argv[1]);
    if (port <= 0) port = 8420;

    octra::ensure_data_dir();

    httplib::Server svr;
    svr.set_read_timeout(300, 0);
    svr.set_write_timeout(300, 0);

    svr.set_post_routing_handler([](const httplib::Request&, httplib::Response& res) {
        res.set_header("X-Frame-Options", "DENY");
        res.set_header("X-Content-Type-Options", "nosniff");
        res.set_header("Content-Security-Policy",
            "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; connect-src 'self' https://storage.googleapis.com");
        res.set_header("Cache-Control", "no-store");
    });
```