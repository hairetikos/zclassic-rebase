# Design: Dandelion++ transaction-origin privacy (BIP-156)

Status: **proposed / not yet implemented.** This document is an implementation
plan for adding Dandelion++ stem-and-fluff transaction relay to the Zclassic
node, behind an off-by-default flag, with graceful fallback to ordinary
diffusion for non-supporting peers.

References:
- BIP-156 — Dandelion++: <https://github.com/bitcoin/bips/blob/master/bip-0156.mediawiki>
- Bitcoin Optech topic: <https://bitcoinops.org/en/topics/dandelion/>
- Original paper: Fanti et al., "Dandelion++: Lightweight Cryptocurrency
  Networking with Formal Anonymity Guarantees" (2018).

## 1. Why

Today every transaction this node originates or relays is announced to *all*
peers more or less simultaneously via symmetric diffusion (`RelayTransaction`
→ `vInventoryToSend` → `INV` flood in `SendMessages`). A network adversary that
maintains enough connections can run a **first-spy / first-timestamp** estimator:
the peer that first hears an `INV` for a transaction is, with high probability,
adjacent to (or is) the origin. This links a transaction — and therefore a
shielded send's *broadcast event* — to an IP address, defeating much of the
network-layer privacy that Tor and shielded addresses are meant to provide.

Dandelion++ breaks the symmetry of diffusion. A new transaction first travels
along a randomized, low-fan-out **stem** (each hop relays it to a *single*
pseudo-randomly chosen peer) before "fluffing" into ordinary diffusion at a
random hop. The first-spy estimator no longer points at the origin; it points at
whatever node happened to fluff, which is some random hops away. The formal
result (BIP-156) is a substantially flatter origin-probability distribution
against a botnet/spy adversary.

This is the **P2P-layer** complement to the wallet-layer chaff-send churn (see
[chaff-send-design.md](chaff-send-design.md)) and to Tor transport: chaff-send
decorrelates *which* transaction and *when*; Dandelion++ decorrelates *from which
IP* a transaction first appears. Each closes a different correlation channel.

## 2. Scope and non-goals

In scope:
- Stem/fluff relay state machine over a new P2P message (`dandeliontx`).
- Per-node, per-epoch stem routing with `DANDELION_MAX_DESTINATIONS = 2`.
- A separate **stempool** so stem transactions are validated but not yet exposed
  in the public `mempool` / not announced by ordinary `INV`.
- Per-transaction **embargo timers** with fail-safe fluffing.
- Backward compatibility: peers that do not advertise support receive ordinary
  diffusion; stem transactions never get stuck.
- Off-by-default rollout (`-dandelion=0` initially), opt-in for testing.

Out of scope (explicit non-goals):
- Any consensus change. Dandelion++ is relay policy only; the wire format of a
  *fluffed* transaction is the ordinary `tx` message and is unchanged.
- Protecting against a **global passive adversary** who already deanonymizes the
  transport. Dandelion++ raises the cost of the first-spy estimator; it is not
  invisibility. State this plainly anywhere we surface the feature.
- Changing mempool acceptance / fee / standardness rules. Stem transactions are
  validated by the *same* `AcceptToMemoryPool` checks before being relayed.

## 3. Where this lands in the current code

| Concern | Existing code | Change |
|---|---|---|
| Wire message types | `src/protocol.{h,cpp}` `NetMsgType` (already extended for `BSMAN`/`BSCHK` etc.) | Add `DANDELIONTX` ("`dandeliontx`"). |
| Per-peer relay state | `CNode` in `src/net.h` (`vInventoryToSend`, `setInventoryKnown`) | Add stem routing fields + Dandelion support flag. |
| Relay entry point | `RelayTransaction()` in `src/net.cpp:1856`; wallet/ATMP callers | Route new local txs into the stem instead of straight to diffusion. |
| Inbound tx handling | `strCommand == "tx"` in `src/main.cpp:6535`; `AcceptToMemoryPool` `src/main.cpp:1417` | Add `dandeliontx` handler; ATMP into stempool. |
| Relay scheduling | `SendMessages` INV loop `src/main.cpp:7250` | Drive embargo timers + stem forwarding. |
| Pools | `CTxMemPool` in `src/txmempool.{h,cpp}`; `mempool` global | Add a sibling `stempool` (or a tagged subset). |
| Version gating | `PROTOCOL_VERSION` `src/version.h:12` (170011) | Bump and gate `dandeliontx` on the new version + a service/feature handshake bit. |

