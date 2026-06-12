# move_data

Starts IBM Storage Protect (TSM) `MOVE DATA` for a tape volume in a source storage pool. The script is intended to run repeatedly from cron: each invocation picks one matching volume and starts a move to the pool's next storage pool.

## Prerequisites

### Next storage pool must be defined

The source storage pool **must have a next storage pool configured** in TSM. The script reads it with:

```sql
SELECT NEXTSTGPOOL FROM STGPOOLS WHERE STGPOOL_NAME = '<stgpool>'
```

If `NEXTSTGPOOL` is empty or not set for the given pool, no move is started and the log will report `no tape to move found in stgpool=...`.

Configure the next pool in TSM before using this script, for example:

```text
UPDATE STGPOOL POOL_SOURCE NEXTSTGPOOL=POOL_TARGET
```

The move target is always the configured next pool; it cannot be overridden on the command line.

### TSM instance and credentials

- The TSM server instance (e.g. `tsminst1`) must be running on the host.
- Pass the instance name with `-i`; the script connects via `dsmadmc -se=<instance>`.
- No opt file is required. `dsmadmc` may emit `ANS0990W` about a missing default `dsm.opt`; this is harmless and is filtered from the log.
- Files under `~/scripts/move_data/`:
  - `tsm_cred` — credentials file for `dsmadmc`
  - `tsm_tapes.out` — main log (script messages and TSM command output)
  - `tsm_tapes.volumes` — volume names successfully started (one per line)

## Usage

```text
move_data -i <tsm_instance> -s <stgpool> [options]
move_data -h
```

### Options

| Option | Description |
|--------|-------------|
| `-i` | TSM server instance name (**required**, e.g. `tsminst1`) |
| `-s` | Source storage pool name (**required**) |
| `-m` | Mediatype filter, comma-separated without spaces (e.g. `3592-JC,3592-JD`). Uses a join to `libvolumes`. |
| `-p` | Only volumes with `PCT_UTILIZED` below this value |
| `-v` | Volume name filter: exact name or SQL `LIKE` pattern (e.g. `A%`) |
| `-M` | Max number of concurrent `Move Data` processes (fixed limit) |
| `-w` | Active time window as `start-end` hours, 24h format (e.g. `20-2` = 20:00–01:59). Requires `-M` and `-I`. |
| `-I` | Max `Move Data` processes outside the time window (requires `-w`) |

### Process limits

| Configuration | Behaviour |
|---------------|-----------|
| No `-M`, no `-w` | No process limit; each run attempts to start a move |
| `-M <n>` only | At most `<n>` concurrent `Move Data` processes at all times |
| `-w <window> -M <n> -I <m>` | Up to `<n>` processes inside the window, up to `<m>` outside |

Overnight windows are supported (e.g. `-w 20-2` covers 20:00 through 01:59).

## Volume selection

The script queries matching volumes in the source pool, ordered by `PCT_UTILIZED`, then picks one at random from the result set.

Without `-m`:

```sql
SELECT VOLUME_NAME FROM volumes
WHERE stgpool_name = '<stgpool>'
  [AND PCT_UTILIZED < <pct>]
  [AND VOLUME_NAME = '<name>' | LIKE '<pattern>']
ORDER BY PCT_UTILIZED
```

With `-m` (join to `libvolumes`):

```sql
SELECT v.VOLUME_NAME FROM volumes v
JOIN libvolumes l ON v.VOLUME_NAME = l.VOLUME_NAME
WHERE v.STGPOOL_NAME = '<stgpool>'
  AND l.MEDIATYPE IN ('<type>', ...)
  [AND v.PCT_UTILIZED < <pct>]
  [AND v.VOLUME_NAME = '<name>' | LIKE '<pattern>']
ORDER BY v.PCT_UTILIZED
```

A volume is moved only if its name matches the pattern `[A-Z][0-9]{5}` (e.g. `A12345`) and a next storage pool is defined.

## What the script does

1. Verify the TSM instance is running.
2. Optionally check the current number of `Move Data` processes against the configured limit.
3. Select a volume using the filters above.
4. Resolve `NEXTSTGPOOL` for the source pool.
5. Set the volume to read-only: `UPD VOL <volume> ACCESS=READONLY`
6. Start the move: `MOVE DATA <volume> STG=<next_stgpool>`

## Examples

```bash
# Move any eligible volume from the pool (no process limit)
move_data -i tsminst1 -s POOL_SOURCE

# Only low-utilization 3592-JD volumes whose names start with A
move_data -i tsminst1 -s POOL_SOURCE -m 3592-JD -p 80 -v A%

# Fixed limit of 6 concurrent move processes
move_data -i tsminst1 -s POOL_SOURCE -M 6

# Higher throughput overnight, lower during the day
move_data -i tsminst1 -s POOL_SOURCE -w 20-2 -M 8 -I 2
```

## Cron

The script is designed to run from cron every few minutes.

Recommended entry with overnight window (20:00–01:59, max 8 processes) and daytime idle limit (max 2):

```cron
1,2,3,4,5 * * * * /home/tsmuser/scripts/move_data/move_data -i tsminst1 -s POOL_SOURCE -w 20-2 -M 8 -I 2
```

Logging is handled by the script itself; a cron redirect (`>> .../tsm_tapes.out 2>&1`) is not required.

Run as the TSM instance owner. Adjust paths to match your deployment (`WORK_DIR` defaults to `~/scripts/move_data/` relative to that user).

Inside the active window (`-w 20-2`), up to 8 `Move Data` processes are allowed; outside 20:00–01:59 only 2 are allowed. With no matching volume or when the limit is reached, the script exits quietly after logging.

## Logging

The script writes two files under `~/scripts/move_data/` (or the configured `WORK_DIR`):

| File | Content |
|------|---------|
| `tsm_tapes.out` | Full run log: timestamped script messages and raw `dsmadmc` output |
| `tsm_tapes.volumes` | Volume name only, appended when `UPD VOL` and `MOVE DATA` both succeed |

Example `tsm_tapes.out`:

```text
20260101_14:30:00 starting instance=tsminst1 stgpool=POOL_SOURCE mediatype=none pct_utilized=none volume_name=any
20260101_14:30:02 move data volume=A12345 from=POOL_SOURCE to=POOL_TARGET
ANR2405E UPDATE VOLUME: Volume A12345 is currently in use by clients and/or data management operations.
ANR2212I UPDATE VOLUME: No volumes updated.
ANS8001I Return code 11.
20260101_14:30:03 skip volume=A12345 in use by clients or data management, upd vol not possible
20260101_14:30:03 end
```

Known TSM conditions are summarized after the raw `dsmadmc` output:

| TSM message | Log entry |
|-------------|-----------|
| ANR2405E (volume in use) | `skip volume=... in use by clients or data management, upd vol not possible` |
| ANR2416E (move already running) | `skip volume=... move data already in progress` |

If `UPD VOL` fails, `MOVE DATA` is not attempted. Other failures are logged as `ERROR ... failed volume=... rc=...`.

Example `tsm_tapes.volumes` (only successful starts):

```text
A12345
B67890
```

Nothing is written to stdout during a normal cron run. When run manually in a terminal, the same log lines and `dsmadmc` output are echoed for debugging (`[ -t 1 ]`). Validation errors before logging is initialized still go to stderr.
