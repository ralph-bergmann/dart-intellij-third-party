# Design: LSP Inlay Hints, Read/Write Highlighting, Usage Count (Dart IntelliJ Plugin)

**Date:** 2026-07-30
**Status:** Approved (pending final spec review)
**Branch:** `DartInlayHints`

## 1. Goal

Bring three IDE features known from Java/Kotlin to the Dart IntelliJ plugin:

1. **More inlay hints** (types, parameter names, return types, type arguments) — today the plugin
   only shows closing labels.
2. **Usage counts** above declarations ("n usages", IntelliJ "Code Vision").
3. **Read vs. write occurrence highlighting** when the caret is on a variable.

Constraints set by the maintainers: follow the repository's DAS→LSP migration strategy (features go
through the `DartBridgeLspServer` / `lsp.handle` tunnel, gated by the *"Turn on experimental LSP
features"* setting), keep the diff small, and reuse existing code.

## 2. Approved decisions

| # | Decision |
|---|----------|
| D1 | Inlay hints are implemented **LSP-first**: a small upstream Dart SDK change makes `textDocument/inlayHint` reachable over the legacy protocol; the plugin only enables the already-bundled LSP client feature. No IDE-side hint computation. |
| D2 | All hint categories the server produces are enabled (server defaults). No per-category IDE settings UI in this iteration. |
| D3 | Usage count is **design-only** in this iteration because the Dart LSP server has no references code lens. Follow-up options are documented (section 6), not implemented. |
| D4 | Read/write highlighting is implemented **twice, complementary**: (a) LSP `textDocument/documentHighlight` via the bridge (experimental-flag users), and (b) a PSI-based `DartReadWriteAccessDetector` that works for all users immediately and additionally classifies Find Usages results. |
| D5 | `DartInlayHintsProvider` (closing labels only) is renamed to `DartClosingLabelsInlayHintsProvider`. The `providerId` `"dart.closing.labels"` is kept, so persisted user settings are unaffected. |

## 3. Verified background (why this design is possible)

Facts verified against `dart-lang/sdk` `main` and this repository on 2026-07-30:

* The plugin runs DAS in legacy-protocol mode and tunnels LSP requests through the custom
  `lsp.handle` request (`DartBridgeLspServer`). Only SDK handlers registered as
  `SharedMessageHandler` (in `InitializedStateMessageHandler.sharedHandlerGenerators`,
  `pkg/analysis_server/lib/src/lsp/handlers/handler_states.dart`) are reachable this way.
* `textDocument/documentHighlight`, `textDocument/references`, `textDocument/codeLens` **are**
  shared handlers. `textDocument/inlayHint` is **not** (it is in
  `InitializedLspStateMessageHandler.lspHandlerGenerators`); calling it over `lsp.handle` returns
  `MethodNotFound`. There is no `inlayHint/resolve` handler at all.
* The SDK inlay hint computer (`pkg/analysis_server/lib/src/computer/computer_inlay_hint.dart`)
  produces LSP `InlayHintKind.Type` and `InlayHintKind.Parameter` hints for: variable types,
  parameter names, parameter types, return types, type arguments, and dot-shorthand types.
  Per-category configuration exists server-side (`dart.inlayHints`), but it is delivered via
  `workspace/didChangeConfiguration`/`workspace/configuration`, which is LSP-only — over the legacy
  protocol the server uses its defaults (all categories enabled).
* The SDK document highlights computer (`computer_document_highlights.dart`) sets
  `DocumentHighlightKind.Read/Write/Text` on current `main`. Older SDKs return highlights without
  kinds; the IntelliJ side then renders everything with the read color, which equals today's
  behavior (graceful degradation).
* The SDK code lens handler only provides augmentation navigation lenses — **no usage counts**.
* The copied JetBrains LSP client (`third_party/thirdPartySrc/platform-lsp`, namespace
  `com.intellij.platform.dartlsp`) already registers everything needed in `dart-lsp-impl.xml`:
  `LspInlayHintsProviderFactory` (EP `codeInsight.inlayProviderFactory`), `LspCodeVisionProvider`
  (EP `codeInsight.codeVisionProvider`), and `LspHighlightUsagesHandlerFactory`
  (EP `highlightUsagesHandlerFactory`, `order="last"`), which maps `DocumentHighlightKind.Write` to
  the write color and everything else to the read color. The generic `LspInlayHintsProvider` is
  deliberately invisible in *Settings | Editor | Inlay Hints* (`isVisibleInSettings = false`).
