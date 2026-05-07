# CASS Semantic Index Build

## What this does

Embeds all 93K messages (1,165 conversations) into vectors using the
`all-MiniLM-L6-v2` model so CASS can do meaning-based search, not just
keyword matching. One-time heavy build; incremental after that.

## Prerequisites

Already done — nothing to install:

- [x] CASS installed (`~/.local/bin/cass`)
- [x] Model downloaded (`cass models install` — 23MB, stored in `~/Library/Application Support/com.coding-agent-search.coding-agent-search/models/`)
- [x] Lexical index built and current
- [x] Cron job: hourly 6am–8pm, lexical only

## The build

```bash
# Close memory-heavy apps (browsers, IDEs) first — this will use ~6GB RAM
# Run from any directory. Expect 15-30 min on this machine (8-core, 24GB).

cass index --semantic 2>&1 | tee /tmp/cass-semantic-build.log &
```

`&` backgrounds it. Monitor with:
```bash
# CPU/memory
ps aux | grep "cass index" | grep -v grep

# Progress (cass logs to stderr)
tail -f /tmp/cass-semantic-build.log
```

## Resource expectations

| Metric | Observed (killed at ~17 min) | Expected full run |
|--------|------------------------------|-------------------|
| CPU | 600%+ (all 8 cores) | Same — fastembed is parallel |
| RAM | ~6 GB peak | ~6 GB peak |
| Duration | Was ~17 min when killed | 20-40 min estimate |
| Disk | — | ~200-400 MB for vector index |

## After the build

Verify:
```bash
cass status
# Should show: Semantic: Status: ready

# Test semantic search
cass search "preview branch seeding bootstrap" --mode semantic --limit 3
```

Then update the cron to include semantic in future incremental runs:
```bash
crontab -e
# Change to:
0 6-20 * * * /Users/hsingh/.local/bin/cass index --semantic --json > /dev/null 2>&1
```

Incremental semantic runs (only new messages) should be fast — seconds, not minutes.

## If it fails or hangs

```bash
# Kill it
pkill -f "cass index"

# Check for corrupt state
cass doctor --fix

# Retry
cass index --semantic
```

## Why we killed it earlier

First build was launched mid-session (2026-05-07 ~10:55am) while actively
using the machine. 6GB RAM + 600% CPU made everything slow. Better to run
in the evening with other apps closed.
