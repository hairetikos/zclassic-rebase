# Design: Chaff-send — daemon-side support for decoy shielded churn

Status: **proposed / not yet implemented.** This document is the **node-side**
implementation plan that backs the chaff-send feature. The user-facing
orchestration (scheduler, plan, UI, encrypted run-state) is specified in the GUI
design and lives in the wallet GUI:

- GUI design (authoritative for UX/scheduler/threat model):
  <https://github.com/hairetikos/zclassic3-gui/blob/chaff-send/docs/CHAFF_SEND_DESIGN.md>

This plan covers what **zclassicd** must provide so that chaff-send is actually
private and correct, rather than leaky. Several requirements in the GUI doc are
explicitly flagged as *outside the GUI's control* — most importantly
"guaranteed per-broadcast circuits require **daemon-side stream isolation**." That
is node work, and it is the centerpiece here.

## 1. Why chaff-send needs the daemon

Chaff-send hides a single real shielded payment among many decoy `z→z` self-sends
with randomized timing and amounts. Its privacy claims only hold if the **network
broadcast** of each transaction is itself decorrelated. The GUI can schedule and
jitter `z_sendmany` calls all it likes, but the moment the daemon broadcasts them,
the following are determined by the *node*, not the GUI:

1. **Which Tor circuit / exit identity** each transaction is announced over.
   Without per-broadcast stream isolation, every chaff and the one real tx leave
   over the same circuit at known times — collapsing the anonymity set the GUI
   worked to build.
2. **The first-spy origin estimator** at the P2P layer (addressed structurally by
   [Dandelion++](dandelion-design.md), which this plan assumes as a sibling).
3. **Confirmation / async-op semantics** that the GUI's idempotent, fail-safe real
   send relies on (`z_sendmany` → opid → txid → confirmed).

So the node must deliver three things: (A) reliable per-transaction Tor stream
isolation for broadcasts, (B) RPC ergonomics the churn engine needs to run many
`z_sendmany` operations safely and idempotently, and (C) clean composition with
Dandelion++ so origin privacy is layered, not redundant.

## 2. What the node already provides

| Capability | Where | Status for chaff-send |
|---|---|---|
| Shielded multi-output send | `z_sendmany` `src/wallet/rpcwallet.cpp:3635`; `asyncrpcoperation_sendmany.{h,cpp}` | Usable as-is; async opid model fits the engine. |
| Fresh shielded address per hop | `z_getnewaddress` `src/wallet/rpcwallet.cpp:3151` | Backs the GUI "fresh address per hop" default. |
| Async op queue + polling | `asyncrpcqueue.{cpp,h}`, `asyncrpcoperation.{cpp,h}` | Backs `z_getoperationstatus` polling / state machine. |
| SOCKS5 with per-connection auth | `Socks5()` `src/netbase.cpp:331`; `ProxyCredentials` `:324` | **Foundation** for stream isolation (already supports user/pass). |
| Proxy credential randomization | `proxyType.randomize_credentials` `src/netbase.h:195`, used at `src/netbase.cpp:613` | Per-*connection* isolation exists; per-*broadcast* does not yet. |
| Tor control | `src/torcontrol.cpp` | Manages the hidden service; does not rotate broadcast streams. |
| Encrypted wallet secrets | `src/wallet/crypter.{cpp,h}` | Pattern to reuse for the GUI's encrypted run-state at rest. |

The gap is precise: we have SOCKS5 username/password **stream isolation by
connection**, but a transaction broadcast reuses whatever peer connections already
exist. There is no mechanism to broadcast a *specific* transaction over a
*fresh, isolated* circuit on demand. Closing that gap is the core node deliverable.

## 3. Threat model (node-side slice)

Inherited from the GUI doc; the node is responsible for the network-layer slice:

- **Defends:** broadcast-time correlation (real vs chaff over distinguishable
  circuits), and origin-IP linkage of the broadcast event (with Dandelion++).
- **Does not defend:** on-chain cryptographic anonymity (already strong for
  shielded), recipient-side leaks, or a global passive adversary who deanonymizes
  Tor itself. Surface this honestly; the node must not imply guarantees it can't keep.

## 4. Node deliverables

### 4.1 Per-broadcast Tor stream isolation (the centerpiece)

Goal: each chaff transaction — and the real one — is announced to the network over
a **distinct Tor circuit**, so an exit/relay observer cannot bucket them together
by circuit identity or correlate the real send's circuit with its timing.

Design options, in increasing daemon involvement:

1. **Isolated broadcast connection per tx (preferred).** Add a relay path that,
   for a flagged transaction, opens a *new* outbound connection to one (or a few)
   peers using **fresh randomized SOCKS5 credentials** (new circuit via Tor's
   `IsolateSOCKSAuth`), pushes the `tx`/`dandeliontx` once, and tears the
   connection down. This reuses the existing `ProxyCredentials` randomization
   (`src/netbase.cpp:613`) but drives it *per broadcast* instead of per long-lived
   peer. Bound concurrency and rate to avoid a distinctive burst signature.
2. **Control-port circuit hints.** Use `torcontrol.cpp` to request/attach streams;
   heavier, more Tor-version-dependent, and entry guards are sticky by design.
   Keep as a fallback, not the primary mechanism.

Expose this as a broadcast option rather than a global mode, because the GUI needs
*some* sends (the chaff stream) isolated while not disrupting normal operation.

### 4.2 RPC surface for the churn engine

The GUI engine needs to drive many sends idempotently and observe them. Add a thin
RPC layer (no consensus impact) so the engine does not have to re-implement node
internals:

- **`z_sendmany` with an isolation flag** (or a sibling `z_sendmany_isolated`):
  request that the resulting broadcast use a fresh isolated circuit per §4.1.
  Default off → behaviour unchanged for everyone else.
- **Operation tracking** already exists (`z_getoperationstatus` /
  `z_getoperationresult`); confirm it surfaces `opid → txid` and a terminal
  `confirmed` state the GUI's `pending → submitted → broadcast → confirmed`
  machine can poll. Add the txid to the success result if not already present.
- **Optional broadcast gating:** a way to ask the node to *build and sign* now but
  *broadcast at* the engine's jittered fire-time, so the broadcast timing (the
  privacy-sensitive event) is controlled to the second by the schedule rather than
  by async-queue latency. Minimal version: the engine simply calls `z_sendmany`
  at fire-time and accepts queue jitter; richer version: a "prepare then
  broadcast" two-step. Start minimal; revisit if queue latency proves to add a
  fingerprintable skew.

### 4.3 Composition with Dandelion++

Per-broadcast circuit isolation (§4.1) decorrelates the **transport**;
Dandelion++ (see [dandelion-design.md](dandelion-design.md)) decorrelates the
**P2P origin**. They compose cleanly and should be enabled together for chaff
broadcasts: a chaff tx enters the stem over its own isolated circuit. Neither
subsumes the other — an isolated circuit still announces to a *direct* peer that
can run the first-spy estimator; Dandelion moves the apparent origin away from us.

### 4.4 Encrypted run-state (shared primitive)

The GUI persists an encrypted plan (real-send nonces + confirmation flags only, no
plaintext recipients) for crash-safe idempotency. The daemon already has the
crypto primitive pattern in `src/wallet/crypter.{cpp,h}` (passphrase-derived key,
authenticated encryption). No node API is strictly required if the GUI owns the
file, but we should ensure the wallet passphrase / lock state the GUI keys against
is queryable so the GUI can refuse to run (or run ephemeral-only) when locked.

## 5. Parameters (node-side)

| Parameter | Default | Purpose |
|---|---|---|
| `-chaffisolation` (or per-call flag) | off | Enable per-broadcast Tor stream isolation. |
| Isolated-broadcast peer fan-out | 1–2 | How many peers receive the one-shot isolated push. |
| Isolated-broadcast max concurrency | small | Cap simultaneous isolated circuits to avoid a burst signature. |
| `-dandelion` | off (see sibling doc) | Origin-layer privacy; recommended on with chaff. |
| Require Tor for isolated broadcast | on | Refuse isolated-broadcast if no SOCKS proxy is configured (fail closed). |

The engine-level parameters (window, steps, fan-out, denomination ladder,
real-exit band, etc.) remain in the GUI design doc and are **not** node config.

## 6. Phased implementation

**P0 — Foundations, no behaviour change.** Audit and document the existing SOCKS5
randomization path (`src/netbase.cpp:613`) and confirm Tor `IsolateSOCKSAuth`
semantics with our proxy setup on testnet. Add a "fail closed" check: refuse the
new isolated-broadcast path when no proxy is configured. Confirm `z_getoperationresult`
exposes the txid.

**P1 — Per-broadcast isolated relay.** Implement the one-shot isolated outbound
broadcast (§4.1 option 1): open → fresh randomized creds → push tx → close, with
concurrency/rate caps. Unit + regtest tests asserting each isolated broadcast uses
distinct SOCKS credentials. This is the single most important node deliverable.

**P2 — RPC ergonomics.** Add the `z_sendmany` isolation flag / `z_sendmany_isolated`
and confirm the opid→txid→confirmed surface the GUI state machine needs. Document
the contract the GUI relies on (idempotency is the GUI's responsibility via
persisted nonces; the node guarantees at-most-once *broadcast* per accepted op).

**P3 — Composition & hardening.** Enable Dandelion++ stem entry for isolated chaff
broadcasts; add metrics (count of isolated broadcasts, distinct circuits); soak on
testnet with the GUI engine driving a real churn window; verify no distinctive
burst/period signature at the network layer.

## 7. Testing

- **Unit:** isolated-broadcast path uses fresh randomized SOCKS creds each call;
  fail-closed when no proxy; opid→txid mapping in op results.
- **Regtest/testnet first** (per the GUI doc): fast blocks, free coins; never
  rehearse on mainnet. Drive the GUI engine against a local node; confirm chaff +
  one real send all confirm and that the real send fires exactly once across an
  induced crash/restart (idempotency end-to-end).
- **Network observation:** with a cooperating Tor test setup, confirm distinct
  circuits per broadcast and that timing matches the engine's jittered schedule,
  not the async-queue cadence.

## 8. Risks and limitations (node-side)

- **Stream isolation is best-effort.** Tor entry guards are sticky by design;
  per-broadcast *circuits* (distinct exits) are achievable via `IsolateSOCKSAuth`,
  but the daemon cannot guarantee fully independent paths end to end. Document the
  exact guarantee we provide and do not oversell it — this is precisely the gap the
  GUI doc deferred to the daemon, and we must describe what we actually deliver.
- **Connection-churn signature.** Opening many short-lived isolated connections is
  itself observable. Cap concurrency/rate and prefer reusing a small set of
  Dandelion stem peers so the isolated broadcasts blend with normal relay.
- **Fee and throughput** are inherited from the engine/chain (block time, ZIP-317)
  and unchanged by the node work.
- **Not a defense against a global passive Tor adversary.** State plainly in any
  surface that mentions chaff-send. The node raises correlation cost; it does not
  grant invisibility.