The repo already ships custom P2P messages (the bootstrap manifest exchange),
so adding a relay-only message type follows an established local pattern and
needs no new framework.

## 4. Protocol design

### 4.1 New message

- `dandeliontx` — identical serialized payload to a `tx` message (a
  `CTransaction`), but semantically "this is a stem-phase transaction; do not put
  it in the public mempool or announce it by `INV` yet."

Only ever sent to a peer that has advertised Dandelion++ support. If a stem
destination is not Dandelion-capable, we **fluff immediately** to that peer (send
ordinary `tx`/`INV`), which is the graceful-degradation path.

### 4.2 Capability negotiation

Advertise support at handshake. Two compatible options:
1. A service bit (preferred if we have a spare `NODE_*` bit) set in `version`.
2. A post-`verack` feature ping (a no-payload `dandelion` message) so peers that
   understand it record `pfrom->fSupportsDandelion = true`.

Gate everything on **both** `nVersion >= DANDELION_PROTO_VERSION` **and** the
advertised flag, so old peers and unaware peers are simply treated as fluff
endpoints.

### 4.3 Epochs and stem routing

- An **epoch** is ~10 minutes (BIP-156 uses an aligned, randomized epoch so that
  not all nodes rotate at once). Track `nDandelionEpoch` and reshuffle on change.
- At each epoch, pick up to `DANDELION_MAX_DESTINATIONS = 2` outbound peers
  uniformly at random as this node's **stem destinations**.
- **Per-inbound-edge mapping (critical):** maintain a deterministic map
  `inbound peer → one of our stem destinations`, fixed for the epoch. Every stem
  transaction arriving on a given inbound edge forwards to the *same* destination.
  This is what defeats the graph-learning / intersection attack; do **not** pick a
  fresh random destination per transaction.
- Locally originated transactions are treated as if they arrived on a virtual
  "self" edge and are likewise mapped to one fixed destination per epoch.

### 4.4 Stem vs fluff decision

On receiving (or originating) a stem transaction, after it passes
`AcceptToMemoryPool` into the **stempool**:

- With probability `q` (BIP-156: fluff probability `q = 0.1` ⇒ expected stem
  length ~10 hops, but each node decides independently per its own state),
  **fluff**: move it from stempool to the public path — relay by ordinary
  diffusion (`RelayTransaction` semantics) and let it enter the announced mempool.
- Otherwise **relay along the stem**: send `dandeliontx` to the mapped stem
  destination for the edge it arrived on. If that destination is unreachable or
  non-supporting, fluff instead.

Implementation detail: BIP-156 fixes each node's stem/fluff *role* per epoch
(a node is either a relayer or a diffuser for the epoch) using a pseudo-random
function seeded by epoch + node identity, rather than flipping a coin per tx.
Adopt the per-epoch role to match the spec and to keep behaviour stable within an
epoch; the coin-flip framing above is the intuition.

### 4.5 Embargo timer (fail-safe)

Stem transactions must never get stuck if an intermediate node goes offline or
maliciously black-holes them.

- When a node accepts a stem transaction it sets a random **embargo timer**
  (BIP-156 suggests an exponential/uniform draw on the order of tens of seconds;
  start with mean ≈ 30s, randomized per tx).
- If the transaction has **not** been seen fluffed (i.e. has not appeared in the
  public mempool via ordinary diffusion) by the time its embargo expires, the
  node **fluffs it itself**.
- This guarantees liveness: worst case, the transaction diffuses after one
  embargo interval. It is also the backward-compat backstop — if a stem
  transaction is forwarded to a peer that silently does not understand it, the
  originator's embargo fires and the tx diffuses normally.

### 4.6 Pools and information flow

- **`mempool`** (existing): public, announced via `INV`, served on `getdata`.
- **`stempool`** (new): stem-phase transactions. Validated by the same ATMP
  checks, but **not** announced by `INV` and **not** served to peers via ordinary
  `getdata` for `MSG_TX`.
- Information is one-directional: a transaction can move `stempool → mempool`
  when it fluffs; nothing flows back. To keep validation consistent (so a stem
  child can be checked against a stem parent), the stempool view should see
  `mempool ∪ stempool`, mirroring BIP-156's "mempool flows into stempool."
- Memory bound the stempool and expire entries on their embargo deadline.

### 4.7 Message-handling rules (summary)

