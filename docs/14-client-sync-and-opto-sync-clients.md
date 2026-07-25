# 14. Client-side sync: patterns and the opto-sync-clients adoption

**Issue:** zed's offline-first story (optimistic writes in the browser → a
durable local store → sync to Postgres/Supabase) is exactly the problem a whole
generation of sync engines already solved. Rather than grow yet another bespoke
engine in `zed-sync`, the direction is to consume
[`github.com/opto-sync/opto-sync-clients`](https://github.com/opto-sync/opto-sync-clients),
which wraps the C merge core
[`syncer.c`](https://github.com/opto-sync/syncer.c). This doc records the
best-practice patterns distilled from the field, confirms the opto-sync client
uses `syncer.c` correctly, and lists exactly what zed-sync still has to supply.

## What the field does (distilled)

Surveying Linear, Replicache/Zero, ElectricSQL, PowerSync, RxDB, WatermelonDB,
and the CRDT engines (Automerge/Yjs), the patterns that consistently matter for
a JSONB-merge (not full-CRDT) client:

1. **Two-tier optimistic write** — apply to the in-memory/reactive view
   instantly, and persist to a **durable local queue before any network I/O**.
   The pending write must never live only in a JS variable.
2. **Durable queue that survives restart**, with a `pending/synced/failed`
   lifecycle; the flush loop re-reads payloads from the store (crash-safe), not
   from memory.
3. **Exactly-once echo dedup** — every mutation carries a stable `clientId` +
   monotonic per-client `mutationId`; the server keeps a `lastMutationID`
   high-water mark and ignores anything `≤` it; the client drops confirmed
   pending on pull. (Replicache's model; Linear's lack of cross-restart dedup is
   the cautionary tale.)
4. **Rebase-on-pull** — server state becomes the base, then un-confirmed local
   mutations replay on top, oldest-first.
5. **Don't timestamp-gate the overlay of your own pending writes** during rebase,
   or a pending edit older than the server's `updatedAt` vanishes then flickers
   back.
6. **Field-level merge, never whole-doc/whole-array replace** — send changed
   fields; resolve per-key/per-element so disjoint concurrent edits both survive.
7. **Logical clock (HLC), not wall-clock, for LWW** — persisted, node-tagged,
   observes remote stamps; one timestamp format per key; sub-ms as digit strings
   to dodge float precision.
8. **Explicit deletes** — an additive/LWW merge can't remove a key; use a
   tombstone field (LWW-gated) or a delete op, and have the server retain death
   certificates.
9. **Idempotent replay** — re-sending a payload is a semantic no-op
   (identity-keyed array merge, not append).
10. **Server stays authoritative and may reject/transform** — surface rejection
    to the app; roll back or rebase the optimistic value (don't silently revert).
11. **Store choice** — IndexedDB for KV/document workloads (Linear, Replicache,
    RxDB-web, Yjs); SQLite / wasm-SQLite for relational or large on-device sets
    (PowerSync, WatermelonDB-native).

## opto-sync-clients uses syncer.c correctly

The C core is a pure merge function; the queue, clock, identity, rebase, and
dedup live in the clients. The **TypeScript client** (`clients/ts`) is the
complete one and its `syncer.c` usage is correct:

- Reconcile options are the core's intended CRDT policy and a pinned cross-tier
  contract: `arrayStrategy: MERGE_BY_KEY`, `arrayMatchKeys: 'id'`,
  `resolveByTimestamp: true`, `lwwKeys: 'updatedAt,syncedAt'`,
  `fwwKeys: 'createdAt'`. (`MERGE_BY_KEY` + `resolveByTimestamp:true` are
  required — the core's own default `REPLACE` would drop local-only array
  elements and skip element-level timestamp gating.)
- Correct LWW direction (base = local, incoming = server; incoming wins unless
  local is newer).
- Honors the binding contract: `undefined` → core default vs `''` → "no keys";
  the extended **path-based** override callback (not key-based); NULL only on
  invalid JSON, surfaced as a throw (never a silent `""`).
- It already ships the optimistic layer the checklist demands: a durable
  Dexie/**IndexedDB** queue, an HLC, `(clientId, mutationId)` echo dedup, and
  `rebasePending`/`localView`/`confirmSyncedUpTo`. Browser loads a real **wasm**
  build of `syncer.c`; Node loads the N-API addon; there is **no JS-fallback
  merge** (a merge that silently no-ops would lose writes). The **Dart** client
  uses Drift/**SQLite** but is materially thinner (no dedup/HLC/rebase yet).

## What zed-sync must supply on top

Consuming the TS package gives correct `syncer.c` usage and the full optimistic
layer out of the box. zed-sync still owns:

- **The transport/push loop** — `triggerBackgroundSync()` is a deliberate stub.
  Implement: `pendingMutations()` (in order) → POST each with
  `(clientId, mutationId)` → `confirmSyncedUpTo(serverWatermark)` →
  `recordPushFailure` on transient errors.
- **A delete/tombstone convention** — the merge cannot remove keys. Use an
  LWW-gated `deletedAt`/`_deleted` field merged as ordinary data; the server
  retains death certificates so other clients learn of deletes.
- **Per-element timestamps** — records inside a `MERGE_BY_KEY` array each need
  their own `updatedAt` for element-level LWW; the client auto-stamps only the
  top-level `updatedAt`.
- **Terminal-row compaction** — prune `synced`/`failed` queue rows so the local
  table doesn't grow unbounded.
- **Dart/Rust parity** — if zed-sync targets those, build the rebase +
  `(clientId, mutationId)` dedup + HLC-into-queue that currently exist only in TS.

The current `zed-sync/sdk` (bespoke `core`/`client`/`merge`/`hlc`) stays the
reference until this migration lands; its conformance fixture and the
`syncer.c` semantics agree on the merge model, which is what makes the swap
low-risk.

## Status: direction set

`opto-sync-clients` is the target; its TS client is verified-correct against
`syncer.c`. The integration items above are the remaining work for zed-sync;
none require changes to `opto-sync-clients` itself (except optionally bringing
the Dart client to TS parity).
