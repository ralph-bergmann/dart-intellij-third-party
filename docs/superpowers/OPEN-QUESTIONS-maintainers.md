# Open questions for the maintainers (LSP migration, tracking issue #207)

**Purpose:** the tasks below are ones we would like to contribute but that are not well-defined enough
to plan without an answer from the plugin maintainers (helin24, pq) or the analysis-server side
(DanTup, bwilkerson). Each entry gives the evidence, the concrete question, and — where we have one —
a proposal, so that the text can be pasted into the linked issue as-is. Analysis backing this file:
`docs/superpowers/specs/2026-08-18-lsp-migration-scope-analysis.md`.

**Status legend:** ⏳ not asked yet · 💬 asked (link) · ✅ answered (summary + date).

---

## Q0 — Process: stacked PRs and what to pick up (post on #207) ⏳

Context: the repo allows two open PRs per external contributor; we want to submit closely related
changes as *stacked* PRs (PR B based on PR A's branch) so that one review covers one pattern.

Questions:
1. Are stacked PRs acceptable to you (review PR A first; PR B is retargeted to `main` after A merges)?
2. Of the open #207 items nobody is assigned to (#396, #400, #401, #402, #403, #404, #405, #406, #407,
   #580), which would you like us to take, and which are you planning to do yourselves? We are
   starting with #159 (inlay hints, sdk#64061 has landed) and #580 (typeDefinition) as one stack.

---

## Q1 — #396 Find usages via `textDocument/references`: how should PSI and LSP targets coexist? ⏳

Evidence:
* `textDocument/references` is available over LSP-over-Legacy since `3.14.0-65.0.dev`
  (dart-lang/sdk#63884, same CL as definition), and the vendored client implements Find Usages via
  `LspSearchTargetsRule` + `LspUsageSearcher` (`textDocument/references` → `Usage`s).
* IntelliJ's `FindUsagesAction` collects **both** `SEARCH_TARGETS` (the LSP rule) **and** the
  editor's PSI usage targets (`USAGE_TARGETS_KEY`, produced by `TargetElementUtil` →
  `DartResolver`). With two variants it shows a *"Choose target"* popup on every invocation
  (`platform/lang-impl/src/com/intellij/find/actions/resolver.kt`, `findShowUsages`, `size > 1`).
  Unlike Go To Declaration, where the LSP implicit reference takes precedence over PSI resolution,
  there is no built-in precedence for Find Usages.
* Legacy pieces: `DartServerFindUsagesHandlerFactory`/`DartServerFindUsagesHandler`
  (`search.findElementReferences`), `DartFindUsagesProvider`, `DartUsageTypeProvider`, the two
  `fileStructureGroupRuleProvider`s. Gating the handler factory alone does **not** remove the PSI
  target variant.

Question: which integration do you want?
* (a) Enable `LspFindReferencesSupport` and suppress the PSI target while the flag is on — e.g. gate
  `DartResolver` for reference resolution **and** add a `TargetElementEvaluatorEx2` for Dart that
  returns no named element/target when LSP find usages is enabled (new class, blast radius on other
  PSI consumers: rename, inline, refactorings, gutter markers).
* (b) Keep `DartServerFindUsagesHandler` as the UI integration (single PSI target, Dart usage
  grouping/usage types stay) but source its results from `textDocument/references` through the
  bridge instead of `search.findElementReferences` — no JetBrains-client feature involved.
* (c) Accept the chooser popup while experimental (we would not recommend it).
* Also: is Find Usages part of the `DartResolver` retirement you had in mind in #546, i.e. should
  the PSI suppression be designed for all of #396/#404/#403 at once?

Our preference: (a) if you want to retire the PSI path, (b) if you want a minimal, reversible change
now. We can prototype either.

---

## Q2 — Usage counts ("n usages" Code Vision) — IDE-side provider acceptable? ⏳

Evidence:
* The SDK `CodeLensHandler` only provides augmentation lenses; there is no references lens. On the
  VS Code request for exactly this (Dart-Code/Dart-Code#1605, open since 2019) DanTup wrote on
  2025-10-27: *"this is best implemented as a VS Code (and LSP) feature (either as something new or by
  using the existing data that exists) and then just the data provided by the extension"* — i.e. the
  server should not compute reference counts.
* IntelliJ's own "usages" Code Vision for Java/Kotlin is IDE-side (`ReferencesCodeVisionProvider`
  subclasses counting via `ReferencesSearch`). For Dart the equivalent would be a
  `DartReferencesCodeVisionProvider` that, per visible declaration, sends `textDocument/references`
  (LSP, via the bridge server for the file) and shows the count; hidden behind the existing
  *Settings | Editor | Inlay Hints | Code vision* toggle plus the experimental-LSP flag. The
  third-party "Flutter Developer Tools" plugin does the same over the legacy
  `search.findElementReferences` (`ReferencesCodeVisionProvider` + `DartServerFindUsagesHandler`, see
  the 2026-07-30 design doc §3); the abandoned "Flutter Enhancement Suite" showed usage counts too.

