# LSP migration scope analysis — what to take on from tracking issue #207

**Date:** 2026-08-18
**Status:** Analysis + decisions (approved by Ralph pending review of this file)
**Branch:** `DartInlayHints` (planning docs only)
**Supersedes/extends:** `2026-07-30-dart-inlay-hints-and-highlighting-design.md` (still valid for
inlay hints, read/write highlighting and the rename; usage count decision D3 is revisited in §7)

## 1. Why this document exists

The Dart SDK change that Part 3 of the inlay-hints plan depended on has landed
(dart-lang/sdk commit `7c18d1fa0e5`, [CL 536565](https://dart-review.googlesource.com/c/sdk/+/536565),
closes [dart-lang/sdk#64061](https://github.com/dart-lang/sdk/issues/64061), 2026-08-17). That
unblocks the last planned PR. Before continuing, Ralph widened the scope: instead of only shipping the
features that replace the abandoned *Flutter Enhancement Suite* (inlay hints, usage counts), Ralph wants to
contribute more of the maintainers' LSP migration
([tracking issue #207](https://github.com/flutter/dart-intellij-third-party/issues/207)) — but only
work that is well-defined, and grouped so that similar changes arrive as *stacked* PRs (the repo
allows him two open PRs at a time; reviewers should see the same pattern once, not five times).

Ground rules for everything below: every claim is validated against the issue trackers or the real
code (plugin repo, vendored JetBrains LSP client, `dart-lang/sdk` at `origin/main` = `00c42422917`,
IntelliJ Community sources), nothing is guessed. Evidence is collected in the appendix (§10).

## 2. State of the world (verified 2026-08-18)

### 2.1 Dart SDK (analysis server, `origin/main`)

Reachable over LSP-over-Legacy (`lsp.handle`) = registered in
`InitializedStateMessageHandler.sharedHandlerGenerators`
(`pkg/analysis_server/lib/src/lsp/handlers/handler_states.dart`):

| Handler | Shared since | Relevant to |
|---|---|---|
| `InlayHintHandler` | **`7c18d1fa0e5`, first dev tag `3.14.0-139.0.dev`** | #159 (Ralph's Part 3) |
| `DefinitionHandler`, `ReferencesHandler` | `c6d728bfd33` → `3.14.0-65.0.dev` (#63884) | #398 ✅ done, #396 |
| `TypeDefinitionHandler`, `ImplementationHandler`, `SuperHandler`, `DocumentSymbolHandler`, `Prepare/…TypeHierarchy`, `Prepare/…CallHierarchy`, `HoverHandler`, `DocumentHighlightsHandler`, `SignatureHelpHandler`, `WorkspaceSymbolHandler`, `Formatting*`, `DocumentColor*` | already shared in Dart 3.3.0 (`SuperHandler` in 3.5.0) | #580, #404, #402, #403, … |
| `CodeLensHandler` (augmentation lenses only) | 3.5.0 | usage count (§7) |
| `CodeActionHandler`, `ExecuteCommandHandler`, `CommandResolveHandler` | 2025-05-13 / 2025-04-30 / 2026-06-03 (blame) | #520 (helin24) |
| `DiagnosticServerHandler` | `eb362d426` → `3.13.0-106.0.dev` | done (#493) |

**LSP-only** (calling them over `lsp.handle` returns `MethodNotFound`; would need an SDK change like
#63884/#64061 first): `CompletionHandler`/`CompletionResolveHandler` (#399), `PrepareRenameHandler`/
`RenameHandler` (#407), `SemanticTokensFull/RangeHandler` (#401), `FoldingHandler`,
`SelectionRangeHandler`, `DocumentLinkHandler`, `WorkspaceDidChangeConfigurationMessageHandler`,
`TextDocumentOpen/Change/CloseHandler`.

**Push notifications** the LSP server sends only in pure-LSP mode (`LspAnalysisServer`, gated by
`initializationOptions`): `dart/textDocument/publishClosingLabels`, `dart/textDocument/publishOutline`,
`dart/textDocument/publishFlutterOutline`. The legacy server has no equivalent LSP notification path
for them yet. Precedent for adding one: `d42063aac44` (2026-08-17, DanTup, sdk#64021) — sends LSP
`textDocument/publishDiagnostics` to LoL clients that advertise the `publishDiagnostics` client
capability, via `LegacyAnalysisServer.sendLspNotification` (arrives as legacy `lsp.notification`).

**Server options are LSP-only too** (`pkg/analysis_server/tool/lsp_spec/README.md`): the
*Initialization Options* (`onlyAnalyzeProjectsWithOpenFiles`, `suggestFromUnimportedLibraries`,
`closingLabels`, `outline`, `flutterOutline`, `allowOpenUri`) travel with `initialize`, and the
*Client Workspace Configuration* (`dart.*`: `analysisExcludedFolders`, `enableSdkFormatter`,
`lineLength` (deprecated), `completeFunctionCalls`, `showTodos`, `renameFilesWithClasses`,
`enableSnippets`, `updateImportsOnRename`, `documentation`, `includeDependenciesInWorkspaceSymbols`,
`inlayHints` with six per-category switches) is pulled via `workspace/configuration` /
`workspace/didChangeConfiguration` — neither exists over LoL, so the legacy server runs on the
defaults of `LspInitializationOptions`/`LspClientConfiguration` (e.g. all inlay-hint categories on).
How a LoL client could pass them, and where their IntelliJ settings UI would live, is question Q13.

**No LSP counterpart at all** (legacy-only protocol): postfix templates (`edit.getPostfixCompletion`,
`edit.listPostfixCompletionTemplates`, #405) and statement completion
(`edit.getStatementCompletion`, #406) — `git grep -i 'postfix\|statementCompletion'` over
`pkg/analysis_server/lib/src/lsp` is empty.

### 2.2 Plugin (`flutter/dart-intellij-third-party` main)

Enabled through the bridge today (`DartLspServerDescriptor.lspCustomization`, all behind *Turn on
experimental LSP features*): hover, definition (`isLspNavigationEnabled`, SDK ≥ 3.14.0-65.0.dev),
documentHighlight (#552), `dart/diagnosticServer` (#493). Everything else is `…Disabled`.

**In flight by helin24 (draft PRs, do not overlap):**
[#526](https://github.com/flutter/dart-intellij-third-party/pull/526) *Implement LSP code actions*
(#520; touches `DartBridgeLspServer`, `DartLspServerDescriptor`, `DartConfigurable`, `patch.py`),
[#612](https://github.com/flutter/dart-intellij-third-party/pull/612) *Use publishDiagnostics to show
errors and warnings* (new `DartLspDiagnosticConverter`; #441 context-message navigation belongs here),
[#614](https://github.com/flutter/dart-intellij-third-party/pull/614) *Update analysis server API for
setClientCapabilities* (moves the `lspCapabilities` JSON into
`DartAnalysisServerService.buildLspCapabilities(sdkVersion)`).

The vendored JetBrains LSP client (`third_party/thirdPartySrc/platform-lsp`, namespace
`com.intellij.platform.dartlsp`, descriptor `resources/dart-lsp-impl.xml`) provides ready-made
features for: hover, definition, **typeDefinition** (`LspImplicitReferenceProvider` +
`LspSymbolTypeProvider`), documentHighlight, **inlay hints** (`LspInlayHintsProviderFactory`),
code lens (`LspCodeVisionProvider`), completion, diagnostics, code actions/intentions, formatting,
import optimizer, on-type formatting, folding, selection range, document links, document color,
find usages (`LspUsageSearcher` + `LspSearchTargetsRule`), rename, structure view helper
(`LspStructureViewSupport` — a helper only, no `PsiStructureViewFactory` registration) + breadcrumbs,
workspace symbols (Go to Class/Symbol), signature help (parameter info), call + type hierarchy
(`language=""`, `order="last"`), semantic tokens. **Not provided:** `textDocument/implementation`
(no customizer, no EP), `dart/textDocument/super`, custom notifications such as
`publishClosingLabels`.

### 2.3 Platform facts that decide how "clean" a migration can be

Verified in IntelliJ Community sources (`platform/lang-impl/src/com/intellij/model/psi/impl/{targets,references}.kt`,
`platform/lang-impl/src/com/intellij/find/actions/{resolver,SearchTargetVariantsDataRule}.kt`,
`codeInsight/navigation/impl/{gtd,gttd}.kt`):

* **Go To Declaration / Go To Type Declaration:** `declaredReferencedData()` looks for symbol
  references first; when an element has none, `ImplicitReferenceProvider`s are asked (the vendored
  `LspImplicitReferenceProvider` answers at the `PsiFile` level); only if that yields nothing does the
  platform fall back to classic `PsiReference` resolution (`fromTargetEvaluator`). Dart PSI has no
  symbol references (`getOwnReferences`/`PsiExternalReferenceHost` are unused in the plugin), so **the
  LSP answer takes precedence and legacy `DartResolver` is only the fallback**. This is why PR #539
  works without gating `DartResolver`, and why `textDocument/typeDefinition` (#580) can be enabled the
  same way with no legacy code to gate (the plugin registers no `TypeDeclarationProvider` at all —
  Go To Type Declaration simply does not exist for Dart today).
* **Find Usages:** `targetVariants()` collects `SEARCH_TARGETS` (LSP `LspSearchTargetsRule`) **and**
  the editor's PSI usage targets (`USAGE_TARGETS_KEY`, from `TargetElementUtil` → `DartResolver`).
  With both present, `findShowUsages()` shows a *"Choose target"* popup — every time. Enabling
  `LspFindReferencesSupport` therefore requires suppressing the PSI target (or a different
  integration), which touches `DartResolver`/target-element evaluation — a design decision the
  maintainers have to make (#546 was closed for exactly this reason: "removing `DartResolver` requires
  all of these other endpoints"). See open question Q1.
* **Hierarchies:** `typeHierarchyProvider`/`callHierarchyProvider` are `LanguageExtension`s; the LSP
  providers are registered for `language=""` with `order="last"`, Dart's own for `language="Dart"`,
  so Dart's providers win. Migrating #403 means the Dart providers must delegate to the LSP ones (or
  be unregistered) — and Dart's method hierarchy has no LSP equivalent.

## 3. The 16 open #207 sub-issues, classified

Legend — **LoL:** endpoint reachable over LSP-over-Legacy today · **Client:** vendored JetBrains LSP
client has the feature · **Legacy:** what the plugin uses now · **Verdict:** simple = plan now;
question = needs a maintainer answer (see `docs/superpowers/OPEN-QUESTIONS-maintainers.md`);
not mine = owned/in progress by maintainers.

| Issue | Endpoint(s) | LoL | Client | Legacy code | Group | Verdict |
|---|---|---|---|---|---|---|
| [#159](https://github.com/flutter/dart-intellij-third-party/issues/159) inlay hints (not a #207 child, but Ralph's) | `textDocument/inlayHint` | ✅ ≥ 3.14.0-139.0.dev | ✅ | none (closing labels are separate) | A/B | **simple — Stack 1, PR A** |
| [#580](https://github.com/flutter/dart-intellij-third-party/issues/580) typeDefinition | `textDocument/typeDefinition` | ✅ (≤ 3.3.0) | ✅ | none | A/B | **simple — Stack 1, PR B** |
| [#396](https://github.com/flutter/dart-intellij-third-party/issues/396) find usages | `textDocument/references` | ✅ ≥ 3.14.0-65.0.dev | ✅ | `DartServerFindUsagesHandler(Factory)`, `DartFindUsagesProvider`, `DartUsageTypeProvider`, grouping rules | A/C | question Q1 (PSI target arbitration) |
| — usage count (no issue) | `textDocument/codeLens` has no references lens; `textDocument/references` | ✅ | ✅ (`LspCodeVisionProvider`) | none | A | question Q2 (SDK lens rejected by DanTup in Dart-Code#1605 → IDE-side provider?) |
| [#400](https://github.com/flutter/dart-intellij-third-party/issues/400) closing labels | `dart/textDocument/publishClosingLabels` (notification) | ❌ not sent to LoL clients | ❌ custom notification | `DartClosingLabelsInlayHintsProvider`, `DartServerData.computedClosingLabels`, `DartClosingLabelManager` | B | question Q3 (SDK opt-in mechanism) |
| [#402](https://github.com/flutter/dart-intellij-third-party/issues/402) outline | `textDocument/documentSymbol` (pull) / `dart/textDocument/publishOutline` (push, LSP-only) | pull ✅ / push ❌ | helper only (`LspStructureViewSupport`), breadcrumbs ✅ | `DartStructureViewFactory/Model/Element` (`PsiTreeElementBase` + `analysis.outline`), Flutter plugin outline listeners | B/C | question Q4 |
| [#401](https://github.com/flutter/dart-intellij-third-party/issues/401) syntax highlighting | `textDocument/semanticTokens/full,range` | ❌ LSP-only | ✅ | `DartServerData` highlights, `DartAnnotator`, `DartSyntaxHighlighter` | B | question Q5 (SDK change + visual regression risk) |
| [#403](https://github.com/flutter/dart-intellij-third-party/issues/403) hierarchy | prepare/type/call hierarchy | ✅ | ✅ (`order="last"`) | `DartTypeHierarchyProvider`, `DartCallHierarchyProvider`, `DartMethodHierarchyProvider` | C | question Q6 |
| [#404](https://github.com/flutter/dart-intellij-third-party/issues/404) implementations & overrides | `textDocument/implementation`, `dart/textDocument/super` | ✅ | ❌ | `DartServerImplementationsMarkerProvider`, `DartServerOverrideMarkerProvider`, `DartServerGotoSuperHandler`, `DartInheritorsSearcher` (`analysis.implemented`/`analysis.overrides` push data) | C | question Q7 |
| [#399](https://github.com/flutter/dart-intellij-third-party/issues/399) completion | `textDocument/completion` (+resolve) | ❌ LSP-only (`initializationOptions`, `lspClientConfiguration`) | ✅ | `DartServerCompletionContributor` | C | question Q8 |
| [#407](https://github.com/flutter/dart-intellij-third-party/issues/407) rename | `textDocument/prepareRename`, `rename` | ❌ LSP-only | ✅ | `DartServerRenameHandler` | C | blocked on #520 per helin24; question Q9 |
| [#405](https://github.com/flutter/dart-intellij-third-party/issues/405) postfix templates | none in LSP | ❌ | ❌ | `DartRemotePostfixTemplate`, `DartPostfixTemplateProvider` | C | question Q10 |
| [#406](https://github.com/flutter/dart-intellij-third-party/issues/406) complete statement | none in LSP | ❌ | ❌ | `DartServerStatementCompletionProcessor` | C | question Q10 |
| [#441](https://github.com/flutter/dart-intellij-third-party/issues/441) context-message navigation | `Diagnostic.relatedInformation` | ✅ (via #612) | ✅ | `DartProblemsView` | B | not mine — belongs to helin24's #612 (`DartLspDiagnosticConverter`); note in Q11 |
| [#520](https://github.com/flutter/dart-intellij-third-party/issues/520) codeAction | `textDocument/codeAction` | ✅ | ✅ | quick fixes/assists/refactorings | C | not mine — helin24, PR #526 |
| [#374](https://github.com/flutter/dart-intellij-third-party/issues/374), [#385](https://github.com/flutter/dart-intellij-third-party/issues/385) analytics/timing | — | — | — | analytics infrastructure | — | not mine (Google-internal analytics decisions) |
| [#479](https://github.com/flutter/dart-intellij-third-party/issues/479) DAP debugger | DAP | — | — | debugger | — | not mine (helin24, P2, different protocol) |

Shared endpoints the vendored client supports that **#207 does not track** (candidates to *offer*, not
to start unasked): signature help (`textDocument/signatureHelp` vs. `DartParameterInfoHandler`),
workspace symbols (`workspace/symbol` vs. `DartClassContributor`/`DartSymbolContributor` — would
duplicate Search Everywhere results unless the Dart contributors yield), formatting
(`textDocument/formatting` vs. `DartFormattingModelBuilder`/`DartStyleAction`), document color
(`textDocument/documentColor` — Flutter-plugin territory, cf. flutter-intellij#7184 in #207's
cross-references). Listed in Q12.

## 4. The three groups Ralph asked for

**Group A — fits the work already planned (inlay hints / usage count) and can ship together:**
#159 inlay hints (ready), #580 typeDefinition (same 4-file "enable an endpoint" pattern as #539/#552,
nothing to gate), #396 find usages (the endpoint the usage count would also use — but blocked on Q1),
usage count itself (Q2). Also #400 closing labels sits next to the inlay-hint provider renamed in #551 —
but is blocked on the SDK protocol (Q3).

**Group B — display-only features:** #159 inlay hints, #400 closing labels, #401 syntax highlighting,
#402 outline/structure view (display, but a whole UI model to build), #441 problems-view navigation
detail, plus typeDefinition/documentHighlight-style navigation aids (#580).

**Group C — interactive features that need IDE UI or user flows:** #396 find usages (Find Usages
tool window + target chooser), #399 completion, #403 hierarchies (Hierarchy tool window), #404
implementations/overrides (gutter icons + navigation popups), #405 postfix templates, #406 complete
statement, #407 rename (dialog + preview), #520 code actions.

## 5. Decisions

| # | Decision | Rationale (evidence) |
|---|---|---|
| S1 | **Implement now — Stack 1:** PR A = inlay hints (Part 3 of the 2026-07-30 plan, `MIN_LSP_INLAY_HINTS_SDK_VERSION = "3.14.0-139.0.dev"`), PR B = `textDocument/typeDefinition` (#580) — originally stacked on PR A; since 2026-08-19 an independent PR (see §6 note). | Both are the proven 4-file pattern (bridge override + capability + `LspMethod` + customizer), no SDK work left, no legacy code to gate, no PSI arbitration problem (§2.3). Identical shape → reviewers see one pattern; stacking keeps Ralph at ≤ 2 open PRs. |
| S2 | typeDefinition gets **no SDK version gate**, only the experimental flag (like hover and documentHighlight). | Shared since ≤ Dart 3.3.0; the plugin only adds `MIN_…` constants for handlers shared recently (navigation `3.14.0-65.0.dev`, diagnostic server `3.13.0-106.0.dev`, inlay hints `3.14.0-139.0.dev`). |
| S3 | typeDefinition advertises `textDocument.typeDefinition.linkSupport = true` and deserializes `List<LocationLink>` exactly like `definition`. The capability goes into `DartAnalysisServerService.buildLspCapabilities` (introduced by helin24's #614) — PR B therefore waits for #614; if #614 dies, the only other place is `RequestUtilities.generateClientCapabilities` under `thirdPartySrc/`, which needs an explicit owner override (ask on the PR, precedent #539). | Mirrors the existing `definition` override and its comment; the SDK's `TypeDefinitionHandler` returns `Location` unless `typeDefinitionLocationLink` is set (`client_capabilities.dart:201`), and `LocationLink.originSelectionRange` is what the vendored `LspImplicitReferenceProvider` uses for the reference range. The vendored executor accepts both shapes, so this is consistency plus better highlighting, not a hard necessity. |
| S4 | **Do not start** #396, usage count, #400, #402, #403, #404 before the maintainers answer Q1–Q7. Prepare the questions so that they can be pasted into the issues. | Each has an unresolved design point (PSI arbitration, SDK opt-in protocol, UI ownership) — planning them now would be guessing. |
| S5 | **Do not touch** #399, #401, #405, #406, #407 beyond documenting the SDK/protocol gaps (Q5, Q8–Q10). | LSP-only handlers or no LSP protocol at all; #407 explicitly blocked by helin24 until #520. |
| S6 | **Not mine:** #520/#526, #612 (+#441), #614, #374, #385, #479. Rebase Stack 1 on top of #612/#614 when they merge (they touch `DartBridgeLspServer.kt`, `DartBridgeLspServerTest.kt`, `DartAnalysisServerService.java`). | Avoid overlapping with the maintainer's in-flight PRs; the CLA/2-PR budget is better spent on things nobody else is doing. |
| S7 | Usage count stays **design-only** for now (spec D3 upheld), but the question is sharpened: DanTup rejected a server-side references lens (Dart-Code#1605, 2025-10-27), so the only viable path is an IDE-side `CodeVisionProvider` counting via `textDocument/references` — ask before building (Q2). | Evidence in §7. |
| S8 | No new SDK handoff document now: Stack 1 needs no SDK change. Handoff docs for closing labels (Q3) or others will be written once the mechanism is agreed. | Writing them before the protocol decision would violate "nothing invented". |

## 6. Stacked-PR strategy

> **Superseded 2026-08-19:** GitHub stacked PRs require the whole stack (trunk included) in one
> repository — "Cross-fork stacks are not supported" (reference docs; github/gh-stack#46 tracks
> fork support as future work), and a PR's base branch must exist in the base repository. From the
> fork, PR A (#617) and PR B (#618) are therefore **independent PRs against `main`** that only overlap
> textually; whichever merges second gets a trivial rebase. The 2-open-PR budget is unchanged. The
> text below is kept for the record.


* GitHub stacked PRs: PR B's `--base` is PR A's branch; after A merges, GitHub retargets B to `main`
  automatically (or Ralph retargets). Only PR A counts as "reviewable now"; B is visible with a
  focused diff.
* Order inside Stack 1: A = inlay hints (bigger, needs the SDK version gate; the reviewer already
  knows the story from #551/#552 and sdk#64061), B = typeDefinition (small, same shape).
* Each PR: its own CHANGELOG line under `## Unreleased / ### Added`, its own manual sandbox
  verification per the migrate-das-to-lsp skill, its own repository code review
  (`.agents/skills/code-review`), commits by `Ralph Bergmann <ralph@dasralph.de>` only, no
  `Co-Authored-By`, `git add` with explicit paths.
* Future stacks (after answers): Stack 2 "navigation family" (#396 references, #404 implementation
  once the arbitration/gutter design is agreed, #403 hierarchies), Stack 3 "push notifications over
  LoL" (#400 closing labels, #402 outline) — each needs an SDK/plugin design first.

## 7. Answers to Ralph's questions

**"Aktuell haben wir inlay hints und usage count geplant, richtig?"** — Inlay hints: yes, Part 3 of
`plans/2026-07-30-dart-inlay-hints.md`, now unblocked (SDK commit `7c18d1fa0e5`, first dev tag
`3.14.0-139.0.dev`). They ship with the server's default configuration (all six `dart.inlayHints`
categories on) because the configuration channel is LSP-only — per-category settings are a
follow-up tied to Q13. Usage count: it was *design-only* (decision D3) because the SDK's `CodeLensHandler`
only emits augmentation lenses. That is still true, and DanTup has since stated on the corresponding
VS Code request (Dart-Code#1605, 2025-10-27) that he considers reference counts a client/LSP-generic
feature, not something the analysis server should compute. So the realistic path is an IDE-side
`CodeVisionProvider` in the Dart plugin that counts `textDocument/references` results per visible
declaration — one server round trip per declaration, cached by the daemon. Whether the maintainers
want that is Q2.

**"find usages scheint mir ähnlich usage count zu sein"** — Same endpoint (`textDocument/references`,
shared since `3.14.0-65.0.dev`), different UI: find usages = Find Usages tool window driven by the
platform's search-target machinery; usage count = Code Vision inlay above the declaration. The
find-usages migration has the PSI-vs-LSP target problem described in §2.3 (a "Choose target" popup
unless the PSI target is suppressed); the usage count does not have that problem but has the cost
question. Both need a maintainer answer (Q1, Q2) before implementation.

**"Change closing labels to LSP können wir ja nun auch machen, war doch die Klasse, die wir
umbenannt hatten, richtig?"** — Half right. `DartClosingLabelsInlayHintsProvider` is the *renderer*;
it reads `DartServerData.getClosingLabels(file)`, which is filled from the legacy
`analysis.closingLabels` notification (subscription `CLOSING_LABELS` in
`DartAnalysisServerService.analysis_setSubscriptions`). "To LSP" means feeding it from
`dart/textDocument/publishClosingLabels` instead — DanTup confirmed on #400 (via helin24, 2026-08-17)
that closing labels stay a custom notification and that the plugin needs "equivalent work to accept
the same". The blocker: the legacy server never sends that notification (only `LspAnalysisServer`
does, gated by `initializationOptions.closingLabels`, `lsp_analysis_server.dart:903`), and there is no
`initializationOptions` over LoL. An SDK change modelled on `d42063aac44` (publishDiagnostics for LoL
clients) is needed, and its opt-in mechanism has to be chosen by the SDK team → Q3. The plugin-side
work afterwards is small (bridge forwards the notification into `DartServerData.computedClosingLabels`
after converting ranges to offsets; the renamed provider stays untouched).

## 8. Out of scope (unchanged)

Modifying `third_party/thirdPartySrc/**` outside `patch.py`; PSI `DartReadWriteAccessDetector`;
per-category inlay-hint settings UI (spec §4.3 follow-up); anything DAP-related.

## 9. Follow-ups after this analysis

1. Ralph reviews this file and `OPEN-QUESTIONS-maintainers.md`; posts the questions (Q1–Q3 first).
2. Implement Stack 1 per `plans/2026-08-18-lsp-endpoint-stack-1.md`.
3. Revisit §3 when answers arrive; open Stack 2/3 planning then.

## 10. Verified facts (2026-08-18)

* SDK: `git -C ../dart-sdk/sdk log origin/main --grep 'inlayHint a shared handler'` → `7c18d1fa0e5`;
  `git merge-base --is-ancestor 7c18d1fa0e5 3.14.0-138.0.dev` → false, `…139.0.dev` → true. Landed
  content: 4 files (+44/−2), `test/lsp_over_legacy/inlay_hint_test.dart` uses `initializeServer()`
  (patch set 3 fix; documented in sdk#64061 comment 2026-08-17). Local checkout `HEAD` is still the
  local CL branch commit `75e7d44de95` — `git checkout main && git pull` before any new SDK work.
* SDK shared-handler list and blame: `handler_states.dart` at `origin/main` (§2.1 table);
  `git show 3.3.0:…/handler_states.dart` confirms typeDefinition/documentSymbol/implementation/
  type-hierarchy were shared in 3.3.0; `SuperHandler` and `CodeLensHandler` in 3.5.0.
* Closing labels: `lsp_analysis_server.dart:697` (`publishClosingLabels`), `:903`
  (`shouldSendClosingLabelsFor` = `initializationOptions.closingLabels && priorityFiles.contains`),
  `LspServerContextManagerCallbacks.handleResolvedUnitResult` (`:1354`); legacy path
  `legacy_analysis_server.dart:1312` (`AnalysisService.CLOSING_LABELS`) →
  `operation_analysis.dart:79 sendAnalysisNotificationClosingLabels`.
* publishDiagnostics-over-LoL precedent: `git show d42063aac44` — `notification_manager.dart`
  checks `editorClientCapabilities.publishDiagnostics` and calls `analysisServer.sendLspNotification`;
  design discussion in sdk#64021 (DanTup proposed reusing the LSP client capability as opt-in).
* Server options: `git show origin/main:pkg/analysis_server/tool/lsp_spec/README.md` sections
  "Initialization Options" (lines 24–32), "Client Workspace Configuration" (33–57), "Method Status"
  (58–166: `inlayHint/resolve`, `codeLens/resolve`, `textDocument/declaration` unsupported),
  "Client Commands" (385–432: `experimental.commands: ["dart.goToLocation"]`);
  `client_configuration.dart:140` `LspClientConfiguration.replace`, `:195` `LspClientInlayHintsConfiguration`
  (`boolean ?? true` defaults, lines 207–217); `legacy_analysis_server.dart:318/407` default
  configuration; `handler_workspace_configuration.dart:31` `fetchClientConfigurationAndPerformDynamicRegistration()`.
* No LSP postfix/statement completion: `git grep -n -i -E 'postfix|statementCompletion' origin/main -- pkg/analysis_server/lib/src/lsp` → empty; legacy handlers exist under `lib/src/handler/legacy/edit_get_postfix_completion.dart`, `edit_get_statement_completion.dart`, `edit_list_postfix_completion_templates.dart`.
* LSP-only handler dependencies: `handler_completion.dart:86-87` uses `server.initializationOptions`;
  `handler_rename.dart:106` uses `server.lspClientConfiguration.global`, `:65/:159`
  `server.refactoringWorkspace`; `AbstractSemanticTokensHandler extends LspMessageHandler`
  (`handler_semantic_tokens.dart:24`).
* Plugin: `plugin.xml` registrations lines 58–187 (see §3 legacy column); no `typeDeclarationProvider`,
  `TypeDeclarationProvider`, `SymbolTypeProvider`, `TargetElementEvaluator`, breadcrumbs provider,
  `getOwnReferences` or `PsiExternalReferenceHost` anywhere under `third_party/src/main`.
  Legacy subscriptions still always on: `DartAnalysisServerService.analysis_setSubscriptions`
  (lines 2003–2019: HIGHLIGHTS, NAVIGATION, OVERRIDES, OUTLINE, IMPLEMENTED, CLOSING_LABELS).
  LSP gates in use: `DartDocumentationProvider:44,97`, `DartReferenceContributor:55`,
  `AnalysisServerDiagnosticsAction:56`, `DartLspServerDescriptor:106,113,132`.
* Vendored client: `dart-lsp-impl.xml` EP list (§2.2); `LspImplicitReferenceProvider.kt` lines
  ~32–139 (acts only for `GotoDeclarationAction`/`GotoTypeDeclarationAction`, file-level element,
  empty when the customizer is Disabled); `LspSymbolTypeProvider.kt` returns the `LspNavigatableSymbol` produced
  by that reference; `LspSearchTargetsRule.kt` builds an `LspSearchTarget` whenever a server with
  `LspFindReferencesSupport` has the file open; `LspTypeHierarchyProvider`/`LspCallHierarchyProvider`
  registered `language=""`, `order="last"`; `LspStructureViewSupport` is `@ApiStatus.Internal` and
  unused inside the vendored copy; `LspRequestExecutor.getTypeDefinitions` maps `Location`s to
  `LocationLink`s.
* IntelliJ platform (GitHub master, fetched 2026-08-18): `model/psi/impl/references.kt`
  `allReferencesInElement` (symbol references, else implicit), `model/psi/impl/targets.kt`
  `declarationsOrReferences` (implicit before `fromTargetEvaluator`),
  `find/actions/SearchTargetVariantsDataRule.kt` `targetVariants` (SEARCH_TARGETS + USAGE_TARGETS),
  `find/actions/resolver.kt` `findShowUsages` (size > 1 → `createTargetPopup`),
  `codeInsight/navigation/impl/gttd.kt` (`SymbolTypeProvider` for non-PSI symbols).
* Maintainer statements: helin24 on #396 (2026-07-24: unblocked by #63884), on #407 (2026-07-13:
  blocked until codeAction), on #400 (2026-08-17: DanTup — closing labels stay a custom notification;
  "we'd need some equivalent work in the IJ plugin to accept the same"), on #546 (2026-08-06: closed,
  not a cohesive work item), on sdk#64061 (2026-08-17: supports inlay hints over LoL); bwilkerson on
  sdk#64061 (wants the protocol to actually be used); DanTup on Dart-Code#1605 (2025-10-27: reference
  counts should be a client/LSP-generic feature, not server-computed).
* helin24 draft PRs #526 (2026-07-16), #612 (2026-08-17), #614 (2026-08-18): file lists in §2.2.
