# Per-category inlay hint checkboxes for Dart (Stage 2) — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development
> (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use
> checkbox (`- [ ]`) syntax for tracking.
>
> **Audience:** an AI coding agent (or human) working in BOTH repositories on this machine:
> * plugin: `~/development/projects/privat/dart-intellij-third-party` (feature branches via
>   worktrees under `.claude/worktrees/`, base `origin/main` of the
>   `flutter/dart-intellij-third-party` fork)
> * SDK: `~/development/projects/privat/dart-sdk/sdk` (`dart-lang/sdk` checkout)
>
> **Last verified against the real code: 2026-08-21** — `dart-lang/sdk` main @ `583f4452117`,
> plugin upstream main @ `d8a651c2` (#614 merged), IntelliJ Platform 2026.1.3 (bytecode of
> `intellij.platform.lang.jar` inspected for the declarative-inlay APIs). Facts that must be
> re-verified at execution time are marked **[verify]**. See "Verified facts" at the end.

**Goal:** One checkbox per server-side inlay hint category (variable types, return types,
parameter types, type arguments, dot shorthand types, parameter-name mode) in
Settings → Editor → Inlay Hints, wired through to the Dart Analysis Server over LSP-over-Legacy.

**Architecture:** Stage 1 (PR #617) added two kind-level provider checkboxes
(`dart.parameter.names`, `dart.types`) that filter client-side by `InlayHint.kind`. The
per-category granularity of Stage 2 is *not visible* on the wire (the client only sees
`kind: Type|Parameter`), so the server must stop producing filtered categories. The server
already honors `dart.inlayHints` configuration in the shared `InlayHintHandler` — only the
*transport* of that configuration is missing for LSP-over-Legacy. Stage 2 therefore (a) teaches
the legacy protocol the standard LSP configuration flow (`workspace/didChangeConfiguration`
notification client→server, `workspace/configuration` request server→client, both wrapped in the
existing `lsp.handle` envelope), and (b) adds `<option>` sub-checkboxes to the two existing
declarative providers, whose state the plugin serializes into the `dart.inlayHints` section.

**Tech Stack:** Dart (analysis_server), Kotlin/Java (IntelliJ plugin), IntelliJ declarative
inlay-provider EP, JUnit3-style fixture tests, `package:test` LSP-over-Legacy tests.

## Global Constraints (binding — copy of the standing project rules)

1. **Every decision and every change is validated** — against an issue in the relevant repository
   and/or against the actual source code in the checkout. Nothing is guessed, assumed by default,
   or invented; unverifiable statements are surfaced as questions instead. Items marked
   **[verify]** MUST be validated at execution time before use.
2. **Follow the existing code, documentation and formatting.** Reviewers should see familiar
   idioms (plugin: 2-space in `hints/`, 4-space in `lsp/`; SDK: `dart format`).
3. **Reuse existing code / classes / methods / test helpers.** This holds even where the existing
   code is suboptimal — improvement proposals go into a separate `.md`, not into the change.
4. **Commits authored by `Ralph Bergmann <ralph@dasralph.de>` only — no `Co-Authored-By:` of any
   kind** (Google CLA check fails otherwise). Verify `git config user.email` before committing.
5. **Workflow: develop → test locally → only then commit & push.** A PR (or a push to a PR
   branch) may only contain locally verified state — never "will enable after testing".
6. TDD: failing test first, watch it fail, minimal implementation, watch it pass.
7. No `!!` in Kotlin — use `requireNotNull(...)` (also in tests).
8. Plugin: `git add <explicit files>` only, never `git add -A` (CRLF trap in
   `third_party/gradlew.bat`). Gradle needs
   `export JAVA_HOME="$HOME/Applications/IntelliJ IDEA.app/Contents/jbr/Contents/Home"`.
9. `third_party/thirdPartySrc/platform-lsp/**` (vendored JetBrains code) is off-limits without
   owner approval. `thirdPartySrc/analysisServer/**` (vendored Google client) IS editable by
   upstream practice — helin24 herself changed `RequestUtilities.java` in #615 and the
   `workspace/applyEdit` dispatch lives in `RemoteAnalysisServerImpl.java` — but expect the repo
   code-review skill to flag it; reference this precedent in the PR text.

---

## Why this transport (decision record)

helin24's stated priority (PR #617, comment 5365375279, 2026-08-21): *"limit any additional code
that will have to be considered again when we migrate to full LSP."* Three candidate transports:

* **A — standard LSP configuration flow over the `lsp.handle` envelope (CHOSEN).**
  `workspace/didChangeConfiguration` + `workspace/configuration` are exactly what the full-LSP
  server already implements. On migration to full LSP only the envelope disappears; the plugin's
  config-building code, the server's handler and the tests of the *behavior* all carry over.
  Missing pieces today are small and generic (see facts F5–F7): the LoL envelope rejects LSP
  *notifications*, and the `didChangeConfiguration` handler is registered LSP-only because its
  implementation calls LSP-only members.
* **B — piggyback a `configuration` field on legacy `server.setClientCapabilities`.** Smaller SDK
  surface, but capabilities ≠ configuration (distinction already made in the PR discussion), it
  invents a non-LSP transport that full LSP will discard, and it duplicates the config-refresh
  semantics that A gets for free. Fallback if the Dart team rejects A.
* **C — new dedicated legacy request (e.g. `server.updateLspConfiguration`).** Most
  legacy-idiomatic, but adds brand-new legacy API surface that the full-LSP migration
  obsoletes — the exact thing helin24 asked to avoid. Rejected.

A's flow end-to-end (all message names are existing LSP/legacy protocol):

```
plugin                                          Dart Analysis Server (legacy)
------                                          -----------------------------
server.setClientCapabilities
  { lspCapabilities: { workspace: { configuration: true, ... } } }   ──▶  stores editorClientCapabilities (F4)
lsp.handle { lspMessage: workspace/didChangeConfiguration }          ──▶  shared handler (SDK CL, Task 2)
                                                                          server calls fetchClientConfiguration
◀── lsp.handle { lspMessage: workspace/configuration request }            via existing sendLspRequest (F6)
plugin answers with [{ ...global "dart" section incl. inlayHints }]  ──▶  lspClientConfiguration.replace(...)
(next textDocument/inlayHint request)                                ──▶  InlayHintHandler already honors it (F2)
```

**Generality note (2026-08-21):** transport A is deliberately NOT inlay-hint-specific — the fetch
replaces `lspClientConfiguration` as a whole, so every future `dart.*` workspace option
(documentation, showTodos, renameFilesWithClasses, …) rides the same flow; plugin-side the only
seam to extend is `DartLspInlayHintsConfiguration.buildDartSection()` (Task 5), which should grow
into a general `dart`-section builder when the second consumer arrives. Big-picture comment with
two design questions posted on sdk#64101 (issuecomment-5374291944): (a) global-only vs
per-workspace-folder `ConfigurationItem`s (relevant for resource-scoped settings like
`dart.analysisExcludedFolders`), (b) restart semantics — which changes the server re-applies live
vs the `initializationOptions` tier that is fixed at initialize time (VS Code hard-codes that
split client-side in Dart-Code's `getSettingsThatRequireRestart()`, incl. `closingLabels`).

## Open questions to settle with the maintainer BEFORE implementing

* Blessing for transport A (SDK team is the second gatekeeper — file the SDK issue first, Task 0).
* Scoping + restart semantics — asked on sdk#64101 (see generality note above); fold the answers
  into Task 2 (per-folder items or not) and Task 8 (which options may need a DAS restart).
* Parameter-name hints are a 3-state server setting (`none | literal | all`, default `all`,
  fact F3). Proposed UI mapping: parent checkbox off → `none`; parent on → sub-option
  **"Only for literal arguments"** off = `all`, on = `literal`. Alternative (2 options "Literal
  arguments"/"Non-literal arguments") cannot express the server's state space (no
  "non-literal-only" mode) — do not use it.
* Category options default to ON (`enabledByDefault="true"`), matching the server defaults
  (F3) — effective only while the parent checkbox (Stage 1, default OFF) is on. Confirm.

---

## Part 1 — Dart SDK CL (dart-lang/sdk)

Model CL for process/style: `7c18d1fa0e5` ("Make textDocument/inlayHint a shared handler",
CL 536565, sdk#64061) — see `docs/superpowers/plans/2026-07-30-sdk-share-inlay-hint-handler.md`
(kept as the SDK-handoff template; follow its Step 0/landing/tag-hunting procedure).

### Task 0: File the upstream SDK issue — ✅ DONE 2026-08-21

**Files:** none (GitHub: dart-lang/sdk)

- [x] **Step 1:** Duplicate search done 2026-08-21 (queries: `lsp over legacy configuration`,
  `didChangeConfiguration`, `in:title lsp-over-legacy`): no existing issue. Nearest neighbors,
  referenced instead: sdk#60326 (the original `dart.inlayHints` config support, closed) and
  sdk#64013 (dart-fix LSP migration umbrella).
- [x] **Step 2:** Filed as **[dart-lang/sdk#64101](https://github.com/dart-lang/sdk/issues/64101)**
  ("Support client configuration (workspace/didChangeConfiguration + workspace/configuration)
  over LSP-over-Legacy"), cross-linked from flutter/dart-intellij-third-party#622
  (issuecomment-5368394167). Now: wait for direction (DanTup/bwilkerson) before writing code,
  as with sdk#64061.

### Task 1: Accept LSP notifications in the `lsp.handle` envelope

**Files:**
- Modify: `pkg/analysis_server/lib/src/handler/legacy/lsp_over_legacy_handler.dart`
- Test: `pkg/analysis_server/test/lsp_over_legacy/configuration_test.dart` (new, next to the
  existing `inlay_hint_test.dart`; helpers in `abstract_lsp_over_legacy.dart`)

**Interfaces:**
- Consumes: `LspHandleParams.lspMessage` (existing), `NotificationMessage.canParse/fromJson`
  (existing, `package:language_server_protocol`).
- Produces: `lsp.handle` accepts `NotificationMessage` payloads; the legacy response is an empty
  `LspHandleResult` acknowledgment (the plugin already ignores it — its
  `DartBridgeLspServer.forwardNotification` KDoc documents exactly that expectation, F8).

Today `handle()` only parses `RequestMessage` (requires `id`) and answers
`INVALID_PARAMETER` for anything else (F5) — an LSP notification has no `id`, so
`workspace/didChangeConfiguration` cannot even reach the server.

- [ ] **Step 1: Write the failing test** — in the new test file, wrap a
  `workspace/didChangeConfiguration` `NotificationMessage` in `lsp.handle` (reuse the sending
  helper of `abstract_lsp_over_legacy.dart` **[verify name]**; remember `initializeServer()`, not
  `waitForTasksFinished()` — lesson from CL 536565 patch set 3) and assert the legacy response is
  a success acknowledgment, not `INVALID_PARAMETER`.
- [ ] **Step 2:** Run it, watch it fail with `INVALID_PARAMETER`.
- [ ] **Step 3: Minimal implementation** — in `handle()`, after the `RequestMessage.canParse`
  branch, add a `NotificationMessage.canParse` branch that dispatches via the same
  `server.immediatelyHandleLspMessage(...)` path **[verify:** exact dispatch method for
  notifications; `immediatelyHandleLspMessage` takes a `RequestMessage` today — the LSP servers
  route notifications through the same `ServerStateMessageHandler.handleMessage`; mirror however
  `LspAnalysisServer` distinguishes them**]** and answers `sendResult(LspHandleResult(null))`
  (shape **[verify]** against how `LspHandleResult` serializes an absent response).
- [ ] **Step 4:** Run the test, watch it pass. Run the whole `lsp_over_legacy` suite.
- [ ] **Step 5:** Commit (`dart format` first).

### Task 2: Share the configuration flow with the legacy server

**Files:**
- Modify: `pkg/analysis_server/lib/src/lsp/handlers/handler_states.dart` (move
  `WorkspaceDidChangeConfigurationMessageHandler.new` from `lspHandlerGenerators` into
  `InitializedStateMessageHandler.sharedHandlerGenerators`)
- Modify: `pkg/analysis_server/lib/src/lsp/handlers/handler_workspace_configuration.dart`
- Modify: `pkg/analysis_server/lib/src/analysis_server.dart` (base class seam)
- Modify: `pkg/analysis_server/lib/src/lsp/lsp_analysis_server.dart`
- Test: extend `pkg/analysis_server/test/lsp_over_legacy/configuration_test.dart`

**Interfaces:**
- Consumes: `sendLspRequest(Method.workspace_configuration, ConfigurationParams(...))` — already
  implemented on BOTH servers (base declaration; legacy impl wraps a reverse `lsp.handle`, F6);
  `editorClientCapabilities` — already populated on the legacy server from
  `setClientCapabilities.lspCapabilities` (#614's SDK side, F4), default
  `fixedBasicLspClientCapabilities` has NO `workspace.configuration` → gate stays closed for
  clients that don't opt in (F7).
- Produces: `AnalysisServer.fetchClientConfiguration()` (new, base class) — fetches the global
  `dart` section via `workspace/configuration` and calls `lspClientConfiguration.replace(...)`;
  `LspAnalysisServer.fetchClientConfigurationAndPerformDynamicRegistration()` keeps its name and
  becomes "fetch (with workspace folders) + dynamic registration" on top of it.

Split carefully — the current method (F6) mixes three concerns:
1. per-workspace-folder config items — LSP-only (`_workspaceFolders`); the legacy server sends
   only the global `ConfigurationItem(section: 'dart')`. `dart.inlayHints` is global-only anyway
   (`LspGlobalClientConfiguration` doc, F3).
2. `lspClientConfiguration.replace(...)` + `affectsAnalysisRoots/affectsAnalysisResults`
   reactions — shared (both servers own an `lspClientConfiguration`, F1). **[verify]** which of
   the two reaction paths (`_refreshAnalysisRoots`, `reanalyze`) exist/apply on the legacy server;
   if a legacy equivalent is missing, restrict the shared reaction to `reanalyze()` and note it in
   the CL description.
3. `capabilitiesComputer.performDynamicRegistration()` — LSP-only, stays in the LSP override.

- [ ] **Step 1: Write the failing end-to-end test** — in `configuration_test.dart`:
  `setClientCapabilities` with `lspCapabilities: {workspace: {configuration: true}}` →
  send `workspace/didChangeConfiguration` (Task 1 envelope) → expect the server to send a reverse
  `lsp.handle` request `workspace/configuration` (assert method + `section: 'dart'`) → respond
  with `[{'inlayHints': {'variableTypes': {'enabled': false}}}]` → request
  `textDocument/inlayHint` on a file with a `final x = 1;` variable and a positional call
  argument → assert parameter-name hints still present, variable-type hint gone.
- [ ] **Step 2:** Run, watch it fail (no reverse request is sent today).
- [ ] **Step 3:** Implement the seam described under "Interfaces". Guard with
  `editorClientCapabilities?.configuration ?? false` exactly like the existing code (F6).
- [ ] **Step 4:** Test green; run `lsp_over_legacy/test_all.dart` and the LSP `configuration`
  tests (`pkg/analysis_server/test/lsp/configuration_test.dart` **[verify name]**) for
  regressions.
- [ ] **Step 5:** Update the `lsp.handle` paragraph of
  `pkg/analysis_server/tool/spec/spec_input.html` (lines ~150–200: document that notifications
  are accepted and that `workspace/configuration` may flow server→client when the capability was
  set; regenerate spec output per `tool/spec` README **[verify command]**). Commit.

### Task 3: Land + establish the version floor

- [ ] **Step 1:** CL through review (relatedIssue = Task 0 issue). Follow-ups per template plan.
- [ ] **Step 2:** After landing, find the first `X.Y.Z-N.0.dev` tag containing the commit
  (`git merge-base --is-ancestor <commit> refs/tags/<tag>` bisection, as done for
  `3.14.0-139.0.dev`). This becomes `MIN_LSP_INLAY_HINTS_CONFIG_SDK_VERSION` in Part 2.

---

## Part 2 — Plugin PR (flutter/dart-intellij-third-party)

Prereq: Part 1 landed and a dev SDK with it exists. Worktree off upstream `main` (Stage 1 /
PR #617 merged). All Stage-1 names below exist on the #617 branch: providers
`DartParameterNamesInlayHintsProvider` / `DartTypesInlayHintsProvider` (IDs
`dart.parameter.names`, `dart.types`), filter `DartLspInlayHintSupport`
(`third_party/src/main/java/com/jetbrains/lang/dart/lsp/DartLspInlayHintSupport.kt`), tests
`DartLspInlayHintSupportTest`.

### Task 4: `<option>` sub-checkboxes in plugin.xml + bundle

**Files:**
- Modify: `third_party/src/main/resources/META-INF/plugin.xml` (the two
  `codeInsight.declarativeInlayProvider` blocks added in Stage 1)
- Modify: `third_party/src/main/resources/messages/DartBundle.properties`

**Interfaces:**
- Produces: option IDs consumed by Tasks 5/6 (verbatim):
  `dart.parameter.names.only.literal` (under `dart.parameter.names`, `enabledByDefault="false"`),
  and under `dart.types`: `dart.types.variable` / `dart.types.return` / `dart.types.parameter` /
  `dart.types.type.arguments` / `dart.types.dot.shorthand` (all `enabledByDefault="true"`,
  `showInTree="true"`).
- Platform contract (verified against 2026.1.3 bytecode, F9): nested
  `<option optionId= enabledByDefault= bundle= nameKey= descriptionKey= showInTree=/>` elements;
  state read via `DeclarativeInlayHintsSettings.isOptionEnabled(optionId, providerId): Boolean?`
  (null = XML default; defaults resolvable via
  `InlayHintsProviderFactory.getProviderInfo(language, providerId).options`).

- [ ] **Step 1:** Add the `<option>` elements. Mapping to server categories (F3):
  variableTypes ← `dart.types.variable` ("Variable types"), returnTypes ← `dart.types.return`
  ("Return types"), parameterTypes ← `dart.types.parameter` ("Parameter types"),
  typeArguments ← `dart.types.type.arguments` ("Type arguments"),
  dotShorthandTypes ← `dart.types.dot.shorthand` ("Dot shorthand types"),
  parameterNames mode ← `dart.parameter.names.only.literal` ("Only for literal arguments").
  Bundle keys follow the Stage-1 pattern `dart.inlay.hints.<provider>.<option>.name`.
- [ ] **Step 2:** `runIde` smoke check: options render under both providers, defaults correct.
  (No unit test — pure registration; behavior is tested via Task 5.)
- [ ] **Step 3:** Commit.

### Task 5: Config builder — settings → `dart.inlayHints` JSON

**Files:**
- Create: `third_party/src/main/java/com/jetbrains/lang/dart/lsp/DartLspInlayHintsConfiguration.kt`
- Test: `third_party/src/test/java/com/jetbrains/lang/dart/lsp/DartLspInlayHintsConfigurationTest.kt`

**Interfaces:**
- Consumes: `DeclarativeInlayHintsSettings.isProviderEnabled/isOptionEnabled`,
  `InlayHintsProviderFactory.getProviderInfo` (default fallback — same pattern as Stage 1's
  `DartLspInlayHintSupport.isProviderEnabled`), option IDs from Task 4.
- Produces: `object DartLspInlayHintsConfiguration { fun buildDartSection(): JsonObject }`
  returning the *global `dart` section*, i.e. `{"inlayHints": {...}}`, with the exact server
  schema (F3): `variableTypes/returnTypes/parameterTypes/typeArguments/dotShorthandTypes`
  → `{"enabled": <bool>}`, `parameterNames` → `{"enabled": "none"|"literal"|"all"}`.
  Parent checkbox OFF ⇒ all its categories disabled (`parameterNames: "none"` resp. all five
  type keys `false`) — the server then stops sending what the Stage-1 client filter would drop
  anyway.

- [ ] **Step 1: Failing tests** (fixture test like `DartLspInlayHintSupportTest`, reset settings
  in `tearDown` via `loadState(DeclarativeInlayHintsSettings.HintsState())`):
  defaults (both parents off) → everything disabled; types parent on + `dart.types.return` off →
  `returnTypes.enabled == false`, `variableTypes.enabled == true`; parameter parent on + literal
  option on → `parameterNames.enabled == "literal"`; option on + parent off → still `"none"`.
- [ ] **Step 2:** Watch them fail (class missing). **Step 3:** implement. **Step 4:** green.
- [ ] **Step 5:** Commit.

### Task 6: Answer `workspace/configuration` from the server

**Files:**
- Modify: `third_party/thirdPartySrc/analysisServer/com/google/dart/server/internal/remote/RemoteAnalysisServerImpl.java`
  (`processLspRequestFromServer` — today it dispatches only `workspace/applyEdit`, F8)
- Modify: `third_party/thirdPartySrc/analysisServer/com/google/dart/server/internal/remote/utilities/RequestUtilities.java`
  (response builder next to `generateShowMessageRequestResponse`)
- Modify: `third_party/src/main/java/com/jetbrains/lang/dart/analyzer/DartAnalysisServerService.java`
  (supply the section content — keep vendored `RemoteAnalysisServerImpl` free of IntelliJ
  settings knowledge; follow how `workspace/applyEdit` bubbles up **[verify]** the exact
  listener/consumer seam used there)
- Test: `third_party/src/test/java/com/jetbrains/lang/dart/lsp/DartBridgeLspServerTest.kt`-style
  test at the `RemoteAnalysisServerImpl` level **[verify]** where the existing
  `processLspRequestFromServer` tests live, mirror them.

**Interfaces:**
- Consumes: reverse `lsp.handle` request shape (F8) — legacy request with
  `params.lspMessage = {id, method: "workspace/configuration", params: {items: [...]}}`.
- Produces: legacy response `{"id": <dasRequestId>, "result": {"lspResponse": {"id": <lspId>,
  "jsonrpc": "2.0", "result": [<one object per requested item>]}}}` where each requested
  `section: "dart"` item gets `DartLspInlayHintsConfiguration.buildDartSection()`.

- [ ] Steps: failing test (assert response array matches `items` length and carries
  `inlayHints`) → fail → implement dispatch branch + builder → green → commit.

### Task 7: Push trigger + cache invalidation

**Files:**
- Modify: `third_party/src/main/java/com/jetbrains/lang/dart/lsp/DartLspInlayHintSupport.kt`
- Modify: `third_party/src/main/java/com/jetbrains/lang/dart/lsp/DartBridgeLspServer.kt`
  (its `forwardNotification(method, params)` exists and is currently unused, F8 — first real
  caller; Task 1 makes the server accept it)
- Modify: `third_party/src/main/java/com/jetbrains/lang/dart/analyzer/DartAnalysisServerService.java`
  (initial push after `server.setClientCapabilities`)
- Test: extend `DartLspInlayHintSupportTest` / bridge test.

There is no settings-changed event for `DeclarativeInlayHintsSettings` (verified: the 2026.1.3
class fires no message-bus topic, F9). Trigger strategy:
1. Initial push right after capabilities are sent (`DartAnalysisServerService`, next to the
   existing `buildLspCapabilities` call site at `analysisServer.server_setClientCapabilities`).
2. Drift detection in `DartLspInlayHintSupport.shouldAskServerForInlayHints` — it runs before
   every inlay-hint request cycle; compare the current effective snapshot with the last-pushed
   one and send `workspace/didChangeConfiguration` (empty `settings`, the server re-pulls — F6)
   on drift.
3. After a push, invalidate cached hints — the server does NOT send
   `workspace/inlayHint/refresh` (verified, F10), and `LspHighlightingCache` only re-requests on
   PSI modification count changes (F11). Candidates **[verify at execution, pick ONE]**:
   (a) call the vendored cache's `clearCache()` — legal Kotlin-wise because `thirdPartySrc` and
   `src` share one sourceSet/module (F12), but adds an `impl` dependency the plugin code
   currently avoids (zero `platform.dartlsp.impl` imports in `src/main`, F12) — if chosen,
   document as deliberate; (b) add a small public invalidation hook to the vendored
   `LspInlayHintCustomizer`/cache API (vendored-code change → upstream-candidate note per
   Global Constraint 3); (c) reject silently-stale display and force a daemon restart via
   `DaemonCodeAnalyzer.getInstance(project).restart()` — **[verify]** that this alone re-runs the
   pass AND that the pass re-requests despite an unchanged PSI mod count (F11 suggests it does
   NOT — expect (c) to be insufficient alone).

- [ ] Steps: failing drift-test (change a setting → next `shouldAskServerForInlayHints` triggers
  exactly one `workspace/didChangeConfiguration` through the bridge) → fail → implement →
  green → sandbox-verify toggling WITHOUT editing the file (the cache-invalidation candidate
  choice is validated here) → commit.

### Task 8: Version gate + docs + PR

**Files:**
- Modify: `DartAnalysisServerService.java` (`MIN_LSP_INLAY_HINTS_CONFIG_SDK_VERSION` from
  Task 3; only push config / answer `workspace/configuration` with `inlayHints` when the SDK is
  new enough — below the floor the options stay visible but ineffective; put that limitation in
  the option `descriptionKey`s and the CHANGELOG. Confirm this UX with helin24 — the alternative,
  hiding options dynamically, is not supported by the static EP XML, F9.)
- Modify: `third_party/CHANGELOG.md` (`## Unreleased`, `(#PR)` placeholder)
- [ ] Sandbox verification (workflow rule!): matrix over {old SDK, new SDK} ×
  {parent on/off, each option} incl. restart-free toggling; then commit, push, PR referencing
  the Stage-1 PR #617 discussion; file the follow-up issue closure.

---

## Verified facts (evidence)

* **F1** `lspClientConfiguration` is an abstract getter on the base `AnalysisServer`
  (`pkg/analysis_server/lib/src/analysis_server.dart:516`); the legacy server owns a real
  instance (`legacy_analysis_server.dart:318,407`), the LSP server likewise
  (`lsp_analysis_server.dart:71,160`). Only the LSP server ever calls `.replace(...)` today
  (`lsp_analysis_server.dart:379` — the single hit in `lib/src`).
* **F2** The shared `InlayHintHandler` (in `sharedHandlerGenerators`,
  `handler_states.dart:147`) already reads
  `server.lspClientConfiguration.global.inlayHints` and passes it to `DartInlayHintComputer`
  (`handler_inlay_hint.dart:60-64`); the computer drops disabled categories in `_addHint`
  (`computer_inlay_hint.dart:182-196`).
* **F3** Config schema + defaults: `LspClientInlayHintsConfiguration`
  (`client_configuration.dart:195-266`): keys `dotShorthandTypes, parameterNames,
  parameterTypes, returnTypes, typeArguments, variableTypes`; value `bool` or
  `{"enabled": bool}`, `parameterNames` additionally `"none"|"literal"|"all"`; defaults: all
  types `true`, `parameterNames = all`. It is global-only (`LspGlobalClientConfiguration`
  doc: workspace-level only, `client_configuration.dart:270+`). Server categories
  (`_InlayHintKind`, `computer_inlay_hint.dart:542`): parameterNameLiteral,
  parameterNameNonLiteral, parameterType, returnType, typeArgument, variableType,
  dotShorthandType. On the wire everything is `kind: Type` except parameter *names*
  (`kind: Parameter`) — incl. parameterType hints (`_addTypePrefix` → `InlayHintKind.Type`).
* **F4** Legacy `server.setClientCapabilities` stores `lspCapabilities` into
  `_editorClientCapabilities` (`legacy_analysis_server.dart:498-520`); spec:
  `tool/spec/spec_input.html:396-455` (fields `requests`, `supportsUris`, `lspCapabilities`;
  line 155: capabilities "can indicate that a client can support lsp.handle requests in the
  server-to-client direction").
* **F5** `LspOverLegacyHandler.handle()` parses **only** `RequestMessage`
  (`RequestMessage.canParse`) and answers `INVALID_PARAMETER` otherwise
  (`lsp_over_legacy_handler.dart:29-60`) — LSP notifications (no `id`) are rejected today.
* **F6** `fetchClientConfigurationAndPerformDynamicRegistration()` is LSP-server-only
  (`lsp_analysis_server.dart:334-399`): gate `editorClientCapabilities?.configuration ?? false`;
  sends `workspace/configuration` via `sendLspRequest`; folder items + global item; calls
  `lspClientConfiguration.replace`, then `_refreshAnalysisRoots()` / `reanalyze()`; finally
  LSP-only `capabilitiesComputer.performDynamicRegistration()`. The **legacy** server implements
  `sendLspRequest` (reverse `lsp.handle` legacy request, `legacy_analysis_server.dart:759-800`)
  and `sendLspNotification` (`lsp.notification`, `:745-756`).
* **F7** `fixedBasicLspClientCapabilities` (`lsp/client_capabilities.dart:13-22`) sets only
  hover markdown + workspaceEdit.documentChanges — no `workspace.configuration`, so the
  config-fetch gate stays closed until the client opts in.
* **F8** Plugin side: reverse `lsp.handle` requests are dispatched in
  `RemoteAnalysisServerImpl.processLspRequestFromServer` (`RemoteAnalysisServerImpl.java:902+`,
  currently only `workspace/applyEdit`); `buildLspCapabilities`
  (`DartAnalysisServerService.java:583-606`) currently advertises `workspace.applyEdit`
  (SDK ≥ 3.8) and `textDocument.definition.linkSupport`; called at
  `DartAnalysisServerService.java:2366`. `DartBridgeLspServer.forwardNotification`
  (`DartBridgeLspServer.kt:380-402`) wraps notifications in `lsp.handle` and has **no callers**
  today; its KDoc already anticipates the dummy-acknowledgment semantics of Task 1.
* **F9** IntelliJ 2026.1.3 (`intellij.platform.lang.jar` bytecode):
  `DeclarativeInlayHintsSettings` (app service) — `isProviderEnabled/setProviderEnabled/
  isOptionEnabled/setOptionEnabled`, `HintsState : BaseState` with no-arg ctor, extends
  `SimplePersistentStateComponent` (⇒ `loadState` reset in tests), fires **no** message-bus
  topic; `InlayProviderOption` — optionId/enabledByDefault/nameKey/descriptionKey/showInTree;
  `InlayHintsProviderFactory.getProviderInfo(Language, String): InlayProviderInfo?` with
  `isEnabledByDefault` and `getOptions`. EP XML is static (no dynamic hiding of options).
* **F10** The Dart server never sends `workspace/inlayHint/refresh` (zero hits for
  `inlayHint.?refresh` in `pkg/analysis_server/lib`).
* **F11** Vendored client cache: `LspInlayHintsCache.isSupportedForFile` consults
  `shouldAskServerForInlayHints` per attempt; `LspHighlightingCache.getHighlightings` only
  schedules a new server request when the PSI modification count changed
  (`LspHighlightingCache.kt:35-45`) — a config-only change does not bump it.
* **F12** `thirdPartySrc/analysisServer` and `thirdPartySrc/platform-lsp/src` are srcDirs of the
  same Gradle sourceSet as `src/main` (`third_party/build.gradle.kts:134-142`) ⇒ Kotlin
  `internal` of platform-lsp is technically accessible; `src/main` currently has zero
  `com.intellij.platform.dartlsp.impl` imports (deliberate layering).
* **F13** Stage 1 (PR #617): server always sets `InlayHint.kind` (F3), the client filter lives in
  `DartLspInlayHintSupport`, provider IDs `dart.parameter.names` / `dart.types`, defaults OFF.
