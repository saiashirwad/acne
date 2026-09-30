# Prior art: mechanisms for an agent-first block workspace

Research for [ticket #5](https://github.com/saiashirwad/acne/issues/5), within [map #1](https://github.com/saiashirwad/acne/issues/1). Sources accessed 2026-09-30.

## Answer

Borrow **Acme's addressable editing surface**, **the plumber's inspectable routing**, **Plan 9's contextual resource names**, **Patchwork's author-aware edit history**, **Potluck/Embark's gradual enrichment**, **marimo's explicit dependency graph**, and **Logseq's references and editable transclusion**. Make these different interfaces to the same blocks, not separate app silos.

Do not copy arbitrary-text shell execution, assume dependency injection is a security boundary, rerun agents automatically whenever text changes, or confuse CRDT convergence with undo semantics or exactly-once side effects. Keep the map's standing choices: agents edit directly; cmd-click dispatches registered commands or an intent; opt-click plumbs; project subtrees isolate agent context. These are acne requirements, not claims about the prior art.

The recommendations below are **design hypotheses**, not prototype-validated decisions. Each numbered item separates the documented mechanism from its proposed mapping. This research does not select the CRDT, final medium, agent protocol transport, or Effect API.

## 1. Acme: make editing programmable, not merely visible

Primary sources: [acme(1), interaction and tags][acme1]; [acme(4), control files][acme4] (Plan 9 from User Space documentation).

1. **Per-window addressable resources.** A window exposes `body`, `tag`, `addr`, `data`, `ctl`, and `event`. Writing a textual address to `addr` sets the range used by `data`; writing `data` replaces that range. The address is independent of the user's selection. `ctl` exposes operations such as showing the selection, renaming, and grouping undo with `nomark`/`mark`. **Borrow:** a uniform block API for read, anchored range edit, reveal, and action grouping. An agent should replace a cited span without stealing the human's cursor. **Avoid:** copying mutable numeric offsets across asynchronous reads and writes; acne needs CRDT-aware anchors or version/precondition checks. Acme's rune offsets are not such a concurrency mechanism. [acme4]
2. **Events as an extension interface.** Opening `event` reports edits and transfers interpretation of button-2/3 actions to the reader; certain actions can be written back for normal handling. Event origins distinguish keyboard, mouse, and file-interface actions—not persistent human/agent authorship. **Borrow:** a scoped action stream and an explicit “handled/pass” contract so agents can implement block tools. **Avoid:** treating this stream as a durable operation log or authorship ledger, or letting a stalled extension swallow user actions indefinitely. [acme4]
3. **Tags as editable command surfaces.** Tags combine a window name, maintained commands, and free text after `|`; commands can also be executed in the body. Unrecognized commands run through a shell. **Borrow:** small contextual command strips and editable command examples alongside content; use the project's registered-command/intent distinction. **Avoid:** Acme's shell fallback for arbitrary text. Also do not copy its Mac modifier mapping by accident: plan9port documents Option as execute and Command as locate, the reverse of acne's chosen cmd/opt roles. [acme1]

## 2. Plumber: routing should be inspectable data

Primary sources: [plumb(7), message and rule syntax][plumb7]; [plumber(4), ports and runtime rules][plumber4].

1. **A context-bearing envelope.** Messages contain source, destination, working directory, type, attributes, and data. A `click` attribute lets matching expand the clicked text into the intended target. **Borrow:** opt-click sends selected text plus project, source block/span, and optional target type to the router. Preserve context instead of asking an agent to reconstruct it from a bare string. [plumb7]
2. **Ordered match/rewrite/action rules.** Rule sets run in order; the first successful set wins. Predicates include `is`, `matches`, and `isfile`; rules can rewrite data and attributes. Rewrites are permanent, so the manual advises placing them after predicates. Rules can be read and changed through `plumb/rules`. **Borrow:** ordered rule blocks with a match preview, test examples, and a visible winning-rule trace. Agents/LLMs may draft rules for unmatched input; a draft is not automatically executable routing policy. **Avoid:** mutation during failed matching and unexplained shadowing; evaluate candidate rules without side effects, then apply the winning transformation. [plumb7][plumber4]
3. **Named output ports decouple tools.** Each client reading a port receives a copy; a `client` rule launches a consumer and retains the message, while `start` launches a command and discards it. **Borrow:** logical destinations such as `open`, `inspect`, or `agent.intent`, independent of the installed handler. **Avoid:** blindly broadcasting an intent to multiple agents that each perform the same write. Separate broadcast observations from single-consumer jobs; make delivery failures visible instead of losing work. [plumber4]

## 3. 9P and namespaces: uniform access, contextual bindings

Primary sources: [9P intro(5)][9p]; [The Use of Name Spaces in Plan 9][names].

1. **One small access vocabulary across resource types.** 9P serves hierarchical resources, including synthesized ones, through navigation/read/write operations. A client handle (`fid`) differs from the server's resource identity (`qid`); request tags correlate concurrent requests and replies. **Borrow:** stable block identity separate from a display path or session handle, plus a small shared protocol for UI and agents. An attention view, task log, and ordinary note can all be addressable. **Avoid:** requiring a 9P server/FUSE mount merely to reproduce this property on one Mac; a typed local protocol can test the idea first. [9p]
2. **Per-process resource views.** `bind` and `mount` compose resources at conventional names; a process can share or copy its namespace. **Borrow:** project-scoped service bindings so a generic agent resolves “project root,” “rules,” and “workspace reader” in its assigned context. Effect layers are the map's proposed implementation analogy. **Avoid:** mistaking that analogy for enforced isolation: authorization must be checked on reads, writes, queries, reference traversal, and external tool access, not only by hiding names in a prompt. Namespace composition and 9P permission checks are distinct documented mechanisms. [names][9p]
3. **Substitution for testing.** The namespace paper demonstrates binding an older library into the usual location and wrapping I/O to observe a process. **Borrow:** replace an agent's external services with recorded/fake services while keeping the same block-facing interface. Use this for safe plumbing and computation prototypes. **Avoid:** assuming everything must be a file; the paper explicitly excludes some operations, including process creation, from its file model. [names]

## 4. Patchwork: review direct edits, preserve agency

Primary sources: [2024 version-control notebook][patchwork]; [2026 task framework implementation][tasks]. These are research reports, not a stable product/API specification.

1. **Dynamic history and retrospective edit groups.** Patchwork saves fine-grained changes, offers author/time groupings and milestones, and explores attaching rationale and reverting edits after contributors edit directly. **Borrow:** each agent run has an identity, a navigable edit group, and optional rationale; the human can inspect/revert without forcing every edit into suggestion mode. **Avoid:** equating a displayed group with proven per-author undo under interleaved edits—the notebook itself questions when atomic groups are useful. [patchwork, entries 03–05]
2. **Bots and pointers as workspace data.** Bot prompts are editable/versioned documents; bots appear as authors in history. A separate experiment lets applications define pointers to text spans, shapes, or cells for shared comments, focus, and diff highlighting. **Borrow:** agent instructions as ordinary blocks, and one block/span pointer used by attention items, rationale, comments, and edit targeting. **Avoid:** requiring Patchwork's bot-branch workflow for every agent edit; it conflicts with acne's direct-edit choice. Borrow reviewability, not mandatory branching. [patchwork, entries 07, 11]
3. **Durable tasks, separate runs.** The 2026 framework stores task code/input and run history (logs, success/failure, timings) in Automerge documents. Its router deliberately permits duplicate execution during failover/partitions and relies on idempotent tasks. **Borrow:** a task block survives short-lived agent sessions, with separate attempt records. **Avoid:** importing distributed worker election for the one-Mac prototype or claiming that CRDT task storage guarantees exactly-once execution. External side effects need their own deduplication/confirmation policy. [tasks]

## 5. Potluck: text can become a tool incrementally

Primary source: [Potluck: Dynamic documents as personal software][potluck].

1. **Extract → compute → annotate.** Reusable live searches extract structured values from free text; small JavaScript expressions compute derived values. **Borrow:** agents can add a detector/formula block to existing notes rather than migrating the notes into a rigid application. Show matched spans and parsed values before trusting a detector. **Avoid:** presuming arbitrary prose parses reliably; the authors report that personal micro-syntax with immediate feedback works better than preexisting messy text. [potluck, “Challenges of parsing”]
2. **Derived overlays are not source edits.** Annotations remain separate from underlying text and are not themselves searchable; explicit buttons can edit text and participate in undo. **Borrow:** visibly distinguish computed output from authored spans, and make “write result back” an explicit action. This prevents a computed annotation from silently becoming a new agent input that re-triggers itself. [potluck, “Adding annotations,” “Editing the text programatically”]
3. **Composable, scoped searches.** Searches can be reused and restricted to regions; the authors found globally propagated search changes easy to break other documents with. **Borrow:** project-scoped rule/formula instances with visible dependencies and intentional upgrades. **Avoid:** an agent modifying one shared rule and unexpectedly changing every project. [potluck, “Tool composition”]

## 6. Embark: the outline is shared context and durable reasoning

Primary source: [Embark: Dynamic documents for making plans][embark].

1. **Mentions are links to structured outline nodes.** Imported entities become inspectable nodes; subtree views read the same data, and formula results expose structured properties for downstream computations. **Borrow:** references to blocks, not opaque embeds/screenshots; agents and humans can inspect the inputs underlying a view or answer. **Avoid:** forcing all structure through text extraction. Embark explicitly moved beyond Potluck's parsing burden to a structured outline. [embark, “Conceptual model,” “Mentions,” “Composing formulas”]
2. **Suggestions become explicit bindings.** Formula-suggestion/repeat heuristics run once; inserted arguments reference specific nodes rather than continually reinterpreting document shape. **Borrow:** agents propose a computation from surrounding context, then record the actual block IDs used. **Avoid:** silently changing dependencies when the user reorders an outline. The tradeoff is real: fixed arguments may become stale relative to their new context, so show them. [embark, “Auto-repeating formulas”]
3. **Data, computations, and views are separable.** A shared outline records reasoning while multiple views combine results. **Borrow:** attention as a view/query over blocks, not an independent inbox database; allow notes around computations. **Avoid:** assuming the prototype solved extension and schema interoperability: functions/views were hardcoded, schemas implicit, and collaboration had prototype-quality issues. [embark, “Documents capture thought process and context,” “Limitations and future work”]

## 7. Malleable software and local-first: properties, not a feature shopping list

Primary sources: [Malleable software (2025)][malleable]; [Local-first software (2019)][localfirst].

1. **An in-place path from use to modification.** The malleability essay advocates a gentle slope: use a tool, tweak it, compose it, then program when needed. It explicitly argues that AI-generated isolated apps do not solve composition. **Borrow:** inspect/edit/copy a rule or computation where it is used; agents help modify these artifacts rather than producing a new app silo. **Avoid:** an LLM as the only way to make even a precise one-field change. [malleable, “AI-assisted coding,” “A gentle slope from user to creator”]
2. **Shared data, interchangeable tools.** Malleability needs both data sharing and UI composition. **Borrow:** human outline editors, agent tools, queries, and alternate views operate on one block model. **Avoid:** a separate agent-memory store that becomes the hidden authoritative workspace; this reinforces the map's “workspace is memory” decision. [malleable, “Sharing data between tools”]
3. **Local authoritative data and optional networking.** The local-first essay makes local copies primary and emphasizes offline access, longevity, collaboration, and ownership; it discusses CRDTs as enabling technology. **Borrow:** local durable persistence, backup/export, and editing that survives agent/network outages. **Avoid:** claiming “uses a CRDT” proves these properties, or that a cloud-model call works offline. Export must preserve block IDs/references and provenance where possible; exported plain prose alone is not a lossless backup. This last requirement is an acne recommendation, not an essay guarantee. [localfirst, “Seven ideals”]

## 8. marimo: a graph for computation, not automatic agency

Primary source: [marimo reactivity guide][marimo].

1. **Dependencies, not visual order, determine execution.** Static analysis finds definitions/references and builds a DAG; running a cell schedules its descendants. **Borrow:** computation blocks declare/reference input block IDs, and moving blocks does not change dependency semantics. Display the graph and diagnose cycles. **Avoid:** applying Python name analysis directly to arbitrary TypeScript, prose, or LLM calls; explicit dependencies are the smallest testable starting point. [marimo, “How marimo runs cells”]
2. **No ambiguous owners or hidden residual variables.** A global name has one defining cell; deleting a cell removes its globals. Mutations and attribute assignments are not tracked. **Borrow:** one producer per output, invalidation on deletion, and immutable/versioned inputs to computations. **Avoid:** an agent mutating a shared object out of band while the scheduler believes dependencies are unchanged. [marimo, “Deleting a cell,” “Variable mutations,” “Global variable names”]
3. **Stale/disabled states and interruption.** Lazy mode marks descendants stale rather than immediately running; disabling blocks a cell and its dependents; interrupts cancel queued dependents. **Borrow:** automatic rerun for cheap pure computations; stale + explicit run/cancel for expensive agent calls or side effects. **Avoid:** spending money, editing repos, or recursively invoking agents on every keystroke. This execution policy is an acne recommendation, not a claim that marimo enforces an effect system. [marimo]

## 9. Logseq: references, transclusion, and query views

Primary sources: official Logseq docs repository pages on [block references][refs], [embeds][embeds], [simple queries][queries], and [advanced queries][advanced]. Scope: these documented mechanisms, not a compatibility promise across Logseq's file-based and newer database editions.

1. **Addressed references instead of copies.** `((block-address))` displays a referenced block's content, and a reference counter exposes backlinks. **Borrow:** stable IDs and backlinks so agent citations, instructions, and attention entries point to one source. **Avoid:** copy/pasting “memory summaries” as silently competing authoritative records. [refs]
2. **Editable transclusion has different semantics from a reference.** Block references show one block; embeds include its descendants and allow editing the source. **Borrow:** distinguish link/reference, editable embed, and snapshot visually and in the agent protocol. An agent editing a transclusion edits the original, not a copy. **Avoid:** granting access through an embed to a project the agent cannot otherwise read/write; authorize the resolved source and guard rendering cycles. [embeds]
3. **Queries are document content.** Simple query blocks combine predicates; advanced Datalog queries expose inputs, transforms, and views, including context such as current/parent block. **Borrow:** saved scoped query blocks for “needs human,” “agent changed,” and “stale computations.” **Avoid:** requiring Datalog for initial use or using result visibility as authorization. The query engine must enforce project scope independently. [queries][advanced]

## Prototype implications and unresolved evidence

Prioritize these small falsifiable experiments (not additional standing decisions):

| Experiment | Pass condition | Main source lineage |
| --- | --- | --- |
| Human and agent edit one block while it is transcluded elsewhere | Stable targeting, no stolen cursor, visible authorship; undoing the agent preserves intervening human edits | Acme + Logseq + Patchwork |
| Opt-click a path, block reference, and unmatched phrase | Show envelope and winning rule; unmatched phrase yields a draft rule, not silent activation; a duplicate consumer cannot duplicate a job | Plumber |
| Two project-scoped agents follow references and queries | Both API and UI deny cross-project access unless explicitly granted; test external tools too | Namespaces |
| Change an input to a pure formula and an expensive agent task | Formula reruns; task becomes stale until explicitly run; cancelled/old results cannot overwrite newer results | marimo + Potluck |
| Reorder a plan containing formulas and an attention query | Stable inputs remain inspectable; derived output stays distinct from authored text; query returns original block identities | Embark + Logseq |
| Restart without network/model service | Notes, provenance, rules, and past task results remain readable/editable; unavailable computations report that state | Local-first + Patchwork |

Not confirmed by these sources: a ready-made per-author undo algorithm matching acne's concurrent edits; an authenticated span-authorship model; a universal pointer representation that survives every structural edit; exactly-once external agent actions; a production-secure in-process Effect-layer sandbox; or the scale/performance limits for this combined system. Historical prototype findings motivate tests, not guarantees. In particular, Acme undo grouping is not author-selective undo, and Patchwork's author-aware history is not proof that a chosen CRDT exposes all required operations today.

## Primary-source index

[acme1]: https://9fans.github.io/plan9port/man/man1/acme.html
[acme4]: https://9fans.github.io/plan9port/man/man4/acme.html
[plumb7]: https://9fans.github.io/plan9port/man/man7/plumb.html
[plumber4]: https://9fans.github.io/plan9port/man/man4/plumber.html
[9p]: https://9p.io/magic/man2html/5/intro
[names]: https://9p.io/sys/doc/names.html
[patchwork]: https://www.inkandswitch.com/patchwork/notebook/2024-version-control/
[tasks]: https://www.inkandswitch.com/patchwork/notebook/tasks-02/
[potluck]: https://www.inkandswitch.com/potluck/
[embark]: https://www.inkandswitch.com/embark/
[malleable]: https://www.inkandswitch.com/essay/malleable-software/
[localfirst]: https://www.inkandswitch.com/essay/local-first/
[marimo]: https://docs.marimo.io/guides/reactivity/
[refs]: https://github.com/logseq/docs/blob/master/pages/The%20basics%20of%20block%20references.md
[embeds]: https://github.com/logseq/docs/blob/master/pages/The%20difference%20between%20block%20embeds%20and%20block%20references.md
[queries]: https://github.com/logseq/docs/blob/master/pages/Queries.md
[advanced]: https://github.com/logseq/docs/blob/master/pages/Advanced%20Queries.md

- Plan 9 from User Space: [Acme interaction][acme1], [Acme API][acme4], [plumber service][plumber4], [plumbing language][plumb7].
- Plan 9: [9P specification][9p], [namespace paper][names]. The historical namespace paper describes an earlier protocol revision; use the manual, not its old message count, for 9P details.
- Ink & Switch: [Patchwork version-control notebook][patchwork], [task framework][tasks], [Potluck][potluck], [Embark][embark], [Malleable software][malleable], [Local-first software][localfirst].
- [marimo reactivity documentation][marimo].
- Logseq official docs: [references][refs], [embeds][embeds], [simple queries][queries], [advanced queries][advanced]. These living sources may change; recommendations are based on their content at the access date above.