* IntelliJ splits read/write occurrences via the `com.intellij.readWriteAccessDetector` EP
  (`IdentifierHighlightingComputer`). Dart registers no detector today, so all occurrences use the
  read color. The same detector also drives the read/write icons and filters in the Find Usages
  view (`UsageInfoToUsageConverter` → `ReadWriteUtil`). When an LSP server with documentHighlight
  support has the file open, the LSP handler replaces the PSI/detector path for caret highlighting;
  Find Usages classification always uses the detector.
* Dart PSI references resolve through DAS navigation regions (`DartResolver`), so the PSI-based
  identifier highlighting and a PSI-based detector work for files open in the editor.
* Third-party proof of the alternative IDE-side approach: the "Flutter Developer Tools" plugin
  (github.com/rjs580/flutter_developer_tools) implements type hints via legacy
  `analysis_getHover().staticType` per identifier, parameter hints via PSI, and usage counts via
  `ReferencesCodeVisionProvider` + `DartServerFindUsagesHandler`. We deliberately do not replicate
  this in the first-party plugin (duplicated server logic, per-identifier round trips).

## 4. Feature 1: LSP inlay hints

### 4.1 Dart SDK change (upstream, `dart-lang/sdk`)

Make the inlay hint handler shared so it is reachable over `lsp.handle`:

* `pkg/analysis_server/lib/src/lsp/handlers/handler_inlay_hint.dart`:
  `class InlayHintHandler extends LspMessageHandler<…>` → `extends SharedMessageHandler<…>`, plus
  `bool get requiresTrustedCaller => false;` (same pattern as `CodeLensHandler`,
  `DocumentHighlightsHandler`, `ReferencesHandler`).
* `pkg/analysis_server/lib/src/lsp/handlers/handler_states.dart`: move `InlayHintHandler.new` from
  `InitializedLspStateMessageHandler.lspHandlerGenerators` to
  `InitializedStateMessageHandler.sharedHandlerGenerators`.
* Add an LSP-over-legacy test in `pkg/analysis_server/test/lsp_over_legacy/` (the suite that covers
  the other shared handlers).
* Before filing: search dart-lang/sdk issues/CLs for existing work on sharing the inlay hint
  handler.

The handler reads `server.lspClientConfiguration`, which exists on the common `AnalysisServer` base
class (shared handlers such as `CodeLensHandler` already use it), so the conversion is expected to
be mechanical. Over legacy the configuration keeps its defaults: all hint categories enabled.

### 4.2 Plugin change (this repository)

Follow the exact pattern used for hover/definition:

* `DartAnalysisServerService`: add `MIN_LSP_INLAY_HINTS_SDK_VERSION` +
  `isDartSdkVersionSufficientForLspInlayHints(...)` + `isLspInlayHintsEnabled(project)` mirroring
  the `MIN_LSP_NAVIGATION_SDK_VERSION` trio. The version value is the first SDK dev version that
  contains the upstream change (known once the SDK CL lands).
* `LspMethod`: add `INLAY_HINT("textDocument/inlayHint", isExperimental = true, presentableName =
  "inlay hints")` — the settings checkbox label picks this up automatically.
* `DartLspServerDescriptor.lspCustomization`: `inlayHintCustomizer` getter returns
  `LspInlayHintSupport()` when `isLspInlayHintsEnabled(project)`, else `LspInlayHintDisabled`.
* `DartBridgeLspServer`:
  * `initialize()`: add `setInlayHintProvider(true)` (capabilities are advertised unconditionally,
    gating happens in the customizer — same as hover/definition today).
  * Override `inlayHint(params: InlayHintParams): CompletableFuture<List<InlayHint>>` forwarding
    `"textDocument/inlayHint"` via `forwardRequest` with a `TypeToken<List<InlayHint>>` (same shape
    as the existing `definition(...)` override).
* CHANGELOG entry under *Unreleased / Added* in the existing grammatical style, e.g.
  "Inlay hints (types, parameter names) with JetBrains LSP as an experimental feature (#PR)".

No new provider classes, no plugin.xml changes — rendering and caching are done by the bundled
`LspInlayHintsProviderFactory`/`LspInlayHintsCache`.

### 4.3 Behavior and documented trade-offs

