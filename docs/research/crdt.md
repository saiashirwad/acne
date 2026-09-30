# CRDT choice for acne

Research for [ticket #2](https://github.com/saiashirwad/acne/issues/2), in the context of [map #1](https://github.com/saiashirwad/acne/issues/1). Investigated 2026-09-30. This is a library-selection decision, not a claim that the complete workspace or its scale has been validated.

## Recommendation

**Use Loro for the first workspace prototype.** Its native, ordered, movable tree directly matches acne's hardest structural requirement: stable blocks that can be reparented while other participants edit them. It also provides rich text, local selective undo and historical checkout. Automerge and Yjs are credible alternatives, but their documented data models do not supply the same native movable-tree abstraction; implementing a cycle-safe tree on their maps/lists would become acne's responsibility. [L1][L2][L3][L4][A1][A2][Y1]

**Do not equate local undo with the required durable per-author undo.** Loro's manager tracks one peer and clears its stacks if that peer ID changes. Human/agent identity must be a separate domain concept from the CRDT peer/session. Prototype author attribution and undo across agent-session boundaries before calling this a final production choice. No candidate examined establishes all of acne's requirements without application work. [L3][L5]

Decision gates: (1) correct concurrent move/edit/delete behavior, (2) attribution that survives interleaved edits and undo, (3) an explicit answer for undo after agent/process restart, and (4) a representative 100k-block benchmark. These are follow-up prototype work, not facts established by this research.

## Comparison

The comparison concerns the documented APIs reviewed here: Loro's current docs and tested `loro-crdt@1.16.3`, Automerge JS API v3.5.0, and Yjs's documented API plus v13.6.27 snapshot implementation. Absence means **not found in these public APIs**, not a claim that no experimental branch or third-party implementation exists.

| Need | Loro | Automerge | Yjs |
| --- | --- | --- | --- |
| Ordered block tree, stable identity, reparenting | Native `LoroTree`; node data maps; `move`, `moveBefore`, `moveAfter`. Uses a replicated-tree move algorithm and fractional sibling ordering. [L1] | Maps, lists, text; no native tree/move primitive in reviewed public API. Store blocks by ID and design parent/order conflict handling yourself. Text block markers are not a movable outline tree. [A1][A2][A3] | Maps/arrays/XML can represent structure, but `Y.Array` has insert/delete, not native reparenting; a shared type can only occur once. Use an ID-indexed block table and a separate structural model, not delete/reinsert of the same nested type. [Y1] |
| Rich text inside blocks | `LoroText` marks, deltas, stable cursors; official ProseMirror and CodeMirror bindings documented. [L2] | Peritext-based text, marks and inline block markers; `spans` exposes text/marks/block values. [A1][A3] | `Y.Text` formatting, delta import/export; UTF-16 offsets. [Y2] |
| Per-span author tint | Custom author mark, explicitly stamped on each insertion. Not automatic authenticated provenance. `applyDelta` lets the insertion specify its full attribute set. [L2] | Custom author marks are possible. Current API also maps actors to authors, but `Span` itself contains text/marks, not an automatic per-character author field. [A1][A3][A4] | Custom author formatting attribute on inserted text; omitted attributes inherit prior formatting, so explicitly supply authorship. [Y2] |
| Per-author undo | Built-in local selective undo; exactly one peer per manager, peer switch clears stack. An author-to-peer/session policy and durable action history remain application work. [L3][L5] | No built-in `UndoManager`/`undo` in reviewed core JS API. Plan an application selective-inverse layer or separately evaluate a maintained integration; historical views are not selective undo. [A2][A5] | Built-in selective `UndoManager`, scoped types and `trackedOrigins`. Particularly convenient for distinct local author origins in one host document. Not automatically a durable user-global undo stack. [Y3] |
| History / time travel | `checkout(frontiers)`, then `attach()` for latest state; full-history snapshots available. Trimming to shallow snapshots removes older history and limits sync. [L4][L6] | `getHeads`, change APIs, immutable `view(doc, heads)`; view shares memory and is cheap. A strong history-oriented alternative. [A2][A5] | Snapshots store state vector + delete set; `createDocFromSnapshot` requires source document GC disabled. Requires an intentional snapshot/history retention policy. [Y4] |
| TypeScript ergonomics | Explicit mutable container APIs; text deltas map well to an editor boundary. Requires handling WASM packaging and container-to-domain decoding. Sample below exercised through Bun. [L2][L7] | Typed document model and change callbacks over JS-like objects/arrays; easiest shape for ordinary domain objects, but collaborative strings require splice/update APIs. [A1][A2] | Direct mutable shared types and synchronous observers; straightforward transaction-origin integration. Types remain CRDT containers, not an automatically validated domain schema. [Y1][Y2][Y3] |
| ~100k blocks | **Unproven for acne.** Native moves avoid a custom structural layer, but 100k maps/text containers, history, and multiple author replicas must be measured. [L1][L7] | **Unproven for acne.** Do not reject current Automerge using results from old 2.x benchmarks. [A2][L7] | **Unproven for acne.** Include retained deleted content/history and tree-repair costs in memory/latency measurements. [Y4][L7] |
| Effect v4 fit | Wrap behind a service/layer; serialize mutation and commit; adapt events and persistence. No special Effect advantage established. | Same integration boundary; immutable document replacement is a natural service-owned state model. | Same boundary; transaction origins simplify local author routing. |

Effect row is an architecture recommendation, not a claim that any of these libraries ships a first-party Effect adapter. Effect v4 retains Effect/Layer/Schema/Stream and currently documents `Context.Service` rather than v3's `Context.Tag`; pin the chosen v4 version instead of copying v3 service examples. [E1]

## The important modeling distinctions

### 1. Blocks, placements and transclusion

**Proposed schema:** one Loro tree of outline placements; each placement points to a stable block ID; a root block map owns each block's text and metadata. Moving a placement changes only the tree. A transclusion creates another placement referencing the existing block, not a second parent on the same tree node. Delete-placement and delete-content must be different commands. This is a domain design recommendation built on the tree's one-parent semantics and per-node maps, not an automatic Loro feature. [L1]

The tiny sample uses text directly under a node to show the API with minimal noise; the actual transclusion schema should use the separate block table above. Never represent a move as copying all text into a new node: that complicates references and authorship unnecessarily.

Native moves do not settle every UX question. Test concurrent moves of the same node to different parents, concurrent A-under-B/B-under-A, reorder collisions, and moving a child while another peer deletes its ancestor. Loro documents subtree deletion and possible fractional-index interleaving; convergence alone does not tell us whether the visible result matches acne's desired behavior. [L1]

### 2. An author is not a peer ID or transaction origin

**Proposed author record:** stable `authorId`, `kind: human | agent | system`, display name, plus session IDs and a peer-to-author mapping. A short-lived agent session may belong to the same logical author as earlier sessions. Loro deliberately generates a new peer ID for each document instance and warns against using user IDs as peer IDs or sharing a reused peer concurrently. [L5]

**Proposed span rule:** attribution means “who inserted this surviving text,” not “who last formatted it.” Stamp every inserted run with its author in the same committed action. Styling must preserve author; replacement text gets the replacer's author. Define paste/copy behavior explicitly (recommended: new author plus optional source-provenance metadata). Do not infer identity from color or nearby marks.

Use `author: { expand: "none" }`, but **that alone is not enough**: it controls boundary expansion, not every insertion inside a marked region. Use insertion deltas with an explicit author and intended formatting attributes. Loro documents that `applyDelta` removes inherited attributes omitted from inserted deltas, whereas plain `insert` may inherit styles. Preserve intended bold/link attributes when constructing that delta. [L2]

Marks are editable replicated data, not proof of identity. A trusted command boundary must derive author identity from the calling session, prevent arbitrary author-relabeling, and validate allowed operations. A raw CRDT update submitted by an untrusted agent must not be treated as authenticated authorship. This is an application security requirement, not a property promised by any source here.

### 3. Session-local undo is only the beginning

For the prototype, use a **long-lived host-owned replica + UndoManager per active author lane**, and import other lanes' updates into it. Short-lived agent processes send commands through their lane rather than becoming the owner of its undo stack. Do not switch `doc.setPeerId` for every agent command: that clears Loro's manager. This design follows its documented single-peer limitation; its memory overhead must be measured before adopting it at 100k blocks. [L3][L5]

Group an agent edit into a semantic action, not thousands of token-level undo steps; Loro supports manual grouping and configurable merging. A human pressing “undo agent X” routes to that lane's manager and distributes the resulting CRDT changes. The manager selectively undoes its local edits in the presence of imported edits; this is not equivalent to replacing the workspace with an old snapshot. [L3]

**Remaining gap:** the reviewed docs do not establish a supported durable serialization/restoration mechanism for undo stacks. Host restart, multiple sessions/peers belonging to one author, and undoing an arbitrary old action rather than the top stack item need a deliberate design and tests. Record a durable semantic action journal (author, action ID, touched blocks, before/after versions, grouping); do not assume that a journal alone implements safe selective inverse operations. If durable per-author undo is mandatory in the first production release, this is a release blocker, not permission to silently downgrade the requirement.

Yjs is the strongest fallback if author-origin filtering in one live document dominates the prototype: separate managers can track separate origins. Those origins are supplied to local transactions; the reviewed API does not make them a replicated user identity or persistent cross-session history. [Y3] Automerge's historical view is useful for inspection, but reverting to a past state would erase unrelated later changes unless a proper selective operation is computed. [A5]

## Effect boundary and storage proposal

Keep raw CRDT handles private to a `WorkspaceStore` service. Expose commands such as `insertText`, `movePlacement`, `undoAuthor` and reads returning validated domain values. Provide project-scoped command services through layers, but check the project authorization on every operation: a layer is an API-scoping mechanism, not protection against a raw whole-workspace update.

Serialize mutation/commit per replica; do not suspend asynchronous Effect work in the middle of a mutable CRDT transaction. Validate before mutation, group edits synchronously, then persist/export. Adapt subscriptions to a scoped stream/queue and release listeners with the service scope. Map import/validation/storage errors into typed failures. Wrapping CPU-heavy WASM in an Effect does not move it off the event loop; use a worker if measurements demand one. These are integration recommendations using Effect's service/layer model, not a tested v4 adapter. [E1]

Start with one logical workspace document for cross-project references/moves, plus append-only updates and periodic full snapshots on the local server. Keep historical version identifiers for action boundaries; inspect history on a separate read replica rather than detaching the live editing document. Do not enable shallow-history trimming until retention/undo policy is defined: it trades away earlier history and limits which peers can sync. [L4][L6]

## Tiny executable API sample

Executed with Bun 1.4.0 and `loro-crdt@1.16.3` in an isolated temporary directory. The full check also used a second replica to append agent-authored `!`; after import, human undo left `!`, redo restored `Hello!` with the two distinct author attributes, and moving the block to root preserved its ID. This is a smoke test, **not** a concurrent-move, durability, Effect, or scale test.

```ts
import { LoroDoc, LoroText, UndoManager } from "loro-crdt";

const doc = new LoroDoc(); // random peer; author identity is separate
const author = "human:owner";
doc.configTextStyle({ author: { expand: "none" } });
const tree = doc.getTree("blocks");
const project = tree.createNode();
const block = project.createNode();
const text = block.data.setContainer("text", new LoroText());
doc.commit(); // schema setup is outside this undo manager

const undo = new UndoManager(doc, { mergeInterval: 0 });
text.applyDelta([{ insert: "Hello", attributes: { author } }]);
doc.commit();
undo.undo(); // empty text, not a whole-workspace rollback
undo.redo(); // "Hello", including attribution
block.move(); // same node becomes a root; no content copying
doc.commit();
console.log(text.toDelta());
// [{ insert: "Hello", attributes: { author: "human:owner" } }]
```

The methods and semantics used above are documented in the tree, text and undo guides. [L1][L2][L3] Install the exact package in a scratch directory with `bun add loro-crdt@1.16.3`, then run a file containing the sample with `bun run <file>.ts`.

## Performance validation required before final selection

The published Loro comparison uses historical versions (including Loro 1.0.0-beta.2, Automerge 2.1.10 and Yjs 13.6.15), predominantly text/map operations, and explicitly warns against treating it as a universal ranking. Its “100K” operations do **not** establish fitness for 100k rich-text outline blocks. No current, apples-to-apples acne workload was run in this research. [L7]

Proposed reproducible gate on the owner's Mac, using pinned versions and the actual web/server runtime:

- Generate 100k placements and blocks, e.g. 200 UTF-16 units per block (~20M units), shallow/wide and deep trees, mixed human/agent marks, realistic deletions and a long edit history. Separately vary block count, text size, author count and history length.
- Measure cold snapshot load, time to first visible subtree, RSS including WASM, persisted bytes, incremental update bytes, local text edit and subtree move p50/p95/p99, remote import, undo/redo, and checkout time. Test 1, 4 and 16 author lanes to expose replicated-state costs.
- Exercise offline conflicting moves/edits, repeated reorder into narrow sibling gaps, large paste, restart/reload, and retained versus trimmed history. Check converged structure and author deltas, not just elapsed time.
- Keep rendering virtualized and avoid whole-workspace `toJSON()` on each edit. Measure UI latency separately from CRDT latency. Start with proposed targets of <16 ms p95 local command processing, <2 s first usable view and <1 GB total process RSS; these are acceptance targets to validate, not measured results or user-approved requirements.

If the replica-per-author design fails the budget, investigate same-peer origin-excluded managers or a durable semantic undo service before sharding. Splitting into per-project documents may lower working sets but makes cross-project moves/history non-atomic; it is a separate design decision, not a free optimization.

## Alternatives and uncertainty

- **Choose Yjs instead** if its transaction-origin undo model and editor integration materially reduce product work and acne accepts owning the cycle-safe tree layer and history policy. [Y1][Y3][Y4]
- **Choose Automerge instead** if historical document views and JS-shaped data are more important than native outline moves, and acne budgets for structural conflict resolution and selective undo. Current actor/author mapping deserves evaluation rather than assuming it has no attribution support. [A1][A4][A5]
- **Do not build a bespoke CRDT yet.** The tree-move research already underpins Loro; reimplementing it adds a correctness obligation without prototype evidence that it is needed. [L1]
- Not confirmed: 100k-block performance for any candidate; durable author-wide undo; correctness of attribution for every editor binding, paste and concurrent formatting case; full conflicting-move UX; a production-ready Effect v4 adapter. The narrow smoke test establishes only that the selected basic APIs and one selective-undo scenario work.

## Primary sources

All URLs reviewed for this report; live documentation may evolve.

- **[L1]** Loro, [Tree](https://loro.dev/docs/tutorial/tree): native move algorithm, data maps, ordered siblings, deletion events and API examples; links to the original replicated-tree research.
- **[L2]** Loro, [Text](https://loro.dev/docs/tutorial/text): marks, expansion policy, insertion deltas, UTF-16 offsets and editor bindings.
- **[L3]** Loro, [Undo/Redo](https://loro.dev/docs/advanced/undo): local selective undo, single-peer limitation, origin exclusions and grouping.
- **[L4]** Loro, [Time Travel](https://loro.dev/docs/tutorial/time_travel): checkout, frontiers and reattachment.
- **[L5]** Loro, [PeerID Management](https://loro.dev/docs/concepts/peerid_management): session IDs, operation counters and unsafe ID reuse.
- **[L6]** Loro, [Shallow Snapshots](https://loro.dev/docs/concepts/shallow_snapshots): history truncation, full snapshots and sync limitations.
- **[L7]** Loro, [JS/WASM Benchmarks](https://loro.dev/docs/performance): test versions, methodology and limitations. Smoke test package: [loro-crdt 1.16.3](https://www.npmjs.com/package/loro-crdt/v/1.16.3).
- **[A1]** Automerge, [Document Data Model](https://automerge.org/docs/reference/documents/): types, marks and JS mapping.
- **[A2]** Automerge, [JS API v3.5.0](https://automerge.org/automerge/api-docs/js/): reviewed public operation surface.
- **[A3]** Automerge, [spans](https://automerge.org/automerge/api-docs/js/functions/spans.html) and [Span type](https://automerge.org/automerge/api-docs/js/types/Span.html): text, marks and inline block markers.
- **[A4]** Automerge, [getAuthorForActor](https://automerge.org/automerge/api-docs/js/functions/getAuthorForActor.html): actor-to-author API (implementation link pinned by the generated docs).
- **[A5]** Automerge, [view](https://automerge.org/automerge/api-docs/js/functions/view.html): immutable historical view, shared memory, required heads.
- **[Y1]** Yjs, [Y.Array](https://docs.yjs.dev/api/shared-types/y.array): nested-type constraints, operations and synchronous observers.
- **[Y2]** Yjs, [Y.Text](https://docs.yjs.dev/api/shared-types/y.text): rich-text attributes, inheritance and deltas.
- **[Y3]** Yjs, [Y.UndoManager](https://docs.yjs.dev/api/undo-manager): selective undo, scopes, tracked origins and grouping.
- **[Y4]** Yjs, [Snapshot.js at v13.6.27](https://github.com/yjs/yjs/blob/v13.6.27/src/utils/Snapshot.js): snapshot representation and GC-disabled restore precondition.
- **[E1]** Effect, [v4 migration guide](https://github.com/Effect-TS/effect/blob/main/MIGRATION.md): retained core model, version pinning and service API changes. [Former v4 repository notice](https://github.com/Effect-TS/effect-smol) confirms the canonical repository has moved to `Effect-TS/effect`.