Questions:
1. Would you accept an IDE-side Code Vision provider that issues one `textDocument/references`
   request per visible declaration (daemon-cached, cancellable, capped result count)? Any
   performance envelope you want us to respect (e.g. only for files below N declarations, debounce)?
2. If not, would you rather have this raised with the analysis-server team as a code lens
   (contradicting DanTup's stated position), or dropped?

---

## Q3 — #400 Closing labels via LSP: how does an LSP-over-Legacy client opt in? (post on #400 and, if agreed, a new dart-lang/sdk issue) ⏳

Evidence:
* Closing labels are a custom notification, `dart/textDocument/publishClosingLabels`
  (DanTup on #400 via helin24, 2026-08-17: not inlay hints; the IJ plugin "would need some
  equivalent work … to accept the same").
* Today only `LspAnalysisServer` sends it, and only when the client passed
  `initializationOptions.closingLabels = true` and the file is a priority (open) file
  (`lsp_analysis_server.dart:903 shouldSendClosingLabelsFor`, `:1354`). The legacy server keeps
  sending `analysis.closingLabels` for the `CLOSING_LABELS` subscription. There is no
  `initializationOptions` over LoL, and no LSP client capability for closing labels.
* Precedent for switching a legacy push to an LSP notification for LoL clients: dart-lang/sdk#64021
  → `d42063aac44` (2026-08-17): the presence of `textDocument.publishDiagnostics` in the LoL client's
  `lspCapabilities` makes `NotificationManager.sendAnalysisErrors` send
  `textDocument/publishDiagnostics` via `sendLspNotification` instead of `analysis.errors`.
* Plugin side (small, once the server sends it): `DartBridgeLspServer.forwardNotificationToClient`
  currently forwards only `textDocument/publishDiagnostics`; closing labels would be routed into the
  existing `DartServerData.computedClosingLabels(fileInfo, labels)` (LSP ranges → offsets), so the
  renamed `DartClosingLabelsInlayHintsProvider` and `DartClosingLabelManager` stay as they are.

Questions:
1. Do you want closing labels migrated now (while still on LoL), or only when the plugin talks
   pure LSP (`initializationOptions` become available)? If now:
2. Which opt-in should the SDK use for LoL clients — (a) a Dart-specific entry in
   `server.setClientCapabilities` (e.g. a new `lspInitializationOptions`/`closingLabels` field),
   (b) reusing the legacy `CLOSING_LABELS` subscription as the trigger and switching the *format* to
   the LSP notification when the client has advertised LSP capabilities (breaks nothing for clients
   that never send `lspCapabilities`, but changes behaviour for the current plugin as soon as it sends
   them — so a plugin-side switch is needed at the same time), or (c) something under
   `lspCapabilities.experimental`?
3. Same question for `dart/textDocument/publishOutline` (#402) — should both go into one SDK issue?

If you tell us the mechanism, we will file the SDK issue, write the CL (mirroring `d42063aac44`,
with an `lsp_over_legacy` test) and the plugin PR.

---

## Q4 — #402 Outline: pull `documentSymbol` or push `publishOutline`, and who owns the structure view UI? ⏳

Evidence:
* Legacy: `DartStructureViewFactory/Model/Element` build the Structure view from the
  `analysis.outline` push (`DartServerData.OutlineListener`); `DartStructureViewElement extends
  PsiTreeElementBase` (the class that breaks on 2026.2 EAP, see the verifier baseline).
* LSP has two shapes: standard `textDocument/documentSymbol` (pull, shared over LoL since ≤ 3.3.0;
  the vendored client offers `LspStructureViewSupport` — an internal helper without any
  `PsiStructureViewFactory`, plus `LspFileBreadcrumbsCollector`, which would give Dart breadcrumbs
  for free) and the custom push `dart/textDocument/publishOutline` (LSP-only, same problem as Q3).
* Consumers of `DartAnalysisServerService.getOutline`/`addOutlineListener` inside this repo:
  only `DartStructureViewModel`. **[verify]** whether the Flutter plugin (Flutter Outline view) reads
  this outline or only its own `flutter.outline` notification before changing the data source.

Questions:
1. Should the Structure view be re-implemented on `textDocument/documentSymbol` (new
  `StructureViewModel`/`TreeElement` classes on top of `LspStructureViewSupport`, icons via
  `symbolKindCustomizer`, live updates by re-requesting on document change), or should the plugin
  keep the outline-push model and switch the *data source* to `publishOutline` once the SDK sends
  it to LoL clients (Q3.3)?
2. Is enabling `LspDocumentSymbolSupport` only for breadcrumbs (leaving the Structure view legacy)
   an acceptable first step?

---

## Q5 — #401 Syntax highlighting via semantic tokens ⏳

Evidence: `SemanticTokensFullHandler`/`RangeHandler` are LSP-only (`AbstractSemanticTokensHandler
extends LspMessageHandler`); an SDK conversion à la #63884 would be needed first, and the legacy
`HIGHLIGHTS` subscription feeds `DartAnnotator` (server highlight regions merged with the lexer
highlighter, colour keys in `DartSyntaxHighlighterColors`). The vendored client's
`LspSemanticTokensCustomizer` maps LSP token types/modifiers to platform text attributes.

Questions: is this on your roadmap, and if we did it, would you accept a token→`TextAttributesKey`
mapping that keeps today's Dart colour scheme keys (so user schemes keep working)? Should the SDK
issue to share the semantic-tokens handlers be filed now?

---

## Q6 — #403 Hierarchies: delegate the Dart providers to the LSP ones? ⏳

Evidence: prepare/type/call hierarchy handlers are shared over LoL (Dart 3.3.0); the vendored
`LspTypeHierarchyProvider`/`LspCallHierarchyProvider` are registered for `language=""` with
`order="last"`, so `DartTypeHierarchyProvider`/`DartCallHierarchyProvider` (`language="Dart"`)
always win. There is no LSP method hierarchy (`DartMethodHierarchyProvider`, `search.getTypeHierarchy`
based) — it would remain legacy or be dropped.

Question: when the flag is on, should the Dart providers delegate all three `HierarchyProvider`
methods to instances of the LSP providers (keeps registration stable, small change), or should the
Dart registrations be removed once LSP is the default? Is losing the method hierarchy acceptable
(document as trade-off) or must it stay?

---

## Q7 — #404 Implementations & overrides: no JetBrains-client feature exists ⏳

Evidence: `textDocument/implementation` and `dart/textDocument/super` are shared over LoL, but the
vendored client has no customizer/EP for them (no `LspGoToImplementation…`). Legacy uses *push*
data (`analysis.implemented` → `DartServerImplementationsMarkerProvider` gutter icons,
`analysis.overrides` → `DartServerOverrideMarkerProvider`) plus `DartServerGotoSuperHandler` and
`DartInheritorsSearcher`. LSP `implementation` is a per-position *pull* request — gutter icons for
every declaration in a file would need one request per declaration or a new server notification.

Questions: do you want (a) only the navigation actions (Go to Implementation / Go to Super) on
LSP now via plugin-side `DefinitionsScopedSearch`/`gotoSuper` implementations calling the bridge,
keeping the gutter icons legacy, or (b) wait for a server-side push (`publishImplemented`?) over
LoL, or (c) accept N `textDocument/implementation` requests per file for the markers?

---

## Q8 — #399 Completion ⏳

Evidence: `CompletionHandler` is LSP-only and depends on `server.initializationOptions`
(`suggestFromUnimportedLibraries`, `completionBudgetMilliseconds`) and `lspClientConfiguration`;
the legacy `DartServerCompletionContributor` is large and has plugin-specific UX (lookup
decoration, `DartCharFilter`, `completion_setSubscriptions` for available suggestion sets).

Question: is completion something you plan to do yourselves (it is P1 and the biggest UX change),
and if not, what is the SDK story — a shared handler plus a legacy way to pass the initialization
options?

---

## Q9 — #407 Rename ⏳

Evidence: `PrepareRenameHandler`/`RenameHandler` are LSP-only (use `server.lspClientConfiguration.global`,
`server.refactoringWorkspace`); helin24 marked #407 blocked until #520 (named constructors /
import prefixes, dart-lang/sdk#61008, #61199). The vendored `LspRenameHandler` exists.

Question: once #526 lands, should the SDK issue to share the rename handlers be filed, and do you
want the vendored `LspRenameHandler` (JetBrains rename dialog) or a Dart-specific handler that keeps
the current dialog options?

---

## Q10 — #405 Postfix templates and #406 Complete statement: no LSP protocol ⏳

Evidence: `git grep -i 'postfix\|statementCompletion' pkg/analysis_server/lib/src/lsp` is empty;
only legacy handlers exist (`edit_get_postfix_completion.dart`, `edit_get_statement_completion.dart`,
`edit_list_postfix_completion_templates.dart`).

Question: are these meant to be (a) new custom LSP methods in the analysis server (who designs
them?), (b) re-implemented client-side (IntelliJ postfix templates / smart-enter without server help),
or (c) dropped when the legacy protocol goes away? Until that is decided we will not touch them.

---

## Q11 — #441 Context-message navigation (coordinate with #612) ⏳

Note for helin24: `Diagnostic.relatedInformation` carries the context-message locations; the new
`DartLspDiagnosticConverter` in #612 is the natural place to keep them navigable in the Dart Analysis
tool window. We can help verify/test once #612 is in — just say so on #441.

---

## Q12 — Shared endpoints not tracked in #207 — do you want issues for them? ⏳

All shared over LoL and supported by the vendored client: `textDocument/signatureHelp` (vs.
`DartParameterInfoHandler`; the vendored `LspParameterInfoHandler` is registered for `language=""`,
so precedence between the two would have to be checked), `workspace/symbol` (vs. `DartClassContributor`/`DartSymbolContributor` — Search
Everywhere would list results twice unless the Dart contributors yield), `textDocument/formatting`
(vs. `DartFormattingModelBuilder`/`DartStyleAction`), `textDocument/documentColor` (Flutter colour
previews — Flutter plugin territory). Should any of these become #207 sub-issues, and would you take
PRs for them?
