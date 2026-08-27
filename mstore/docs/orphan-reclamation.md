# Orphan Data Reclamation

## Problem

Before this feature, deleting an object removed only the metadata (xl.meta / RocksDB record). The data files on disk (`part.N` / erasure shards) were left behind with no mechanism to reclaim them, causing disk usage to grow unboundedly after deletes.

## How It Works

MStore now uses a **two-phase delete** backed by a durable journal:

1. **Atomic delete (hot path).** A single RocksDB `WriteBatch` simultaneously removes the object metadata and writes a `pending_delete` journal row. The journal row records the exact data-file paths that must be removed. This is crash-safe: the batch either commits in full or not at all. Physical files are never deleted inline, so delete latency is unchanged.

2. **Reclaim worker (background).** An async task in `mstore-object/src/reclaim.rs` drains the journal:
   - Woken by an event-bus signal on every delete **and** by a periodic tick (see `tick_secs`).
   - On startup, the worker replays the entire journal — crash replay is automatic.
   - Each pass reads up to `batch_size` journal rows (oldest first), calls `remove_file` on each target (idempotent — "not found" is silently ignored), then removes the journal row on success.
   - Rows that fail (e.g. I/O error on a degraded drive) are retained and retried on the next pass.
   - Throughput is throttled by `max_files_per_sec` to avoid saturating disks under foreground I/O.

## Configuration (`[reclaim]`)

The `[reclaim]` section is **optional** — omitting it means all defaults apply. The reclaim worker always runs regardless.

```toml
# Öksüz veri geri kazanım worker'ı (opsiyonel — tümü varsayılan)
[reclaim]
tick_secs = 30                 # worker uyanma aralığı (sn)
max_files_per_sec = 10000      # throttle: saniyede en fazla silinen dosya
batch_size = 1000              # bir turda işlenen pending satır sayısı
deep_scan_min_age_secs = 86400 # deep purge: bu yaştan genç metadatasız dosya öksüz sayılmaz
```

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `tick_secs` | `u64` | `30` | Worker periodic wake-up interval (seconds). The event bus also wakes the worker immediately on each delete. |
| `max_files_per_sec` | `u64` | `10000` | Throttle: maximum data files deleted per second. `0` means unlimited (not recommended in production). |
| `batch_size` | `usize` | `1000` | Number of pending-delete journal rows processed per pass. |
| `deep_scan_min_age_secs` | `u64` | `86400` | Deep-scan grace period (seconds). Files without metadata that are younger than this threshold are not treated as orphans — protects in-flight multipart uploads from being deleted prematurely. |

## Admin API

Two HTTP endpoints are available under the MStore admin API (root credentials required):

### `POST /mstore/api/v1/orphans/scan`

Returns a report of pending-delete journal entries and (optionally) a deep disk scan.

**Request body (JSON):**
```json
{
  "bucket": "your-bucket",
  "deep": false,
  "dry_run": true
}
```

All fields are optional. `bucket` restricts the scan to a single bucket. `deep` triggers a disk↔metadata comparison to find orphans that predate the journal (legacy orphans). `dry_run` is accepted but has no effect for a scan (read-only).

**Response (JSON):**
```json
{
  "pendingCount": 42,
  "deepOrphans": 0,
  "deepBytes": 0
}
```

`deepOrphans` and `deepBytes` are populated only when `deep: true`.

### `POST /mstore/api/v1/orphans/purge`

Drains the journal and/or removes deep-scan orphans. **Defaults to dry run** — pass `dry_run: false` to perform actual deletions.

**Request body (JSON):**
```json
{
  "bucket": "your-bucket",
  "deep": false,
  "dry_run": true
}
```

**Response (JSON):**
```json
{
  "pendingCount": 42,
  "deepOrphans": 0,
  "deepBytes": 0
}
```

Every purge is written to the audit log regardless of log level.

## CLI

```
mstore orphans scan|purge [OPTIONS]

Options:
  --bucket <BUCKET>            Restrict to a single bucket
  --deep                       Include deep disk↔metadata scan for legacy orphans
  --no-dry-run                 Actually delete files (purge only; default is dry run)
  --admin-endpoint <URL>       Admin API endpoint (default: http://127.0.0.1:9010)
```

### Examples

```bash
# Report journal backlog and deep orphans for all buckets
mstore orphans scan --deep

# Report for a specific bucket only
mstore orphans scan --deep --bucket your-bucket

# Dry-run purge (safe — no files deleted)
mstore orphans purge --deep --bucket your-bucket

# Actually delete legacy orphans in a bucket
mstore orphans purge --deep --no-dry-run --bucket your-bucket
```

## One-Time Legacy Cleanup

If the cluster was running before this feature was introduced, existing orphaned files on disk are not in the journal and will not be reclaimed by the background worker. Run a one-time deep purge to clear them:

```bash
# 1. Inspect first (dry run)
mstore orphans scan --deep [--bucket NAME]

# 2. Delete after confirming the count looks correct
mstore orphans purge --deep --no-dry-run [--bucket NAME]
```

The deep scan respects `deep_scan_min_age_secs` from config (default 24 h), so in-flight uploads are never mistakenly deleted.

After this one-time cleanup, all subsequent deletes go through the journal; routine deep scans are not needed.