* Visible only with *Turn on experimental LSP features* enabled **and** a sufficiently new SDK.
* Hint categories: server defaults (all on). There is **no per-category toggle** in
  *Settings | Editor | Inlay Hints* for these hints (the generic LSP provider is invisible there by
  platform design). Follow-ups if demand arises: client-side filtering by `InlayHintKind`
  (Type/Parameter granularity) via an `LspInlayHintSupport` subclass overriding
  `shouldDisplayInlayHint`, or a second SDK change sharing `workspace/didChangeConfiguration` to
  make `dart.inlayHints` configurable.
* Labels longer than the framework default of 42 chars are truncated
  (`LspInlayHintSupport.getMaxInlayHintChars`, can be tuned later).
* Closing labels are unaffected: they come from the DAS `CLOSING_LABELS` subscription and the
  (renamed) declarative provider; LSP inlay hints do not include closing labels, so no duplication.

### 4.4 Sequencing

The plugin-side change hard-depends on the SDK CL only for the version constant and for end-to-end
verification. Bridge/customizer code plus unit tests can be developed and reviewed in parallel; the
PR merges after the SDK CL lands and the dev version is known.

## 5. Feature 2: Read/write occurrence highlighting

Two independent, complementary parts. No SDK change required.

### 5.1 Part A — LSP `textDocument/documentHighlight` (experimental-flag users)

* `DartLspServerDescriptor.lspCustomization`: `documentHighlightsCustomizer` getter returns, when
  `DartConfigurable.isExperimentalLspFeaturesEnabled(project)`, an
  `object : LspDocumentHighlightsSupport() { override fun shouldAskServerForDocumentHighlights(psiFile: PsiFile) = true }`
  (the default implementation only serves plain-text/TextMate files), else
  `LspDocumentHighlightsDisabled`.
* `DartBridgeLspServer`: `initialize()` adds `setDocumentHighlightProvider(true)`; new override
  `documentHighlight(params: DocumentHighlightParams)` forwarding
  `"textDocument/documentHighlight"` (returns `List<DocumentHighlight>`).
* `LspMethod`: add `DOCUMENT_HIGHLIGHT("textDocument/documentHighlight", isExperimental = true,
  presentableName = "read/write highlighting")`.
* No SDK version gate: the method has long been shared; SDKs without highlight kinds degrade to
  today's all-read rendering. Bonus on new SDKs: `DocumentHighlightKind.Text` gives related-keyword
  highlighting (`break`/`continue`/loop keywords).
* CHANGELOG entry.

### 5.2 Part B — `DartReadWriteAccessDetector` (all users, plus Find Usages)

* New class `com.jetbrains.lang.dart.highlight.DartReadWriteAccessDetector` extending
  `com.intellij.codeInsight.highlighting.ReadWriteAccessDetector`, registered in plugin.xml via
  `<readWriteAccessDetector implementation="…"/>`.
* Semantics (PSI-based, using existing generated PSI):
  * `isReadWriteAccessible(element)`: `DartComponentName` (or an element resolving to one) whose
    component is variable-like: local/top-level variables and fields (`DartVarAccessDeclaration` /
    `DartVarDeclarationListPart`), formal parameters (`DartSimpleFormalParameter` etc.), for-in
    loop variables (`DartForInPart`), setter/getter pairs (`DartSetterDeclaration` /
    `DartGetterDeclaration`). Classes, methods, functions are excluded.
  * `isDeclarationWriteAccess(element)`: declaration carries an initializer (`DartVarInit`) or is a
    for-in loop variable.
  * `getReferenceAccess(referencedElement, reference)` delegates to
    `getExpressionAccess(reference.getElement())`.
  * `getExpressionAccess(expression)`: walk up from the reference expression —
    * rightmost reference of the LHS of a `DartAssignExpression`: `=` → `Write`; compound
      operators (`+=`, `-=`, `??=`, …, via `DartAssignmentOperator`) → `ReadWrite`;
    * operand of `++`/`--` in `DartPrefixExpression`/`DartSuffixExpression` → `ReadWrite`;
    * LHS of a `DartPatternAssignment` → `Write`;
    * otherwise `Read`. Qualifiers stay reads (`a` in `a.b = x`).
