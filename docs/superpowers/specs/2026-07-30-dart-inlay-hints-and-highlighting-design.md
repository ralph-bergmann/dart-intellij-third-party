# Design: LSP Inlay Hints, Read/Write Highlighting, Usage Count (Dart IntelliJ Plugin)

**Date:** 2026-07-30 (updated after ecosystem research, same day)
**Status:** Approved (pending final spec review)
**Branch:** `DartInlayHints`
**Related issues:** flutter/dart-intellij-third-party#159, #207, #400, #546, #92 ·
dart-lang/sdk#63884 (precedent), #62929 · Dart-Code/Dart-Code#1605

## 1. Goal

Bring three IDE features known from Java/Kotlin to the Dart IntelliJ plugin:

1. **More inlay hints** (types, parameter names, return types, type arguments) — today the plugin
   only shows closing labels. Requested in flutter/dart-intellij-third-party#159 (open, unclaimed).
2. **Usage counts** above declarations ("n usages", IntelliJ "Code Vision").
3. **Read vs. write occurrence highlighting** when the caret is on a variable.

Constraints set by the maintainers' migration strategy (tracking issue #207 and the in-repo
`migrate-das-to-lsp` skill): features go through the `DartBridgeLspServer` / `lsp.handle` tunnel,
gated by the *"Turn on experimental LSP features"* setting plus SDK version checks, with the legacy
implementation as fallback. Keep the diff small and reuse existing code. PSI-based machinery is
being removed as LSP endpoints become available (PR #539 removed `DartReferenceContributor`,
issue #546 plans to replace `DartResolver`), so new PSI-based features are avoided.

## 2. Approved decisions

| # | Decision |
|---|----------|
| D1 | Inlay hints are implemented **LSP-first**: a small upstream Dart SDK change makes `textDocument/inlayHint` reachable over the legacy protocol; the plugin only enables the already-bundled LSP client feature. No IDE-side hint computation. |
| D2 | All hint categories the server produces are enabled (server defaults). No per-category IDE settings UI in this iteration; IntelliJ does not auto-generate settings checkboxes for LSP-provided hints (the settings tree is fed by declarative providers only). Follow-up options in 4.3. |
| D3 | Usage count is **design-only** in this iteration because the Dart LSP server has no references code lens. Follow-up options are documented (section 6), not implemented. |
| D4 | Read/write highlighting is implemented via **LSP `textDocument/documentHighlight` only**, aligning with issue #546 (which lists this endpoint as required for the `DartResolver` replacement). A PSI-based `DartReadWriteAccessDetector` was considered and **deliberately not implemented**: it would build on the PSI resolve infrastructure that the maintainers are phasing out. It is documented as a follow-up proposal to be raised as a plugin-repo issue (section 5.3). |
| D5 | `DartInlayHintsProvider` (closing labels only) is renamed to `DartClosingLabelsInlayHintsProvider`. The `providerId` `"dart.closing.labels"` is kept, so persisted user settings are unaffected. (Related: #400 tracks migrating closing labels themselves to LSP later.) |

## 3. Verified background (why this design is possible)

Facts verified on 2026-07-30 against `dart-lang/sdk` `main` (GitHub and the local checkout at
`../sdk`, version 3.14.0-dev), this repository, and both issue trackers:

* The plugin runs DAS in legacy-protocol mode and tunnels LSP requests through the custom
  `lsp.handle` request (`DartBridgeLspServer`). Only SDK handlers registered as
  `SharedMessageHandler` (in `InitializedStateMessageHandler.sharedHandlerGenerators`,
  `pkg/analysis_server/lib/src/lsp/handlers/handler_states.dart`) are reachable this way.
* `textDocument/documentHighlight`, `textDocument/references`, `textDocument/codeLens` **are**
  shared handlers. `textDocument/inlayHint` is **not** (it is in
  `InitializedLspStateMessageHandler.lspHandlerGenerators`); calling it over `lsp.handle` returns
  `MethodNotFound`. There is no `inlayHint/resolve` handler at all. No SDK issue or Gerrit CL to
  share the inlay hint handler exists yet — it has to be filed (see 4.1).
* **Precedent for the SDK change:** dart-lang/sdk#63884 ("Make textDocument/definition and
  textDocument/references shared handlers…", filed by DanTup with the rationale "This would allow
  IntelliJ to use them via LoL while being migrated to LSP") was fixed the same day by Gerrit CL
  527540: 6 files, +121/−5, "a trivial change of types (plus some basic tests) with no
  implementation changes". Verified in the local checkout: all of `InlayHintHandler`'s dependencies
  (`requireResolvedUnit`, `extractDocumentVersion`, `fileHasBeenModified` in the generic
  `HandlerHelperMixin<S extends AnalysisServer>`; abstract `lspClientConfiguration` on the base
  `AnalysisServer`) are already server-agnostic — the conversion has no blockers.
* The SDK inlay hint computer (`pkg/analysis_server/lib/src/computer/computer_inlay_hint.dart`)
  produces LSP `InlayHintKind.Type` and `InlayHintKind.Parameter` hints for: variable types,
  parameter names, parameter types, return types, type arguments, and dot-shorthand types.
  Per-category configuration exists server-side (`dart.inlayHints`), but it is delivered via
  `workspace/didChangeConfiguration`/`workspace/configuration`, which is LSP-only — over the legacy
  protocol the server uses its defaults (all categories enabled). (The legacy
  `server.setClientCapabilities` request does carry LSP client *capabilities* via its
  `lspCapabilities` field — the plugin already uses it — but not *configuration*.)
* The SDK document highlights computer sets `DocumentHighlightKind.Read/Write/Text` since
  dart-lang/sdk#62929 (closed 2026-05). Older SDKs return highlights without kinds; the IntelliJ
  side then renders everything with the read color, which equals today's behavior (graceful
  degradation). Bonus: the LSP computer handles object patterns, likely fixing #92 in LSP mode.
* The SDK code lens handler only provides augmentation navigation lenses — **no usage counts**.
  No SDK issue requests a references code lens; the closest prior art is the VS Code request
  Dart-Code/Dart-Code#1605 (open since 2019).
* The copied JetBrains LSP client (`third_party/thirdPartySrc/platform-lsp`, namespace
  `com.intellij.platform.dartlsp`) already registers everything needed in `dart-lsp-impl.xml`:
  `LspInlayHintsProviderFactory` (EP `codeInsight.inlayProviderFactory`), `LspCodeVisionProvider`
  (EP `codeInsight.codeVisionProvider`), and `LspHighlightUsagesHandlerFactory`
  (EP `highlightUsagesHandlerFactory`, `order="last"`), which maps `DocumentHighlightKind.Write` to
  the write color and everything else to the read color. The generic `LspInlayHintsProvider` is
  deliberately invisible in *Settings | Editor | Inlay Hints* (`isVisibleInSettings = false`);
  the checkbox tree there is fed exclusively by declarative providers.
* IntelliJ splits read/write occurrences via the `com.intellij.readWriteAccessDetector` EP
  (`IdentifierHighlightingComputer`). Dart registers no detector today, so all occurrences use the
  read color. When an LSP server with documentHighlight support has the file open, the LSP handler
  replaces the PSI path for caret highlighting. The same detector EP also drives read/write
  icons/filters in the Find Usages view — an LSP documentHighlight cannot provide that part
  (see follow-up 5.3).
* Third-party proof of the alternative IDE-side approach: the "Flutter Developer Tools" plugin
  (github.com/rjs580/flutter_developer_tools) implements type hints via legacy
  `analysis_getHover().staticType` per identifier, parameter hints via PSI, and usage counts via
  `ReferencesCodeVisionProvider` + `DartServerFindUsagesHandler`. We deliberately do not replicate
  this in the first-party plugin (duplicated server logic, per-identifier round trips, against the
  migration direction).

## 4. Feature 1: LSP inlay hints

### 4.1 Dart SDK change (upstream, `dart-lang/sdk`)

Make the inlay hint handler shared so it is reachable over `lsp.handle`:

* File a dart-lang/sdk issue mirroring #63884 ("Make textDocument/inlayHint a shared handler so
  it's available over DTD/Legacy — allows IntelliJ to show inlay hints while being migrated to
  LSP"). Verified 2026-07-30 that no such issue or CL exists yet.
* `pkg/analysis_server/lib/src/lsp/handlers/handler_inlay_hint.dart`:
  `class InlayHintHandler extends LspMessageHandler<…>` → `extends SharedMessageHandler<…>`, plus
  `bool get requiresTrustedCaller => false;` (same pattern as `CodeLensHandler`,
  `DocumentHighlightsHandler`, and CL 527540).
* `pkg/analysis_server/lib/src/lsp/handlers/handler_states.dart`: move `InlayHintHandler.new` from
  `InitializedLspStateMessageHandler.lspHandlerGenerators` to
  `InitializedStateMessageHandler.sharedHandlerGenerators`.
* Add `pkg/analysis_server/test/lsp_over_legacy/inlay_hint_test.dart` (mirroring
  `document_highlights_test.dart` / the tests added in CL 527540) and register it in `test_all.dart`.
* Estimated size, verified against the local `../sdk` checkout: ~60–80 lines including the test.

The handler reads `server.lspClientConfiguration`, which exists on the common `AnalysisServer` base
class (shared handlers such as `CodeLensHandler` already use it), so the conversion is mechanical.
Over legacy the configuration keeps its defaults: all hint categories enabled.

### 4.2 Plugin change (this repository)

Follow the exact pattern used for hover/definition; link the PR to issue #159:

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
  *Settings | Editor | Inlay Hints* for these hints, and IntelliJ does not generate one (the tree
  is populated by declarative providers only; the LSP provider is invisible there by platform
  design). Follow-ups if demand arises: client-side filtering by `InlayHintKind`
  (Type/Parameter granularity only) via an `LspInlayHintSupport` subclass overriding
  `shouldDisplayInlayHint` backed by Dart-settings checkboxes, or a second SDK change providing a
  legacy path into `LspClientConfiguration` to make `dart.inlayHints` fully configurable.
* Labels longer than the framework default of 42 chars are truncated
  (`LspInlayHintSupport.getMaxInlayHintChars`, can be tuned later).
* Closing labels are unaffected: they come from the DAS `CLOSING_LABELS` subscription and the
  (renamed) declarative provider; LSP inlay hints do not include closing labels, so no duplication.
  (#400 tracks a possible later migration of closing labels themselves.)

### 4.4 Sequencing

The plugin-side change hard-depends on the SDK CL only for the version constant and for end-to-end
verification. Bridge/customizer code plus unit tests can be developed and reviewed in parallel; the
PR merges after the SDK CL lands and the dev version is known.

## 5. Feature 2: Read/write occurrence highlighting (LSP only)

### 5.1 Plugin change

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
* No SDK version gate: the method has long been shared; SDKs without highlight kinds (pre-#62929)
  degrade to today's all-read rendering. Bonus on new SDKs: `DocumentHighlightKind.Text` gives
  related-keyword highlighting (`break`/`continue`/loop keywords), and object patterns work (#92).
* CHANGELOG entry.
* Reference #546 in the PR (this delivers one of the endpoints it lists).

### 5.2 Behavior

With the experimental flag on and the file open in an LSP-served editor, the platform prefers the
LSP highlight handler over the PSI path: writes get the "Write identifier under caret" color, reads
the normal one. Without the flag, behavior is unchanged (all-read PSI highlighting).

### 5.3 Follow-up (not implemented): PSI `DartReadWriteAccessDetector`

A ~100-line `ReadWriteAccessDetector` for Dart PSI would additionally provide (a) read/write caret
colors for users without the experimental flag and (b) read/write icons + filters in the Find
Usages view (which the LSP path cannot feed). It was deliberately dropped from this iteration
because it builds on the PSI resolve infrastructure the maintainers are phasing out (#546). If the
Find-Usages classification is wanted, propose it first as a flutter/dart-intellij-third-party issue
and let the maintainers decide whether a legacy-path shim is acceptable.

## 6. Feature 3: Usage count (design-only, not implemented now)

IntelliJ's "n usages" is Code Vision (rendered as an inlay, configured under *Settings | Editor |
Inlay Hints | Code vision*). The Dart LSP server currently provides no references code lens, so per
decision D3 nothing is implemented in this iteration. Documented follow-up options, in order of
strategic preference:

1. **Upstream (preferred):** add a references `CodeLensProvider` to the SDK's `CodeLensHandler`
   (config via the existing `dart.codeLens` client configuration). Benefits every LSP client —
   VS Code has requested exactly this since 2019 (Dart-Code/Dart-Code#1605). IntelliJ side
   afterwards: enable `LspCodeLensSupport` in the descriptor, forward `textDocument/codeLens` in
   the bridge, and implement `codeLensClicked` (show references popup via
   `Lsp4jService.extractLocationsFromJson` + `navigateOrShowPopup`). Note: `codeLens/resolve` is
   not implemented by the server, so lenses must arrive with resolved commands (the augmentation
   lenses already do).
2. **IDE-side:** `DartReferencesCodeVisionProvider extends
   com.intellij.codeInsight.hints.codeVision.ReferencesCodeVisionProvider`, registered via
   `codeInsight.daemonBoundCodeVisionProvider`, counting through the existing
   `DartServerFindUsagesHandler` (legacy `search.findElementReferences`) with a result cap. Proven
   viable by the "Flutter Developer Tools" plugin; cost: one server search per visible declaration,
   and it deepens reliance on the legacy path the maintainers are leaving.

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
* **Settings**: `DartLspExperimentalFeaturesTest` stays green; new `LspMethod` entries appear in
  the checkbox label.
* **Manual sandbox verification** (per the migrate-das-to-lsp skill): `./gradlew clean
  prepareSandbox --no-build-cache`; verify on files open at startup and opened later, external
  files (`.pub-cache`, `dart:io`), flag-toggle lifecycle; `./gradlew verifyPlugin` (update
  baselines via `third_party/tool/update_baselines.sh` if needed). For read/write highlighting:
  caret on a variable that is read, assigned (`=`), compound-assigned (`+=`), incremented — writes
  must use the write color.
* **SDK side**: LSP-over-legacy inlay hint test in the SDK CL (section 4.1).
* End-to-end inlay hint verification requires an SDK build containing the upstream change (dev SDK
  once the CL lands; a locally built SDK from `../sdk` with the patch applied works too).

## 9. Out of scope

* PSI-based `DartReadWriteAccessDetector` (follow-up proposal, section 5.3).
* Semantic-tokens-based highlighting (not available over legacy; Dart emits no `modification`
  modifier anyway).
* Per-category inlay hint settings UI (documented as follow-up in 4.3).
* Usage count implementation (section 6).
* Any modification of files under `third_party/thirdPartySrc/` (copied sources; changes would have
  to go through `.agents/skills/patch-copied-lsp-sources/scripts/patch.py` — none are needed for
  this design).