For a Dandelion-capable inbound peer, per BIP-156:
- Receive `dandeliontx` → validate into stempool → stem-relay or fluff per §4.4,
  arm embargo per §4.5.
- We do **not** announce stempool contents via `INV`, and we do **not** answer
  `getdata MSG_TX` from the stempool (stem txs are push-only along the stem).
- On fluff, the transaction follows the *existing* INV/getdata/tx path unchanged.

## 5. Parameters

| Constant | Value | Meaning |
|---|---|---|
| `DANDELION_MAX_DESTINATIONS` | 2 | Stem destinations chosen per epoch. |
| `DANDELION_EPOCH_SECONDS` | ~600 | Mean epoch length before reshuffle. |
| `DANDELION_FLUFF_PROBABILITY q` | 0.1 | Per-hop fluff probability (≈10-hop stems). |
| `DANDELION_EMBARGO_MEAN` | ~30 s | Mean fail-safe embargo before self-fluff. |
| `DANDELION_EMBARGO_JITTER` | randomized | Per-tx randomization of the embargo. |
| `DANDELION_PROTO_VERSION` | new ≥ 170012 | Min peer version for `dandeliontx`. |
| `-dandelion` (config) | `0` (default off) | Master enable; `1` to opt in. |

All of these are tunables for testing; ship with conservative BIP-156 defaults.

## 6. Phased implementation

**P0 — Plumbing, no behaviour change.** Add `DANDELIONTX` to `NetMsgType`
(`src/protocol.{h,cpp}`), the `DANDELION_PROTO_VERSION` constant and capability
flag/handshake, and `CNode` fields (`fSupportsDandelion`, stem-destination map).
Bump `PROTOCOL_VERSION`. No relay path changes yet; verify handshake negotiation
on a regtest swarm.

**P1 — Stempool.** Introduce the stempool (a second `CTxMemPool` instance or a
tagged overlay) and route stem-accepted txs through the *same* `AcceptToMemoryPool`
validation. Add embargo bookkeeping and expiry. No stem *forwarding* yet — every
accepted stem tx fluffs immediately (functionally identical to today), proving the
stempool/embargo machinery in isolation.

**P2 — Stem routing.** Implement epochs, per-edge destination mapping, the
per-epoch stem/fluff role, and `dandeliontx` send/receive. Wire local originations
(wallet `CommitTransaction` / `RelayTransaction`) into the stem. Drive embargo
timers and stem forwarding from the `SendMessages` loop. Graceful fallback to
fluff when the destination is non-capable/unreachable.

**P3 — Hardening & rollout.** DoS limits on stempool size and per-peer stem rate;
metrics (`getnetworkinfo`/a `getdandelioninfo` RPC reporting epoch, destinations,
stempool size, embargo counts); fuzz/unit tests; soak on testnet. Flip
`-dandelion` default to on only after testnet evidence that liveness (embargo
fallback) and propagation latency are acceptable.

## 7. Testing

- **Unit:** epoch reshuffle determinism, per-edge mapping stability, embargo
  expiry → self-fluff, fluff probability distribution, capability gating.
- **Regtest swarm:** multi-node line/star topologies; assert a stem tx visits N
  hops as `dandeliontx` before any node's public mempool shows it; kill an
  intermediate node and assert embargo fallback diffuses the tx.
- **Backward-compat:** mixed swarm of Dandelion-capable and legacy nodes; assert
  no tx is ever lost and that legacy peers only ever see ordinary `tx`/`INV`.
- **Adversarial sim:** a passive "spy" node connected widely; measure the
  first-spy origin-guess accuracy with and without Dandelion to confirm the
  expected flattening.

## 8. Risks and limitations

- **Latency:** stem hops add propagation delay (bounded by embargo). Acceptable
  for privacy-sensitive sends; measure on testnet before defaulting on.
- **Stem black-holing / DoS:** mitigated by embargo fallback and stempool size
  caps; a malicious destination can at worst delay a tx by one embargo interval.
- **Partial deployment:** privacy benefit scales with the fraction of
  Dandelion-capable peers; early on the stem frequently degrades to fluff. This
  is safe (never worse than today) but means the privacy gain ramps with adoption.
- **Not transport anonymity:** an adversary who already deanonymizes Tor, or who
  observes the originating link directly, is not stopped by Dandelion++. Pair it
  with Tor and chaff-send; advertise it honestly as one layer, not a guarantee.
