# LSP endpoint candidates

**Purpose:** A complete map of which LSP endpoints the plugin could still wire up — as a basis for discussing the
maintainers' migration planning. **Deliberately no issues filed yet** for the new candidates; rows from tables B–D
become issues (or not).

**Status update 2026-10-07 (GitHub re-checked):**

- **Merged since the last update:** #693 document highlight out of experimental mode, #695 file content sync before
  LSP requests (`tryRunReadAction`), #624 completion, #673 structure view (#633).
- **Open PRs (others):** #640 syntax highlighting, #689 formatting (#634), #700 semantic highlighting fix, #711 outline
  / file structure (#402, #655, draft), #703 go to usages fix (draft).
- **Optimize Imports (#691):** helin's answers — answering `applied: true` early is fine, reuse the code-actions gate
  (#526), Sort Members separately → **PR #719 open** (2026-10-07). The `applyEdit` is caught in the LSP client layer
  (`DartLspServerDescriptor.createLsp4jClient`) instead of the bridge, so it survives the full LSP switch. Sort Members
  filed as **#716** (same mechanism, after #719).
- **Rename (#407):** helin's answers — standard IntelliJ rename UI, local variables over LSP too, defer
  `renameFilesWithClasses` / `updateImportsOnRename` (filed as **#717** / **#718**, both ask where the setting should
  live), make the server's *Rename Anyway* prompt modal in the rename PR (not in #657), implement the rename tricks
  first and decide afterwards. Next after #719.

**Status update 2026-09-27 (GitHub + SDK `main` re-checked):**

- **LoL column:** dart-lang/sdk@3e33bc81443 (DanTup, sdk#64339, 2026-09-21) shared all remaining LSP handlers with
  LSP-over-Legacy, first in `3.14.0-252.0.dev` (no protocol bump → gate by SDK version). Completion
  (`612f4b34a40`) and semantic tokens (`90b1c38b10f`) were shared before that. Every former ✗ below is therefore ✅ on
  SDK `main`, and step 3 of the suggested order (sharing CLs) is obsolete. Client configuration over LoL
  (`workspace/configuration`) = `6700ccc4316`, `3.14.0-258.0.dev`.
- **Merged since the last update:** #615 type definition, #623 find usages, #627 type/call hierarchy, #638
  `willRenameFiles` for file rename/move, #663 closing labels, #526 code actions, #679 per-category inlay hint options.
- **Open PRs (others):** #624 completion, #640 syntax highlighting, #673 structure view, #689 formatting (#634).
- **Rename (#407):** implementation plan + open questions posted on
  [#407](https://github.com/flutter/dart-intellij-third-party/issues/407#issuecomment-5855528632) — keep IntelliJ's
  rename UI (in-place vs. dialog per the IDE setting) and swap only the backend via a `RenamePsiElementProcessor`;
  waiting for a go.
- **Optimize Imports:** new issue **#691** with plan + open questions (IntelliJ's import optimizer via the
  `source.organizeImports` code action; the Alt+Enter source action already came with #526); waiting for a go.

**Status update 2026-08-21 (GitHub re-checked):**

- PR #614 (client capabilities, helin24) has been **merged** (2026-08-20).
- **NEW:** PR #623 (ranbeuer, 2026-08-21) implements Find Usages via
  `textDocument/references`, gated behind the experimental flag → that row in table C is taken.
- Inlay hints stage 2 is now filed as #622 + dart-lang/sdk#64101 (cross-linked).
- **Evening addendum:** PR #612 (diagnostics) has also been **merged** (2026-08-21);
  PR #617 was approved by helin24 (the isExperimental clarification + a merge of main followed).

Sources (all verified on 2026-08-19):

- **JB client** = the bundled JetBrains LSP client (`thirdPartySrc/platform-lsp`, 26 customizers in
  `LspCustomization.kt`); plugin status taken from
  `DartLspServerDescriptor.kt:105-146` (anything still set to `…Disabled` is a candidate).
- **LoL?** = the handler is listed in `sharedHandlerGenerators` in
  `pkg/analysis_server/lib/src/lsp/handlers/handler_states.dart` on SDK `origin/main`
  (`9da766e4c31`, 2026-08-19) → reachable via LSP-over-Legacy (`lsp.handle`). ✗ = real LSP server only → first needs a
  sharing CL in the SDK (template: our inlay-hint CL `7c18d1fa0e5`; plan file
  `plans/2026-07-30-sdk-share-inlay-hint-handler.md` is in our internal notes).
- Server support per `pkg/analysis_server/tool/lsp_spec/README.md`.
- Gating follows the #615 principle: **no legacy path → enable globally, no flag; legacy path exists → experimental flag
  as the legacy↔LSP switch.**

## A. Done / in progress (hands off)

| Feature                        | LSP method                                                 | Status                                                                                                             |
|--------------------------------|------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Hover                          | `textDocument/hover`                                       | ✅ merged (#291)                                                                                                   |
| Go to Declaration (⌘B)         | `textDocument/definition`                                  | ✅ merged (#398 / PR #539)                                                                                         |
| Read/write highlighting        | `textDocument/documentHighlight`                           | ✅ merged (PR #552); enabled for all users since PR #693 (2026-09-28)                                              |
| Go to Type Declaration (⌘⇧B) | `textDocument/typeDefinition` | ✅ merged (PR #615, helin24); fixes #237 + #580 |
| Inlay hints | `textDocument/inlayHint` | ✅ merged 2026-09-09 (PR #617); SDK gate 3.14.0-139.0.dev. Stage 2 (#622, per-category options) ✅ merged 2026-09-25 (PR #679): `workspace/configuration` (server→client pull) + `workspace/didChangeConfiguration` (client→server, as legacy `lsp.notification`), SDK gate 3.14.0-258.0.dev (dart-lang/sdk@6700ccc4316, sdk#64101) |
| Code actions | `textDocument/codeAction` + `executeCommand` + `applyEdit` | ✅ merged 2026-09-22 (PR #526, helin24; #520); includes the Organize Imports / Sort Members source actions in Alt+Enter; extract method/variable shortcuts → #684 |
| Diagnostics                    | `textDocument/publishDiagnostics`                          | ✅ merged 2026-08-21 (PR #612); #441 still open; issue #292 already closed in 2026-05                              |
| Client capabilities            | `initialize` rework                                        | ✅ merged 2026-08-20 (PR #614)                                                                                     |
| Find Usages (`findReferences`) | `textDocument/references` | ✅ merged (PR #623, ranbeuer); fixes #396 |
| Completion | `textDocument/completion` + `completionItem/resolve` | ✅ merged 2026-10-05 (PR #624, juan-mora-google; #399); SDK gate 3.14.0-226.0.dev |
| Structure view | `textDocument/documentSymbol` | ✅ merged 2026-10-05 (PR #673, ranbeuer; #633); outline / file structure → PR #711 (draft) |

## B. Candidates WITHOUT a legacy path → global, no gating (least review friction)

| Feature (customizer)                     | LSP method(s)                                     | LoL?   | Issue           | Notes                                                                                                                                            |
|------------------------------------------|---------------------------------------------------|--------|-----------------|--------------------------------------------------------------------------------------------------------------------------------------------------|
| Color preview + picker (`documentColor`) | `textDocument/documentColor`, `colorPresentation` | ✅     | —               | Gutter color chips for `Color(…)` — a visible win for Flutter, JB client is ready                                                                |
| Go to Imports (`—`, custom)              | `dart/textDocument/imports`                       | ✅     | **#582 exists** | Custom method: no customizer, needs its own action in the plugin                                                                                 |
| Code Lens (`codeLens`)                   | `textDocument/codeLens`                           | ✅     | —               | Server currently mostly serves augmentation links; DanTup rejected usage counts server-side (Dart-Code#1605) → small benefit, but small cost too |
| Document Links (`documentLink`) | `textDocument/documentLink` | ✅ | — | Shared with LoL since dart-lang/sdk@3e33bc81443 (3.14.0-252.0.dev); moderate benefit (clickable URIs in comments) |

## C. Candidates WITH a legacy path → experimental flag (legacy↔LSP switch)

| Feature (customizer)                                             | LSP method(s)                                                | LoL?   | Issue | Legacy in the plugin                            | Notes                                                                                                                                                                                                                                                                  |
|------------------------------------------------------------------|--------------------------------------------------------------|--------|-------|-------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Formatting (`formatting`, `onTypeFormatting`) | `formatting`, `rangeFormatting`, `onTypeFormatting` | ✅ | #634 | `edit.format` | PR #689 (danxorzum) open. Addresses formatter pain points #35/#152/#153/#309 |
| Optimize imports (`optimizeImports`) | `codeAction` `source.organizeImports` → `executeCommand` → `applyEdit` | ✅ | #691 | `edit.organizeDirectives` (`DartImportOptimizer`) | **Ours: PR #719 open (2026-10-07).** The vendored `LspImportOptimizer` can't be used (the Dart action only carries a command; client-side `applyEdit` would deadlock a synchronous optimizer) → own `DartLspImportOptimizer`; the edit is caught in the LSP client layer (`createLsp4jClient` wrapper + `DartLspApplyEditCapture`) and applied in the optimizer's write command. Code-actions gate (3.14.0-137.0.dev) |
| Sort members (`—`) | `codeAction` `source.sortMembers` → `executeCommand` → `applyEdit` | ✅ | #716 | `edit.sortMembers` (`DartSortMembersAction`) | **Ours: filed 2026-10-07**, after #719 (same capture mechanism); Alt+Enter already offers it via #526 |
| Parameter info (`signatureHelp`)                                 | `textDocument/signatureHelp`                                 | ✅     | —     | `ParameterInfoHandler` (PSI)                    | The ⌘P popup                                                                                                                                                                                                                                                           |
| Structure view / outline (`documentSymbol`)                      | `textDocument/documentSymbol`                                | ✅     | #402  | `analysis.outline` subscription                 | **Taken:** structure view merged (PR #673), outline / file structure PR #711 (ranbeuer, draft). `dart/textDocument/publishOutline` (notification), by contrast, is LSP-only + `initializationOptions` → not reachable via LoL |
| Go to Symbol (`workspaceSymbol`)                                 | `workspace/symbol`                                           | ✅     | —     | `search.findTopLevelDeclarations` among others  |                                                                                                                                                                                                                                                                        |
| ~~Find Usages (`findReferences`)~~                               | `textDocument/references`                                    | ✅     | #396  | `search.findElementReferences`                  | **Taken → table A:** PR #623 (ranbeuer) since 2026-08-21. Our verification: LSP + PSI targets both land in the "Choose target" popup                                                                                                                                   |
| Call/type hierarchy (`callHierarchy`, `typeHierarchy`) | 6 methods (`prepare…`, `sub/supertypes`, `in/outgoingCalls`) | ✅ | #403 | `search.getTypeHierarchy` (21 files) | ✅ merged (PR #627) |
| Go to Implementation (⌘⌥B)                                       | `textDocument/implementation`                                | ✅     | #404  | `DefinitionsScopedSearch`                       | **The JB client has NO customizer for this** (only `LspDynamicCapabilities` bookkeeping) → client work needed (patch the vendored code, workflow #452)                                                                                                                 |
| Rename (`rename`) | `prepareRename`, `rename` | ✅ | #407 | `edit.getRefactoring` RENAME | Shared with LoL since dart-lang/sdk@3e33bc81443 (3.14.0-252.0.dev). **Ours: go from helin (2026-10-06), next after #719:** keep IntelliJ's rename UI (in-place vs. dialog per the IDE setting), swap only the backend via a `RenamePsiElementProcessor` (must coexist with #638's `DartLspRenamePsiElementProcessor` for files); local variables over LSP too; modal *Rename Anyway* prompt in the same PR; rename tricks implemented, then decided; `renameFilesWithClasses` → #717, `updateImportsOnRename` → #718 |
| ~~Completion (`completion`)~~ | `completion`, `completionItem/resolve` | ✅ | #399 | `completion.getSuggestions2` + a lot of ranking | **Taken → table A:** merged 2026-10-05 (PR #624) |
| Syntax highlighting (`semanticTokens`) | `semanticTokens/full`, `/range` | ✅ | #401 | `analysis.highlights` subscription | PR #640 (juan-mora-google) open; shared with LoL since `90b1c38b10f`; performance-relevant |
| Extend/shrink selection (`selectionRange`) | `textDocument/selectionRange` | ✅ | — | `DartWordSelectionHandler` + PSI (works today) | **Merge** the handlers (the platform collects all `ExtendWordSelectionHandler`s) → gating via `shouldAskServerForSelectionRange`; the win = new syntax the IntelliJ grammar doesn't know yet. Shared with LoL since `3e33bc81443` |
| Folding (`foldingRange`) | `textDocument/foldingRange` | ✅ | — | `DartFoldingBuilder` (PSI) | Shared with LoL since `3e33bc81443`; would likely settle the folding bugs #96/#150/#151/#161 |

## D. Custom methods / file operations (no standard customizer)

| Feature                    | Method                                                  | LoL?   | Notes                                                                                                                                        |
|----------------------------|---------------------------------------------------------|--------|----------------------------------------------------------------------------------------------------------------------------------------------|
| Go to Super (⌘U)           | `dart/textDocument/super`                               | ✅     | Legacy exists via PSI/DAS; no JB customizer → needs its own action wiring                                                                    |
| Fix imports on move/rename | `workspace/willRenameFiles` | ✅ | ✅ merged (PR #638, helin24; #635). `dart.updateImportsOnRename` has no effect over LoL (it only controls a registration the legacy server doesn't do); #638 always updates imports. An option for it → #718 |
| Closing labels | `dart/textDocument/publishClosingLabels` (notification) | ✅ | ✅ merged (PR #663, #400): enabled by the client capability `experimental.closingLabels` in `setClientCapabilities`, SDK gate 3.14.0-219.0.dev |
| Inline values (debugger)   | `textDocument/inlineValue`                              | ✅     | No JB customizer → not usable right now; just noting it                                                                                      |

## E. No LSP equivalent in the server (SDK work first, or drop)

| Feature                   | Legacy                           | Notes                                                         |
|---------------------------|----------------------------------|---------------------------------------------------------------|
| Postfix templates (#405)  | `edit.getPostfixCompletion` etc. | Missing from the LSP README; VS Code doesn't have the feature |
| Complete statement (#406) | `edit.getStatementCompletion`    | Same                                                          |

## Suggested order (our proposal)

1. **B without SDK work:** `documentColor`, `dart/textDocument/imports` (#582) — global, no gating, visible benefit.
2. **C with ✅ LoL that close bugs:** formatting, signatureHelp, documentSymbol, workspaceSymbol.
3. ~~**Small SDK sharing CLs as a pipeline**~~ — obsolete since dart-lang/sdk@3e33bc81443 shared all remaining handlers
   (2026-09-21); foldingRange, selectionRange, documentLink only need the plugin side now (SDK gate 3.14.0-252.0.dev).
4. Big chunks per the maintainers' plan: ~~completion~~ (PR #624), ~~hierarchy~~ (PR #627), ~~find usages~~ (PR #623),
   implementation (client work!).

## Questions for Helin (in addition to Q0–Q13 in our internal OPEN-QUESTIONS-maintainers.md)

- Is there an internal roadmap/ordering for the 16 open #207 sub-issues — where do external PRs help most without
  colliding?
- Should features without an `[lsp]` issue (documentColor, folding, formatting, selectionRange, …) get their own issues?
  Filed by us or by you?
- Are SDK sharing CLs (pattern `7c18d1fa0e5`) welcome, or is the plan to move to the real LSP server mid-term (in which
  case sharing CLs only pay off for short-term work)?
- ~~`initializationOptions`/`dart.*` config over LoL (Q13)~~ — resolved: closing labels via a client capability (#663),
  `dart.*` config via `workspace/configuration` (#679, SDK 3.14.0-258.0.dev).
- Postfix/complete statement: file an SDK feature request, or drop them with the LSP switch?
