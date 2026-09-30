# Effect v4 AI + Effect Agent as acne's agent substrate

Resolves [ticket #3](https://github.com/saiashirwad/acne/issues/3), for [map #1](https://github.com/saiashirwad/acne/issues/1). Research snapshot: 2026-09-30. This is source/doc research, **not a tested integration or prototype result**.

## Decision

**Yes: prototype on Effect v4 AI with Effect Agent as a replaceable harness.** It supplies the core model/tool interfaces and the chief/sub-agent execution patterns acne needs. Keep acne's workspace protocol, project authorization, and block storage outside the harness. Do not build another agent loop first. Effect Agent already documents bounded execution, approvals, attached children, durable background workers, and streaming.[1][2][3]

**Layers are dependency wiring, not a security sandbox.** Give every project an independently constructed, resource-checking workspace service; use explicit tool grants and authorization as well. A model prompt or a project ID parameter must not confer authority.[2][4]

**ChatGPT OAuth is now officially feasible, but not plug-and-play with Effect AI.** OpenAI documents a public-client Sign in with ChatGPT flow for open-source clients and public Responses inference. Its preview wire restrictions require an adapter audit and likely request/result translation. Do not assume copying a Codex token into the ordinary OpenAI provider is sufficient.[7][8][9]

## What exists today

| Need | Evidence and implication for acne |
| --- | --- |
| Typed tools and structured output | Effect Agent consumes native Effect AI `Tool`/`Toolkit` declarations with parameter, success, and failure schemas, implemented through `Toolkit.toLayer`. It decodes model tool calls and supports an agent output schema. Choose `failureMode: "return"` deliberately for recoverable model-visible errors; the default `"error"` fails the run.[1][2] |
| Streaming | `AgentRuntime.run`, `stream`, and `start` share the loop. Semantic events cover text/reasoning deltas, tool activity, approval and terminal status. Streams have bounded backpressure; interrupting the sole ephemeral stream consumer interrupts execution. A scoped start handle supports re-observation. Persisted/durable work has a different cancellation contract.[1] |
| Multiple providers | Current Effect source includes Anthropic, OpenAI Responses, OpenAI-compatible, OpenRouter, and TypeSafe packages (the last is a decision-model provider, not another general chat model). Child agents can have their own model Layers. Do not infer identical tool/schema/stream behavior across providers from one interface.[3][5] |
| MCP client | **Effect Agent**, rather than the inspected core AI module, supplies `McpClient`: Streamable HTTP and stdio transports; bounded tool discovery becomes native tools plus handler Layers. Connections are scoped. Server sampling/elicitation requests are declined and disconnected event streams are not resumed. Credentials are supplied through transport headers/HTTP client; this is not a complete MCP OAuth login UX.[2][6] |
| MCP server | Effect's `McpServer` implements stdio and Streamable HTTP layers and registration for toolkits, resources, and prompts. This can expose acne blocks/commands to external agents; acne must still implement caller authentication and project authorization.[5] |
| Sub-agents | `Subagent.make` presents a specialist as a tool with input/output projections. Children have separate conversation context and do not inherit the parent's toolkit. In-memory attached, durable attached, and durable background lifetimes are documented. Background work survives parent completion/abort; attached cancellation propagates. This directly fits short-lived chief runs supervising longer project tasks.[3][4] |
| Limits and permissions | Child grants narrow tool access; nested delegation has depth/slot/budget ceilings. Shared delegation reservations exist, but a global spending quota still needs host accounting. Handoff is explicitly unsupported in the inspected subagent reference.[4] |
| Local durability | The docs distinguish completed history retention from durable execution, and describe a Node host with SQLite. Uncertain external effects are not blindly replayed after a crash. This is operational state, not a replacement for acne's chosen CRDT workspace.[1][2] |

### Version trap

The inspected Effect Agent tree is `0.1.0-beta.156` and pins Effect/provider packages to `4.0.0-rc.117`; its examples import `effect/unstable/ai`. Inspected Effect main declares `4.0.0-rc.118` and exports `effect/ai` instead. **Do not mix main-branch examples with the harness's pinned release.** Start from the harness's exact dependency set and lock it; compile and test before upgrading. These are repository manifest observations, not a claim that every inspected main commit has been published to npm.[10]

## Proposed acne composition (design recommendation)

1. **One local Effect application runtime.** It owns credential storage, HTTP, durable scheduling if needed, and the workspace store. Browser/UI code never receives provider refresh tokens.
2. **Project capability Layers.** Construct `ProjectWorkspace` with a host-authorized project root and author identity captured outside model input. Its read/write methods check stable block IDs against that root, including transclusions. Tool handlers depend on this restricted service, not the global store. Enforce the same checks on MCP calls and direct user-to-child routes.
3. **Chief toolkit:** inspect allowed project summaries, start/inspect/cancel project workers, and publish attention items. Do not give the chief unrestricted editing merely because it orchestrates. Child toolkits get project-specific block tools. Install `RunToolAuthorization`: the documented default is allow-all, and hidden tool schemas alone are not resource authorization.[2]
4. **Attached for quick requests; durable background for independent tasks.** Start with in-memory attached children in the throwaway prototype. Use the durable host when tasks must outlive chief runs or app restarts. Durable services are captured at runtime acquisition: wrapping a later worker call in a different Layer does not replace them. Bind policy at host construction and derive trusted per-run project capabilities there.[1][3]
5. **Workspace is memory.** Build each short run's context from authorized blocks and write durable findings back as attributed block operations. Retain execution/thread logs only for debugging, approvals, and recovery. Effect Agent's context preparation/history hooks are useful plumbing, not the workspace's canonical memory.[1]
6. **Own an agent-facing protocol.** Define stable agent/thread/run/project IDs plus submit, events, cancel, reply, approval and settlement semantics. Adapt Effect Agent behind it; external agents need not speak its internal event union. Directly addressing a child should use that same protocol and checks, not depend on a living chief.

## Codex / ChatGPT OAuth: feasible path and remaining work

OpenAI's current official docs distinguish ordinary Codex subscription login from a third-party **Sign in with ChatGPT plan-usage** integration. The latter offers initial dynamic registration (`dynamic_agent_client`), an issued per-account client ID, stable host ID, browser loopback callback, authorization code + PKCE, state/nonce validation, and ID-token verification. No client secret or partner API key is required for the documented direct flow. Store/refresh credentials per verified identity and issued client ID, atomically and privately.[7][11]

For inference, the documented route is `https://api.openai.com/v1/responses`, **not ChatGPT backend-api endpoints**. Obtain the account-specific model catalog, use OAuth bearer auth, set `store: false` and `stream: true`, provide history explicitly, and wait for `response.completed`. A stream can fail after emitting text, including subscription usage failures.[8]

The preview is narrower than generic Responses: no HTTP `previous_response_id`; no `max_output_tokens`, `temperature`, `conversation`, and several other fields; system-role message items are rejected; function/custom tools must be namespaced or supplied through `additional_tools`. Hosted MCP, Code Interpreter, file search, and Responses `tool_search` are unsupported. **Local MCP execution remains possible** because acne can run its tools itself.[9]

Effect's OpenAI client already accepts a bearer credential and client transformation, but that is only transport plumbing. Inspected `OpenAiLanguageModel` emits ordinary function tools at top level and chooses system/developer roles by model; it is not evidence of SIWC conformance. The inspected Effect Agent packages contain no OAuth/Codex implementation found by name search.[6][12]

Therefore implement a separate auth service plus SIWC-compatible model adapter (or a carefully tested request/response translation layer): enforce the restricted request shape, translate namespaced tool calls, inject fresh tokens per request, serialize refresh rotation, fetch models, handle revocation/usage failures, and never fall back silently to paid API billing. API-key mode should be an explicit alternative. First test text, one tool round-trip, streaming failure, and token refresh with the owner's account. Account eligibility and end-to-end compatibility were **not verified** here.

An alternative is to host Codex app-server behind acne's protocol. OpenAI documents SIWC configuration, local tool/child-agent execution and thread resume, but with its `env_key` setup the host must refresh the token and restart app-server. That imports another agent runtime rather than simply authenticating Effect's own agent loop.[9]

## Compared with building on Pi

Pi's current first-party READMEs describe `@earendil-works/pi-ai` and `@earendil-works/pi-agent-core` (do not assume older package names). Pi AI has a broad provider catalog, provider-owned auth/refresh and credential-store abstraction, tool calling, streaming, and context serialization. It still explicitly labels its OpenAI Codex provider **legacy**. That is evidence of implementation, not proof it meets the new official SIWC flow.[13]

Pi agent-core supplies the loop, tool events, cancellation, steering/follow-up queues, context conversion, and before/after tool hooks. Its README points to separate `pi-mcp` and `pi-codemode` packages. Thus “Pi has no MCP” would be an outdated comparison.[14]

| Choice | Trade-off for acne |
| --- | --- |
| Effect AI alone | Clean Effect-native model/tool substrate, but acne would own more orchestration, run limits, approval plumbing and child lifecycle code that Effect Agent already supplies.[1–4] |
| Effect AI + Effect Agent | Best architectural fit for the standing Effect/layer decision; explicit child budgets, typed services, native schemas and documented durability. Costs: prerelease coupling, integration verification, and maintaining acne's boundary rather than adopting every harness concept.[1–4][10] |
| Pi AI + Pi agent-core | Attractive provider/auth breadth and ready interactive agent controls. Would require a Promise/AsyncIterable/AbortSignal-to-Effect boundary and TypeBox/Effect Schema translation or a deliberate second schema system. The inspected core README does not establish equivalence to Effect Agent's durable child reservation/recovery model; do not call it impossible, but budget separate orchestration evaluation.[13][14] |

**Recommendation:** use Effect Agent for the first prototype; keep Pi as an alternate provider/agent adapter behind acne's protocol if provider support becomes a blocker. Do not adopt Pi solely to obtain “Codex OAuth” without checking its current flow against OpenAI's official requirements.

## Gaps acne still owns and prototype exit criteria

- **Block semantics:** CRDT mutations, author attribution, per-author undo, project/transclusion boundaries, idempotent writes and reconciliation. Harness history/durability does not implement these product semantics.
- **Trust boundary:** deny cross-project block access; constrain filesystem, process environment and remote MCP capabilities. Layers cannot stop arbitrary JavaScript or shell code from accessing ambient OS authority. Do not grant a general shell and describe it as project isolation.
- **User protocol/UI:** attention items, direct child conversations, reconnectable event transport, safe provisional streaming, approvals and operator-visible unknown outcomes.
- **OAuth:** the adapter and secure credential lifecycle above; confirm account eligibility before depending on subscription access.
- **Compatibility:** lock versions, compile a real run, and exercise provider tools/errors. No dependency install, paid model call or code prototype was performed for this research.

Prototype passes only if: (1) chief delegates two scoped children and streams results; (2) a malicious cross-project block ID is denied, including through MCP; (3) a child remains directly addressable after chief completion using the chosen lifecycle; (4) cancellation releases scoped resources; (5) crash/restart does not duplicate a block edit; and (6) an OAuth tool round-trip, refresh and terminal failure work without forbidden wire fields. Begin with an explicit API key if OAuth would obscure the architecture test; keep OAuth acceptance as a separate gate.

## Primary sources

Repository links are pinned to inspected commits where possible. Documentation URLs are live snapshots, retrieved 2026-09-30.

1. [Effect Agent: Run & stream](https://github.com/danieljvdm/effect-agent/blob/b6ec526daf05a71d318fec0b31c5b31db54fed35/docs/guide/run-agents.md).
2. [Effect Agent: Tools & layers, authorization and MCP](https://github.com/danieljvdm/effect-agent/blob/b6ec526daf05a71d318fec0b31c5b31db54fed35/docs/guide/tools.md).
3. [Effect Agent: Subagents](https://github.com/danieljvdm/effect-agent/blob/b6ec526daf05a71d318fec0b31c5b31db54fed35/docs/guide/subagents.md).
4. [Effect Agent: Subagent policies and recovery](https://github.com/danieljvdm/effect-agent/blob/b6ec526daf05a71d318fec0b31c5b31db54fed35/docs/reference/subagents.md).
5. Effect [provider packages](https://github.com/Effect-TS/effect/tree/0af6d0c8ceaf08656f76bba41e5965bbf9a76bac/packages/ai) and [McpServer source](https://github.com/Effect-TS/effect/blob/0af6d0c8ceaf08656f76bba41e5965bbf9a76bac/packages/effect/src/ai/McpServer.ts).
6. [Effect Agent packages/source tree](https://github.com/danieljvdm/effect-agent/tree/b6ec526daf05a71d318fec0b31c5b31db54fed35/packages), including [McpClient implementation](https://github.com/danieljvdm/effect-agent/blob/b6ec526daf05a71d318fec0b31c5b31db54fed35/packages/effect-agent/src/capabilities/McpClient.ts).
7. [OpenAI: SIWC registration and sign-in](https://developers.openai.com/siwc/token-sharing-open-source/sign-in.md).
8. [OpenAI: SIWC models and inference](https://developers.openai.com/siwc/token-sharing-open-source/models-and-inference.md).
9. [OpenAI: SIWC preview limitations, including app-server](https://developers.openai.com/siwc/token-sharing-open-source/preview-limitations.md).
10. [Effect Agent dependency catalog](https://github.com/danieljvdm/effect-agent/blob/b6ec526daf05a71d318fec0b31c5b31db54fed35/package.json), [harness version](https://github.com/danieljvdm/effect-agent/blob/b6ec526daf05a71d318fec0b31c5b31db54fed35/packages/effect-agent/package.json), and [Effect version/exports](https://github.com/Effect-TS/effect/blob/0af6d0c8ceaf08656f76bba41e5965bbf9a76bac/packages/effect/package.json).
11. [OpenAI: Codex authentication](https://developers.openai.com/codex/auth).
12. Effect [OpenAiClient bearer/client transformation](https://github.com/Effect-TS/effect/blob/0af6d0c8ceaf08656f76bba41e5965bbf9a76bac/packages/ai/openai/src/OpenAiClient.ts) and [OpenAiLanguageModel request/tool conversion](https://github.com/Effect-TS/effect/blob/0af6d0c8ceaf08656f76bba41e5965bbf9a76bac/packages/ai/openai/src/OpenAiLanguageModel.ts).
13. [Pi AI README: providers, auth and legacy Codex](https://github.com/badlogic/pi-mono/blob/99e85eb7d1ec5c1424445692d87ae7825c20cb75/packages/ai/README.md).
14. [Pi agent-core README: loop, hooks, MCP and Code Mode](https://github.com/badlogic/pi-mono/blob/99e85eb7d1ec5c1424445692d87ae7825c20cb75/packages/agent/README.md).
