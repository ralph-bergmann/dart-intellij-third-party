# LSP endpoint candidates

**Purpose:** A complete map of which LSP endpoints the plugin could still wire up — as a basis for discussing the
maintainers' migration planning. **Deliberately no issues filed yet** for the new candidates; rows from tables B–D
become issues (or not).

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
| Read/write highlighting        | `textDocument/documentHighlight`                           | ✅ merged (PR #552)                                                                                                |
| Go to Type Declaration (⌘⇧B)   | `textDocument/typeDefinition`                              | PR #615 (helin24) open; our #618 closed as duplicate. Fixes #237 + #580 (cross-links still missing in the PR body) |
| Inlay hints                    | `textDocument/inlayHint`                                   | PR #617 (us) approved by helin24 (2026-08-21); SDK gate 3.14.0-139.0.dev; stage 2: #622 + sdk#64101                |
| Code actions                   | `textDocument/codeAction` + `executeCommand` + `applyEdit` | #520, helin24 PR #526 in progress (draft)                                                                          |
| Diagnostics                    | `textDocument/publishDiagnostics`                          | ✅ merged 2026-08-21 (PR #612); #441 still open; issue #292 already closed in 2026-05                              |
| Client capabilities            | `initialize` rework                                        | ✅ merged 2026-08-20 (PR #614)                                                                                     |
| Find Usages (`findReferences`) | `textDocument/references`                                  | **NEW:** PR #623 (ranbeuer) open since 2026-08-21, gated behind the experimental flag; fixes #396                  |

## B. Candidates WITHOUT a legacy path → global, no gating (least review friction)

| Feature (customizer)                     | LSP method(s)                                     | LoL?   | Issue           | Notes                                                                                                                                            |
|------------------------------------------|---------------------------------------------------|--------|-----------------|--------------------------------------------------------------------------------------------------------------------------------------------------|
| Color preview + picker (`documentColor`) | `textDocument/documentColor`, `colorPresentation` | ✅     | —               | Gutter color chips for `Color(…)` — a visible win for Flutter, JB client is ready                                                                |
| Go to Imports (`—`, custom)              | `dart/textDocument/imports`                       | ✅     | **#582 exists** | Custom method: no customizer, needs its own action in the plugin                                                                                 |
| Code Lens (`codeLens`)                   | `textDocument/codeLens`                           | ✅     | —               | Server currently mostly serves augmentation links; DanTup rejected usage counts server-side (Dart-Code#1605) → small benefit, but small cost too |
| Document Links (`documentLink`)          | `textDocument/documentLink`                       | **✗** | —               | Needs an SDK sharing CL; moderate benefit (clickable URIs in comments)                                                                           |

## C. Candidates WITH a legacy path → experimental flag (legacy↔LSP switch)

| Feature (customizer)                                             | LSP method(s)                                                | LoL?   | Issue | Legacy in the plugin                            | Notes                                                                                                                                                                                                                                                                  |
|------------------------------------------------------------------|--------------------------------------------------------------|--------|-------|-------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Formatting (`formatting`, `onTypeFormatting`, `optimizeImports`) | `formatting`, `rangeFormatting`, `onTypeFormatting`          | ✅     | —     | `edit.format`, `edit.organizeDirectives`        | Addresses formatter pain points #35/#152/#153/#309                                                                                                                                                                                                                     |
| Parameter info (`signatureHelp`)                                 | `textDocument/signatureHelp`                                 | ✅     | —     | `ParameterInfoHandler` (PSI)                    | The ⌘P popup                                                                                                                                                                                                                                                           |
| Structure view / outline (`documentSymbol`)                      | `textDocument/documentSymbol`                                | ✅     | #402  | `analysis.outline` subscription                 | `dart/textDocument/publishOutline` (notification), by contrast, is LSP-only + `initializationOptions` → not reachable via LoL                                                                                                                                          |
| Go to Symbol (`workspaceSymbol`)                                 | `workspace/symbol`                                           | ✅     | —     | `search.findTopLevelDeclarations` among others  |                                                                                                                                                                                                                                                                        |
| ~~Find Usages (`findReferences`)~~                               | `textDocument/references`                                    | ✅     | #396  | `search.findElementReferences`                  | **Taken → table A:** PR #623 (ranbeuer) since 2026-08-21. Our verification: LSP + PSI targets both land in the "Choose target" popup                                                                                                                                   |
| Call/type hierarchy (`callHierarchy`, `typeHierarchy`)           | 6 methods (`prepare…`, `sub/supertypes`, `in/outgoingCalls`) | ✅     | #403  | `search.getTypeHierarchy` (21 files)            | Watch out for provider precedence: Dart's `language="Dart"` beats the LSP providers (`language=""`, `order="last"`)                                                                                                                                                    |
| Go to Implementation (⌘⌥B)                                       | `textDocument/implementation`                                | ✅     | #404  | `DefinitionsScopedSearch`                       | **The JB client has NO customizer for this** (only `LspDynamicCapabilities` bookkeeping) → client work needed (patch the vendored code, workflow #452)                                                                                                                 |
| Rename (`rename`)                                                | `prepareRename`, `rename`                                    | **✗** | #407  | `edit.getRefactoring` RENAME                    | SDK sharing CL needed; careful: file rename on class rename (`renameFilesWithClasses` is LSP-only config, see Q13)                                                                                                                                                     |
| Completion (`completion`)                                        | `completion`, `completionItem/resolve`                       | **✗** | #399  | `completion.getSuggestions2` + a lot of ranking | Biggest chunk; a sharing CL is probably not trivial (doc state) — wait for the maintainers' plan                                                                                                                                                                       |
| Syntax highlighting (`semanticTokens`)                           | `semanticTokens/full`, `/range`                              | **✗** | #401  | `analysis.highlights` subscription              | SDK sharing CL needed; performance-relevant                                                                                                                                                                                                                            |
| Extend/shrink selection (`selectionRange`)                       | `textDocument/selectionRange`                                | **✗** | —     | `DartWordSelectionHandler` + PSI (works today)  | **Merge** the handlers (the platform collects all `ExtendWordSelectionHandler`s) → gating via `shouldAskServerForSelectionRange`; the win = new syntax the IntelliJ grammar doesn't know yet. The handler only needs `requireUnresolvedUnit` → the sharing CL is small |
| Folding (`foldingRange`)                                         | `textDocument/foldingRange`                                  | **✗** | —     | `DartFoldingBuilder` (PSI)                      | Sharing CL needed; would likely settle the folding bugs #96/#150/#151/#161                                                                                                                                                                                             |

## D. Custom methods / file operations (no standard customizer)

| Feature                    | Method                                                  | LoL?   | Notes                                                                                                                                        |
|----------------------------|---------------------------------------------------------|--------|----------------------------------------------------------------------------------------------------------------------------------------------|
| Go to Super (⌘U)           | `dart/textDocument/super`                               | ✅     | Legacy exists via PSI/DAS; no JB customizer → needs its own action wiring                                                                    |
| Fix imports on move/rename | `workspace/willRenameFiles`                             | ✅     | Would solve #95/#187; JB client support for file operations unclear → investigate                                                            |
| Closing labels             | `dart/textDocument/publishClosingLabels` (notification) | **✗** | #400; gated behind `initializationOptions` → unreachable via LoL, needs an SDK opt-in analogous to `d42063aac44`/sdk#64021 (the Q13 complex) |
| Inline values (debugger)   | `textDocument/inlineValue`                              | ✅     | No JB customizer → not usable right now; just noting it                                                                                      |

## E. No LSP equivalent in the server (SDK work first, or drop)

| Feature                   | Legacy                           | Notes                                                         |
|---------------------------|----------------------------------|---------------------------------------------------------------|
| Postfix templates (#405)  | `edit.getPostfixCompletion` etc. | Missing from the LSP README; VS Code doesn't have the feature |
| Complete statement (#406) | `edit.getStatementCompletion`    | Same                                                          |

## Suggested order (our proposal)

1. **B without SDK work:** `documentColor`, `dart/textDocument/imports` (#582) — global, no gating, visible benefit.
2. **C with ✅ LoL that close bugs:** formatting, signatureHelp, documentSymbol, workspaceSymbol.
3. **Small SDK sharing CLs as a pipeline** (template exists): foldingRange, selectionRange, documentLink,
   semanticTokens, rename — one CL + one plugin PR with a version gate each.
4. Big chunks per the maintainers' plan: completion, hierarchy, ~~find usages~~ (PR #623 open), implementation (client
   work!).

## Questions for Helin (in addition to Q0–Q13 in our internal OPEN-QUESTIONS-maintainers.md)

- Is there an internal roadmap/ordering for the 16 open #207 sub-issues — where do external PRs help most without
  colliding?
- Should features without an `[lsp]` issue (documentColor, folding, formatting, selectionRange, …) get their own issues?
  Filed by us or by you?
- Are SDK sharing CLs (pattern `7c18d1fa0e5`) welcome, or is the plan to move to the real LSP server mid-term (in which
  case sharing CLs only pay off for short-term work)?
- `initializationOptions`/`dart.*` config over LoL (Q13) — blocks closing labels, publishOutline, and the rename config.
- Postfix/complete statement: file an SDK feature request, or drop them with the LSP switch?
