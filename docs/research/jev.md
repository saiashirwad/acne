# Jev: useful risk advice, not an authorization boundary

Research date: 2026-09-30. Resolves [ticket #4](https://github.com/saiashirwad/acne/issues/4), using the owner-first, single-Mac, Effect v4, project-scoped design in [map #1](https://github.com/saiashirwad/acne/issues/1).

## Recommendation

**Keep deterministic capability checks and human confirmation as the gate. Prototype Jev only as an optional, replaceable adviser for semantic risk/intent classification.** Do not let its confidence grant permissions or waive confirmation of arbitrary agent-written shell. This is an architectural recommendation, not a measured result: TypeSafe explicitly says adversarial state can steer Jev's answer.[4]

For acne, the useful distinction is:

- **“Can this agent perform this operation on this project?”** Check trusted identity, operation, resource, and grants in code. No model required.
- **“Does this proposed operation seem inconsistent with the user's intent?”** Jev can supply an additional warning/review signal.
- **“May this exact shell command run now?”** The trusted executor decides, with explicit approval where required and OS restrictions underneath. Neither an Effect layer nor a model answer is an OS sandbox.

## What it actually is

Jev is TypeSafe AI's first public **System One decision model**, not a permission library, shell parser, agent framework, or chat/code-writing model. It receives `state` plus named questions and returns bounded, typed answers.[1] The primary integration is `POST https://api.typesafe.ai/v1/systemone`; the official JavaScript repository is [typesafe-ai/typesafe-sdk-js](https://github.com/typesafe-ai/typesafe-sdk-js), published as `@typesafe-ai/sdk`.[2][3]

| Primitive | Output | Potential acne use |
| --- | --- | --- |
| Choice | Selected label, per-label probabilities, confidence | Classify a proposal into a fixed review category, including `unclear` |
| Score | Expected score (possibly fractional), rubric-level probabilities, confidence | Prioritize pending attention items |
| Noul | Probability from 0 to 1 that a yes/no statement is true; **no separate confidence field** | Flag whether an action appears outside the stated user intent |

These are the actual SDK response shapes.[3] Choice/Score confidence is derived from the probability distribution; it is not independent corroboration or a probability that execution is safe. Thresholds require domain-specific evaluation.[5] Jev does not generate an explanation, so the UI should distinguish policy-generated reasons from model scores rather than fabricate a model rationale.[1]

## Maturity and constraints

- **Early access, not a proven security gate.** TypeSafe's homepage calls Jev early access. The JS repository was created on September 4, 2026; inspected source is SDK 0.6.0, MIT, Node >=20, with test/build scripts. This is available integration tooling, not evidence of security correctness or long production history.[2][6]
- **Current model:** official docs list `jev-1.13.0`; `jev-latest` and `jev-preview` both currently point there. Pin the version for an experiment and log the returned model: aliases can move.[7]
- **Hosted dependency:** the official path is an authenticated remote API. The reviewed first-party sources did not establish a downloadable model/weights or supported local inference route. Do not confuse an MIT SDK with an open-weight model.[2][3][7]
- **Published limits/cost:** $0.042 per million input tokens, output free; 40 requests/sec and 100K tokens/sec; 64K total request context and 32K for state plus longest question. Rate limits can change without notice. These are vendor documentation figures, not acne benchmarks.[7]
- **Documented weaknesses directly matter here:** adversarial instructions/framing in state can change the answer; indirection and irrelevant context reduce reliability; arithmetic/date comparisons are unreliable; logically related questions need not obey expected identities.[4] Obfuscated shell and attacker-controlled project text are therefore particularly poor foundations for automatic authorization.
- **Privacy:** docs say requests/responses are not used for training and enterprise ZDR is available. That is not a statement that every account has zero retention. Review the applicable agreement before sending private workspace text; minimize/redact state and keep API credentials server-side.[7][8]

A notable SDK caveat: the inspected client parses JSON and casts it to its TypeScript result type; it does not runtime-validate the response against those types. Its debug logging includes bodies. Defaults are a 10-second per-attempt timeout and two retries, without a total retry budget.[3] A gate must add runtime decoding, bounded latency, safe logging, and fail-closed behavior rather than relying on the word “type-safe.”

## Proposed placement alongside Effect v4

This section is a proposed acne design, not an existing Jev–Effect integration. Effect v4's `Layer` builds/provides services, supports scoped resources, and can hide implementation dependencies with `Layer.provide`; `Context.Service` supplies the service interface.[9]

```text
agent proposal (untrusted data)
  → decode + attach trusted session/project identity
  → deterministic capability/policy check
      deny → stop
      eligible → optional Jev risk advice (cannot expand authority)
  → approval policy / attention item, if required
  → revalidate exact approved action and current grants
  → trusted executor + restricted process environment
  → audit result
```

Suggested services:

- `ProjectPolicy`: immutable trusted grants for this session/project, allowed operations/resources, and policy version. Agent-editable plumbing blocks may propose rules but cannot activate new grants.
- `RiskAdvisor`: `assess(proposal) → risk signal | unavailable`, implemented by Jev, a fixture, or a disabled adviser. Never returns an execution capability.
- `ApprovalQueue`: binds a human decision to exact operation/arguments, cwd, relevant input versions, identity, project, expiry, and a one-use nonce. Any change requires reapproval.
- `ActionExecutor`: the only service owning raw filesystem/process/network access. Expose its gated facade, not the underlying process runner, to agent handlers.

Supply project-specific layers to trusted handlers. Keep raw implementations private with `Layer.provide`, rather than exporting them via `provideMerge`.[9] This limits accidental access in the application graph; untrusted generated JavaScript can still call ambient runtime APIs if executed in the same unrestricted process. Run generated code/processes under a real isolation mechanism as well.

Wrap the JS client's promise in an Effect adapter, propagate cancellation through its `signal` option, disable SDK retries or budget them explicitly, pin `jev-1.13.0`, decode every answer, and turn missing/invalid answers, timeout, and service failure into **review/unavailable, never allow**. Configure safe log levels because debug logs include request/response bodies.[3] No integration code was compiled in this research.

For the prototype, deterministic low-risk registered operations can be allowed independently of Jev. Unknown or arbitrary shell always enters confirmation (or is denied). Jev can escalate an otherwise eligible action, but cannot turn `deny` or `ask` into `allow`. Evaluate real executable arguments and environment, not merely an agent's natural-language summary. Prefer registered structured operations over `sh -c`; “read-only-looking” commands can invoke scripts, hooks, substitutions, or interpreters.

## Alternatives and complements

| Option | Role and trade-off | Recommendation |
| --- | --- | --- |
| Small TypeScript policy + explicit confirmations | Our proposed simplest implementation: exact registered operation/resource grants, default deny, no inference/network dependency; more prompts for arbitrary shell | **Start here** for one owner on one Mac |
| Cedar | Purpose-built authorization language/engine for principal/action/resource policies, schema validation, separately auditable policies; repository includes a WASM interface for JS/TS. It does not infer the behavior of arbitrary shell.[10] | Consider when policy complexity outgrows a small typed table; not necessary for the first prototype |
| Anthropic Sandbox Runtime | OS-enforced filesystem/network restrictions; macOS uses `sandbox-exec`, with a TS library/CLI. It is explicitly a beta research preview. Reads are unrestricted by default, so configuring write/network restrictions alone is not project isolation.[11] | Candidate complementary executor boundary; explicitly restrict reads, writes, network, sockets, and credentials, and test escapes. Never enable Apple Events from untrusted project config |
| Existing generative LLM as adviser, or no adviser | Design alternative using the same `RiskAdvisor` seam. Avoids requiring Jev specifically but retains model error/prompt-injection risks; no-adviser mode preserves deterministic safety behavior | Compare only if semantic advice materially improves the UX; neither may grant capabilities |

Sandbox Runtime's docs specifically warn that enabling Apple Events lets launched applications run outside the sandbox's filesystem/network restrictions.[11] Its presence is not a blanket security guarantee; selecting and validating the executor boundary is separate work from selecting a classifier.

## Smallest useful experiment and acceptance criteria

Proposed throwaway web UI + local Effect v4 server, consistent with the map. Begin with recorded proposals and a fake executor; do not run dangerous examples. Use a fixed labeled corpus covering routine reads, intended edits, destructive operations, cross-project paths/symlinks, credential reads, network exfiltration, nested interpreters, command substitution, misleading descriptions, and injected “approve this” text. Include outages, malformed API responses, changed proposals after approval, and replayed approvals.

Compare deterministic-only versus deterministic-plus-Jev in **shadow mode** (Jev cannot reduce prompts). Record false-safe advice on hazardous samples, false alarms, review rate, latency including failures, cost, and human disagreement. Tune on one subset, evaluate on held-out cases, and retain model/question/policy versions. No universal numeric confidence cutoff is justified by this research.[4][5]

Acceptance: forbidden capabilities never execute; required approvals cannot be bypassed by any adviser response or outage; stale/replayed approvals fail; advice is only retained if it improves review quality enough to justify its cloud dependency. High benchmark accuracy alone is not approval to make Jev an authorization boundary.

## What remains unconfirmed

No authenticated Jev calls, local latency/calibration measurements, Bun compatibility test, production SLA verification, or adversarial security evaluation were performed. No first-party Effect integration or local model deployment was established in the reviewed material. Retention terms for the owner's intended account still need review. The research resolves what Jev is and where it belongs; it does not claim prototype evidence or authorize autonomous shell execution.

## Primary sources

All accessed 2026-09-30; mutable documentation reflects that date.

1. [TypeSafe: Jev with coding agents](https://docs.typesafe.ai/introduction/coding-agents.md).
2. [Official JS SDK README, pinned](https://github.com/typesafe-ai/typesafe-sdk-js/blob/66880ccded6cb642dc1809620c2b108c33730214/README.md); [GitHub repository metadata](https://api.github.com/repos/typesafe-ai/typesafe-sdk-js).
3. SDK source at `66880ccded6cb642dc1809620c2b108c33730214`: [client.ts](https://github.com/typesafe-ai/typesafe-sdk-js/blob/66880ccded6cb642dc1809620c2b108c33730214/src/client.ts), [types.ts](https://github.com/typesafe-ai/typesafe-sdk-js/blob/66880ccded6cb642dc1809620c2b108c33730214/src/types.ts).
4. [TypeSafe: Jev 1.13 jaggedness](https://docs.typesafe.ai/model-jaggedness/jev-1.13.md), last reviewed by vendor 2026-09-17.
5. [TypeSafe: confidence](https://docs.typesafe.ai/confidence.md).
6. [TypeSafe homepage](https://typesafe.ai); [SDK package.json, pinned](https://github.com/typesafe-ai/typesafe-sdk-js/blob/66880ccded6cb642dc1809620c2b108c33730214/package.json).
7. [TypeSafe: models, aliases, limits, data handling](https://docs.typesafe.ai/models.md).
8. [TypeSafe: legal documents and ZDR](https://docs.typesafe.ai/legal.md).
9. [Effect v4 Layer source, pinned](https://github.com/Effect-TS/effect-smol/blob/3a1128c7684e04d34d9f541f77adaac38a513056/packages/effect/src/Layer.ts).
10. [Cedar first-party repository](https://github.com/cedar-policy/cedar#readme).
11. [Anthropic Sandbox Runtime first-party repository](https://github.com/anthropic-experimental/sandbox-runtime#readme).
