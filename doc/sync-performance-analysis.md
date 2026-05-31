# Block Sync Performance Analysis

This document analyzes block-synchronization performance in this ZClassic
codebase, explains the "Park block ... as it would cause a deep reorg" slowdown,
and lays out a prioritized set of improvements — including upstream
Bitcoin Core / Zcash changes that post-date this fork's baseline.

## 1. What baseline are we on?

- Client version: **2.1.2** (`src/clientversion.h`).
- Validation lives in a **monolithic `src/main.cpp`** — the upstream Bitcoin
  Core `net_processing.cpp` / `validation.cpp` split (Core 0.14–0.15, 2017) has
  **not** been merged.
- The newest network upgrade compiled in is **Sapling** (`src/consensus/upgrades.cpp`);
  Blossom/Heartwood/Canopy/NU5 are absent.

Conclusion: the validation/networking layer tracks **Zcash ~2.1.x, early 2020**,
which itself rebased Bitcoin Core ~0.11–0.12 networking. Several years of
sync-relevant upstream work is therefore missing.

## 2. The "Park block ... deep reorg" slowdown (root cause + fix)

### Where it comes from
The message is emitted in `AcceptBlock()` (`src/main.cpp`):

```cpp
if (GetBoolArg("-parkdeepreorg", true)) {
    const CBlockIndex *pindexFork = chainActive.FindFork(pindex);
    if (pindexFork && pindexFork->nHeight + 1 < pindex->nHeight) {
        LogPrintf("Park block %s as it would cause a deep reorg.\n", ...);
        pindex->nStatus |= BLOCK_PARKED_FLAG;
        setDirtyBlockIndex.insert(pindex);
    }
}
```

### This is NOT stock Zcash
`ParkBlock` / `BLOCK_PARKED_FLAG` / `-parkdeepreorg` were added to **this fork**
in commit `d57bf7a "Deep Reorg Protection (#25)"`, ported from **Bitcoin ABC**.
Stock Zcash (and Bitcoin Core) have no parking mechanism. So this slowdown is
specific to ZClassic, exactly as suspected.

### Why it murders sync speed
Parking is an **anti-51%-attack** defence designed for a node that is **already
at the network tip**: it makes the node refuse to reorg away from the chain it
saw first unless a competitor accumulates ~2× the work since the fork point
(`ActivateBestChainStep`, `src/main.cpp:3513`).

The trigger had **no `IsInitialBlockDownload()` guard**. During IBD a node:
- pulls blocks from many peers, frequently out of order;
- routinely sees short, valid side-forks of historical chain.

Every such block deeper than 1 from the current fork gets `BLOCK_PARKED_FLAG`,
which then forces the unpark / `requiredWork` accounting walk in
`ActivateBestChainStep`, plus `setDirtyBlockIndex` churn (extra disk writes).
When many parked candidates queue up, this repeats "MANY times in a row" and
throttles the connect loop — the symptom you reported.

### The fix (implemented on this branch)
Gate the parking heuristic on **not being in IBD**:

```cpp
if (GetBoolArg("-parkdeepreorg", true) && !IsInitialBlockDownload()) {
    ...
}
```

**Why this is safe (no consensus change):** parking never changes a block's
*validity* — a parked block stays fully valid and `ActivateBestChainStep` still
selects the most-work valid chain. Parking only changes *local fork-choice
tie-breaking* near the live tip. Skipping it during IBD restores plain
follow-most-work behavior (identical to stock Zcash) for the historical sync,
while the 51%-defence remains fully active once the node is synced. This is the
single highest-value, lowest-risk sync win and addresses the reported symptom
directly.

### Operational escape hatch
`-parkdeepreorg=0` already disables parking entirely. Worth documenting for
operators doing a big resync, but with the IBD guard it should no longer be
necessary.

## 3. Is sync already multi-threaded / parallel? (Yes, partially)

| Stage | Parallel today? | Notes |
|---|---|---|
| Block **download** | **Yes** | Headers-first; up to `MAX_BLOCKS_IN_TRANSIT_PER_PEER = 128` in flight per peer across a `BLOCK_DOWNLOAD_WINDOW = 4096` window (`src/main.h`). Fetched from multiple peers concurrently. |
| **Script/signature** verification | **Yes** | `CCheckQueue` with `-par` worker threads (`nScriptCheckThreads`, `ContextualCheckInputs` → `control.Add(vChecks)`). |
| Block **connect** (`ConnectTip`/`ConnectBlock`) | **No (serial)** | Inherently sequential — each block mutates the UTXO set for the next. This is the throughput ceiling and is serial in Bitcoin Core too. |
| Equihash **PoW** check on connect | Skipped below last checkpoint | Good. |

So download and signature-checking are already parallel. The connect loop is
serial by nature; the wins there come from **doing less redundant work** and
**larger caches**, not more threads.

## 4. Prioritized improvements

### Tier 1 — implemented here
1. **IBD guard on deep-reorg parking** (section 2). Direct fix for the reported
   slowdown.

### Tier 2 — high value, low risk (recommended next, not yet done)
2. **Raise default `-dbcache`.** `nDefaultDbCache = 450` MiB (`src/txdb.h`) is a
   2016-era default. A bigger UTXO/coins cache means far fewer flushes during
   IBD. The init code already auto-bumps on 64-bit (`src/init.cpp:2130`); raising
   the floor (e.g. 1–2 GiB when RAM allows) is a large, safe IBD speedup.
3. **`assumevalid`-style signature skipping** (Bitcoin Core #9484, post-baseline).
   Skip ECDSA/script checks for ancestors of a hardcoded recently-validated
   block hash. Today only blocks below the last *checkpoint* skip script checks
   (`fScriptChecks`); `assumevalid` extends that window dramatically. Biggest CPU
   win for IBD, and it does not weaken consensus (PoW + merkle still checked,
   and it's user-overridable).

### Tier 3 — larger merges, structural
4. **Backport the `net_processing` improvements** that followed this baseline:
   `compact blocks` (BIP152) help at the tip more than IBD, but the headers-sync
   and block-fetch scheduling refinements (e.g. better stalling-peer detection,
   `BLOCK_DOWNLOAD_WINDOW`/per-peer tuning) reduce sync stalls.
5. **Per-block UTXO prefetch / batched coin reads** during IBD to keep the
   serial connect loop fed (reduces LevelDB random-read latency).
6. **LevelDB tuning** (larger write buffer / block cache during IBD) — pairs with
   #2.

### Tier 4 — verify / tune existing knobs
7. Confirm `-par` defaults use all cores during IBD on multi-core machines.
8. Audit `DATABASE_WRITE_INTERVAL` / flush cadence (`FlushStateToDisk`) so we are
   not flushing too aggressively mid-IBD with a small cache.

## 5. Scope of the change on this branch

This branch implements **only Tier 1** (the IBD park guard) — a minimal,
consensus-neutral change that fixes the reported symptom. Tiers 2–4 are
documented here as a roadmap and can be taken in follow-up PRs, each
independently testable.
