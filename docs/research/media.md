# acne's final medium: tradeoffs and a decision gate

Research for [ticket #6](https://github.com/saiashirwad/acne/issues/6), within [map #1](https://github.com/saiashirwad/acne/issues/1). Investigated 2026-09-30. This is desk research, not a benchmark or a prototype result.

## Recommendation

**Keep the TypeScript/Effect web surface as the leading candidate; prefer Tauri packaging if a standalone Mac app is wanted. Do not select a rewrite before measuring the prototype.** The map explicitly defers the final medium until after experiments. A browser and Tauri can share the editor, while GPUI or native AppKit requires replacing that editor. Tauri adds app packaging and host APIs, not a faster or native text widget. [1–3]

**SwiftUI chrome + AppKit/TextKit 2 is the strongest alternative for a one-Mac, native-text-first product.** GPUI is credible when custom rendering control is the actual bottleneck, but comes with a Rust frontend rewrite and a pre-1.0 framework. A TUI is attractive for an auxiliary agent/attention client, **not the leading final surface under the unchanged cmd-click requirement**: the inspected Ghostty and Kitty mouse encoders transmit Shift/Alt/Ctrl, not Command/Super. Kitty keyboard protocol support does not change that mouse encoding. [4–10]

These are recommendations, not a claim that the owner has selected the final medium.

## Constraints that matter

The map calls for stable block IDs, transclusion, per-span authorship, direct agent edits, per-author undo, cmd-click for registered commands (otherwise agent intent), and opt-click for plumbing. Prototypes use a local server and TypeScript on Effect v4; the target is the owner on one Mac. [Map](https://github.com/saiashirwad/acne/issues/1)

Consequently, rendering plain text quickly is insufficient. The real workload is **editable, attributed, addressable text changing concurrently while selection, composition, click targeting, and undo remain correct**. No UI toolkit alone supplies the map's CRDT semantics or per-author undo. This is an architectural requirement derived from the map, not a toolkit performance claim.

## Comparison

Effort ratings below are relative engineering judgments for this project, assuming the planned web prototype exists—not measured schedules. They include editing and integration, not just displaying a paragraph.

| Medium | Text latency: credible mechanism, not a ranking | Span styling and targeting | Cmd/Opt-click | Effect/TS reuse | Native feel and relative effort |
|---|---|---|---|---|---|
| Local browser app | DOM layout/paint; a virtualized editor can limit work to visible text and batch measurements. Main-thread work remains a risk. [1] | CSS-backed mark decorations and widgets; editor maps positions through edits. [1] | Mouse events expose `metaKey`/`altKey`. Must resolve editor/browser default actions. [11] | Highest: same client and local server, subject to runtime-specific APIs. | Browser chrome and web controls; **lowest initial effort**. App lifecycle/launch is still needed. |
| Tauri + web editor | Same general web pipeline, OS webview rather than a bundled custom renderer; **not inherently faster than the browser**. [2] | Same web editor approach. [1–2] | Same event model, but validate actual Mac webview and menu interactions. [2,11] | High: web frontend retained; server code can remain a separately bundled runtime/sidecar. [3] | Native host window/menu integration, web content inside; **low–medium incremental effort** for packaging, IPC, permissions, lifecycle. [2–3] |
| Rust + GPUI | GPU-accelerated custom framework; Metal on macOS, explicit elements and invalidation. This provides control, not proof of lower end-to-end latency. [4–5] | `StyledText` runs and `InteractiveText` clickable ranges exist. A styled label is not a complete collaborative editor. [6] | Mouse-down events carry modifiers; inspect platform and Alt flags in the selected GPUI version. Range-click helper alone omits the original mouse event. [6–7] | Rust UI replaces TS UI. Preserve Effect service over IPC or rewrite it; neither is zero-cost. | Custom-drawn UI, not AppKit controls; **high effort**, plus pre-1.0 API churn. [4] |
| Terminal TUI in Ghostty/Kitty | Input → PTY → app → escape stream → terminal parser/layout/render. App redraw strategy and terminal configuration matter; GPU terminal rendering alone cannot rank the app. [8–10,12] | Cell attributes can express authorship colors/styles. Application must map screen cells (or supported pixel reports) to blocks/ranges; reflow and Unicode widths complicate it. [8–10,12] | Alt/Option has a mouse bit; **no independent Command bit in inspected encoders**. Terminal bindings/selection can consume events. [8–10] | A TS TUI could retain Effect logic/runtime; web DOM/editor must be replaced. Rust TUI needs a bridge or port. | Terminal interaction, not native document editing; **medium for a read-only client, high/risky for full editor parity**. |
| SwiftUI + AppKit/TextKit 2 | Viewport-based native text layout supports large content; no evidence here that it beats a well-built web editor on acne's workload. [13] | Attributed text storage, native editing, and custom hit-testing/event routing. Current SwiftUI `TextEditor` also supports attributed text. [13–14] | AppKit `NSEvent.modifierFlags` explicitly provides `.command` and `.option`; implement routing in text view. [15] | Swift frontend rewrite; retain Effect service over a local protocol or port backend. | Best access to platform text behavior and native controls; **medium–high frontend/integration effort**. Wrap an AppKit editor in SwiftUI via `NSViewRepresentable`. [16] |

## What the capabilities mean in practice

### 1. Web/Tauri: reuse the editor, not just the language

CodeMirror's official guide describes viewport rendering, transactional updates, UTF-16 offsets, and separate DOM-write/measurement phases. Its decorations support per-range styles/attributes and inline/block widgets. This is concrete evidence that browser text need not mean rendering one gigantic DOM tree or rebuilding the whole document per agent token. It is evidence of feasibility, **not a recommendation to choose CodeMirror as the block editor without testing**: its document is a flat string, whereas acne's source of truth is a block graph. [1]

Suggested implementation boundary: maintain stable block/CRDT addresses outside the renderer; map visible authorship ranges to decorations and map gestures back to addresses. Keep interactive editing local and dispatch service work asynchronously, rather than require a server round-trip before drawing each keystroke. Use a mature editor's extension/event APIs, not ad hoc mutations inside its managed DOM. These are design recommendations grounded in the documented update model. [1]

On macOS, `metaKey` denotes Command and `altKey` denotes Option. Recognized acne gestures should suppress the relevant default action (e.g. link navigation or editor multi-selection), while ordinary selection/drag remains intact. Test both modifiers together and clicking a selected range. Prefer command dispatch over pretending arbitrary command text is a navigable URL. DOM modifier support establishes capability, not proof of conflict-free behavior. [11]

Tauri uses a Rust host plus HTML/JS in the OS webview. Its sidecar mechanism can bundle an executable server, including a JS runtime-based service, but requires per-architecture binaries and permission configuration. Thus **Tauri does not automatically run Node/Bun-specific Effect services inside the webview**. Preserve the local server as a sidecar initially, or deliberately replace its platform boundary with host commands. Browser-compatible TS can stay in the frontend. This is an integration choice, not an Effect v4 compatibility test. [2–3]

### 2. GPUI: promising control, substantial editor ownership

The first-party README explicitly calls GPUI GPU-accelerated and pre-1.0 with frequent breaking changes. It offers high-level views and low-level elements, the latter intended for custom editor layouts and large lists. Its state system queues effects and invalidates dirty windows for a subsequent frame. None of these claims implies a universal millisecond advantage. [4–5]

The source has `StyledText::with_runs` and `InteractiveText::on_click(ranges, listener)`. Importantly, that listener receives the range index, window, and app—not the original mouse event. `MouseDownEvent` separately carries modifiers. For acne, capture modifiers through the appropriate event path and preserve them across the gesture; don't assume the simplest clickable-text example fully handles cmd/opt-click. [6–7]

GPUI supplies rendering primitives, not a turnkey copy of Zed's whole editing engine. Budget selection, IME, clipboard, accessibility, navigation, text-range mapping, and CRDT integration explicitly. Recommendation: choose GPUI only after a small editor slice demonstrates a material gain worth that ownership, and pin the dependency version.

### 3. TUI: distinguish terminal shortcuts, keyboard protocols, and mouse reports

This is the decisive compatibility wrinkle:

- Ghostty's inspected `mouse_encode.zig` adds **4 for Shift, 8 for Alt, 16 for Ctrl** outside X10 mode. There is no Super/Command addition. Kitty's `mouse.c` similarly checks `GLFW_MOD_SHIFT`, `GLFW_MOD_ALT`, and `GLFW_MOD_CONTROL` in its mouse encoder. [8–9]
- Kitty's extended **keyboard** protocol defines more keyboard modifiers, including Super. That does not make Command available in the separately encoded **mouse** event. [10]
- Ghostty lets users disable mouse reporting altogether, and its Shift capture configuration determines whether Shift extends terminal selection or is passed to the application. Therefore even a supported bit is not an unconditional delivery guarantee. [12]
- Pixel-coordinate reports do not create DOM-like semantic text targets or extra modifier bits. The TUI still owns mapping coordinates to its displayed ranges. The inspected encoders include SGR pixel handling. [8–9]

A one-owner custom terminal binding, alternate chord such as Ctrl-click, or custom protocol could work around this. They are **changes to the input contract or additional terminal integration**, not demonstrated out-of-box cmd-click support. This research did not verify a reliable custom workaround across both terminals. Do not quietly substitute Ctrl for Command in the final spec.

Authorship tinting itself is feasible with terminal styling; high-fidelity proportional typography and native editing are a different question. Ghostty documents cell metrics and grapheme-width modes, including potential cursor desynchronization when applications and terminal disagree. A TUI needs explicit Unicode/wrapping hit-test tests. [12]

### 4. Native: use current SwiftUI capabilities, escape to AppKit where needed

It would be outdated to dismiss SwiftUI `TextEditor` as plain-text-only: current Apple documentation explicitly supports attributed text formatting with the appropriate initializer. That still does not establish that all of acne's arbitrary span-click, composition, and remote-edit requirements fit the high-level view. `NSViewRepresentable` is the documented path for embedding an AppKit view in SwiftUI. [14,16]

Apple describes TextKit 2's viewport-based layout and attributed-string-backed text views, including conversions between text ranges and offsets. This is a good basis for a native block editor adapter, not native support for acne's CRDT addresses. Take care not to accidentally access TextKit 1's `layoutManager`: Apple documents that this can trigger an expensive, one-way compatibility fallback. Prefer TextKit 2 APIs consistently. [13]

Use SwiftUI for surrounding UI and an AppKit text view where precise gesture routing/edit transactions require it. Treat authorship as renderer metadata derived from storage, not editable user formatting. Native undo must be connected to per-author CRDT undo rather than assumed equivalent. These are proposed integration rules, not validated implementation results.

## Decision gate: test the workload, not framework marketing

No comparable, same-machine acne benchmark was found or run. Do **not** publish a numeric latency ranking from these sources. Text shaping time, input-to-visible-edit latency, agent-update-to-paint latency, and smooth scrolling are distinct measurements.

Proposed next experiment (not work completed by this ticket):

1. Build one web editor slice with 1k/10k blocks, a very long block, dense alternating authorship spans, wrapping, emoji/combining marks, transclusions, and sustained agent edits. Record exact hardware, OS, runtime, browser/webview versions, and display refresh rate.
2. Exercise plain selection, cmd-click command/intent, opt-click plumbing, modifier changes mid-gesture, range boundaries after remote edits, IME composition, copy/paste, per-author undo, and keyboard-accessible alternatives to mouse actions.
3. Measure p50/p95 input-to-visible-update and dropped frames, both idle and under agent load. App timestamps alone do not measure physical key-to-photon delay; use visual capture for that claim. Keep service latency separate from renderer latency.
4. Compare the same editor in the chosen browser and a minimal Tauri shell. Establish an acceptance budget before interpreting results—for example, no sustained dropped frames and p95 local edit feedback within two display frames under the agreed load. This is a proposed product threshold, not a measured capability.
5. If the web implementation fails after profiling/viewport fixes, reproduce that exact failing slice in AppKit/TextKit 2; investigate GPUI when custom rendering, rather than native text behavior, is the reason to leave web. A terminal spike must first prove the unchanged Command/Option gesture contract or explicitly propose changing it.

**Exit decision:** web/Tauri remains the default candidate if editing correctness and measured responsiveness pass. Native AppKit is the first alternative for Mac text fidelity; GPUI for demonstrated custom-rendering needs; TUI for an optional secondary client unless the gesture contract changes.

## Sources and verification limits

All links are primary project documentation, official platform documentation, or first-party source. Repository source snapshots identified during this research: Zed `5d80b4e784636899e209cae89626c3be4487e14f`, Ghostty `4da7523faba68ccb4042ea20585817098a51c015`, Kitty `3d06c27a76526306ec003c1fbdce38718c844238`. Live documentation may change.

1. CodeMirror: [system guide](https://codemirror.net/docs/guide/) (viewport, update cycle, offsets, flat text model); [decorations and mouse-handler example](https://codemirror.net/examples/decoration/).
2. Tauri: [architecture](https://v2.tauri.app/concept/architecture/) (Rust host, OS webview, JS API, native window/menu infrastructure).
3. Tauri: [embedding external binaries](https://v2.tauri.app/develop/sidecar/) (sidecars, target triples, permissions).
4. GPUI: [README](https://github.com/zed-industries/zed/blob/5d80b4e784636899e209cae89626c3be4487e14f/crates/gpui/README.md) (Metal, elements, maturity).
5. Zed: [ownership and data flow in GPUI](https://zed.dev/blog/gpui-ownership) (updated 2025-12-12; dirty-window scheduling).
6. GPUI: [text elements](https://github.com/zed-industries/zed/blob/5d80b4e784636899e209cae89626c3be4487e14f/crates/gpui/src/elements/text.rs) (`StyledText`, `InteractiveText`, click callback).
7. GPUI: [input events](https://github.com/zed-industries/zed/blob/5d80b4e784636899e209cae89626c3be4487e14f/crates/gpui/src/interactive.rs) (`MouseDownEvent.modifiers`).
8. Ghostty: [mouse encoder](https://github.com/ghostty-org/ghostty/blob/4da7523faba68ccb4042ea20585817098a51c015/src/input/mouse_encode.zig).
9. Kitty: [mouse encoder](https://github.com/kovidgoyal/kitty/blob/3d06c27a76526306ec003c1fbdce38718c844238/kitty/mouse.c).
10. Kitty: [keyboard protocol](https://sw.kovidgoyal.net/kitty/keyboard-protocol/#modifiers). Keyboard capability must not be conflated with mouse capability.
11. Mozilla: [MouseEvent.metaKey](https://developer.mozilla.org/en-US/docs/Web/API/MouseEvent/metaKey), [MouseEvent.altKey](https://developer.mozilla.org/en-US/docs/Web/API/MouseEvent/altKey).
12. Ghostty: [configuration reference](https://ghostty.org/docs/config/reference), specifically `mouse-reporting`, `mouse-shift-capture`, `grapheme-width-method`, and cell metrics.
13. Apple: [What's new in TextKit and text views, WWDC22](https://developer.apple.com/videos/play/wwdc2022/10090/) (official transcript, viewport layout, attributed storage, compatibility fallback).
14. Apple: [TextEditor](https://developer.apple.com/documentation/swiftui/texteditor); [machine-readable documentation inspected](https://developer.apple.com/tutorials/data/documentation/swiftui/texteditor.json).
15. Apple: [NSEvent.ModifierFlags](https://developer.apple.com/documentation/appkit/nsevent/modifierflags-swift.struct), including Command and Option.
16. Apple: [NSViewRepresentable](https://developer.apple.com/documentation/swiftui/nsviewrepresentable).

Unconfirmed: actual latency ordering, behavior of the owner's installed Ghostty/Kitty versions/configuration, a reliable Command-click terminal workaround, exact effort in days, full Effect v4/sidecar integration, and editor/CRDT/IME compatibility. No app was built and no interaction benchmark was run. Closing this research ticket resolves the comparison, not these prototype gates or the final medium choice.