* Effect: read/write colors for caret highlighting for **all** users (PSI path), read/write
  icons + filters in the Find Usages view (all modes — the LSP path cannot provide this). When
  Part A is active, the platform automatically prefers the LSP handler for caret highlighting; the
  detector keeps serving Find Usages.
* CHANGELOG entry.

## 6. Feature 3: Usage count (design-only, not implemented now)

IntelliJ's "n usages" is Code Vision (rendered as an inlay, configured under *Settings | Editor |
Inlay Hints | Code vision*). The Dart LSP server currently provides no references code lens, so per
decision D3 nothing is implemented in this iteration. Documented follow-up options, in order of
strategic preference:

1. **Upstream (preferred):** add a references `CodeLensProvider` to the SDK's `CodeLensHandler`
   (config via the existing `dart.codeLens` client configuration). Benefits every LSP client
   (VS Code included). IntelliJ side afterwards: enable `LspCodeLensSupport` in the descriptor,
   forward `textDocument/codeLens` in the bridge, and implement
   `codeLensClicked` (show references popup via `Lsp4jService.extractLocationsFromJson` +
   `navigateOrShowPopup`). Note: `codeLens/resolve` is not implemented by the server, so lenses
   must arrive with resolved commands (the augmentation lenses already do).
2. **IDE-side:** `DartReferencesCodeVisionProvider extends
   com.intellij.codeInsight.hints.codeVision.ReferencesCodeVisionProvider`, registered via
   `codeInsight.daemonBoundCodeVisionProvider`, counting through the existing
   `DartServerFindUsagesHandler` (legacy `search.findElementReferences`) with a result cap. Proven
   viable by the "Flutter Developer Tools" plugin; cost: one server search per visible declaration,
   needs careful capping/caching.

Enabling `LspCodeLensSupport` today would only surface the server's augmentation lenses and is out
of scope.

## 7. Rename

`com.jetbrains.lang.dart.hints.DartInlayHintsProvider` →
`com.jetbrains.lang.dart.hints.DartClosingLabelsInlayHintsProvider` (file rename included).
Touch points: the class itself, `plugin.xml` (`implementationClass` of the
`codeInsight.declarativeInlayProvider` registration), and `DartClosingLabelManager` (import +
`PROVIDER_ID` references). `PROVIDER_ID` stays `"dart.closing.labels"`; registration attributes
(`nameKey`, `group`, …) are unchanged, so the *Settings | Editor | Inlay Hints | Other | Dart |
Closing Labels* entry and persisted user settings are untouched.

## 8. Testing

* **Bridge unit tests** (extend `third_party/src/test/java/com/jetbrains/lang/dart/lsp/DartBridgeLspServerTest.kt`,
  which already stubs the DAS connection): request forwarding + response unwrapping for
  `textDocument/inlayHint` and `textDocument/documentHighlight`, modeled after
  `testDiagnosticServerRequest`/`testForwardRequest`.
* **Detector unit tests**: new `DartReadWriteAccessDetectorTest` (fixture-based, à la existing
  `DartCodeInsightFixtureTestCase` tests) asserting `Access.Read/Write/ReadWrite` for: plain read,
  `=` assignment, compound assignment, `++`/`--` (pre/post), declaration with/without initializer,
  for-in variable, qualified LHS (`a.b = x`), pattern assignment.
* **Settings**: `DartLspExperimentalFeaturesTest` stays green; new `LspMethod` entries appear in
  the checkbox label.
* **Manual sandbox verification** (per the migrate-das-to-lsp skill): `./gradlew clean
  prepareSandbox --no-build-cache`; verify on files open at startup and opened later, external
  files (`.pub-cache`, `dart:io`), flag-toggle lifecycle; `./gradlew verifyPlugin` (update
  baselines via `third_party/tool/update_baselines.sh` if needed).
* **SDK side**: LSP-over-legacy inlay hint test in the SDK CL (section 4.1).
* End-to-end inlay hint verification requires an SDK build containing the upstream change (dev SDK
  once the CL lands).

## 9. Out of scope

* Semantic-tokens-based highlighting (not available over legacy; Dart emits no `modification`
  modifier anyway).
* Per-category inlay hint settings UI (documented as follow-up in 4.3).
* Usage count implementation (section 6).
* Any modification of files under `third_party/thirdPartySrc/` (copied sources; changes would have
  to go through `.agents/skills/patch-copied-lsp-sources/scripts/patch.py` — none are needed for
  this design).
