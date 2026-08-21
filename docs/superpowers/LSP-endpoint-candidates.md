# LSP-Endpoint-Kandidaten — Vorbereitung Call mit Helin (Fr 2026-08-21)

**Zweck:** Vollständige Landkarte, welche LSP-Endpoints das Plugin noch anbinden könnte —
als Gesprächsgrundlage für den Call zur Migrations-Planung der Maintainer.
**Bewusst noch keine Issues angelegt**; je nach Ausgang des Calls werden Zeilen aus
den Tabellen B–D zu Issues (oder eben nicht).

**Status-Update 2026-08-21 (GitHub erneut geprüft):**

- PR #614 (Client Capabilities, helin24) ist **gemergt** (2026-08-20).
- **NEU:** PR #623 (ranbeuer, 2026-08-21) implementiert Find Usages via
  `textDocument/references` mit Experimental-Flag-Gating → Zeile in Tabelle C ist vergeben.
- Inlay Hints Stufe 2 ist jetzt als #622 + dart-lang/sdk#64101 angelegt (kreuzverlinkt).

Quellen (alle am 2026-08-19 verifiziert):

- **JB-Client** = gebündelter JetBrains-LSP-Client (`thirdPartySrc/platform-lsp`,
  26 Customizer in `LspCustomization.kt`); Plugin-Status aus
  `DartLspServerDescriptor.kt:105-146` (was auf `…Disabled` steht, ist Kandidat).
- **LoL?** = Handler steht in `sharedHandlerGenerators` in
  `pkg/analysis_server/lib/src/lsp/handlers/handler_states.dart` auf SDK `origin/main`
  (`9da766e4c31`, 2026-08-19) → über LSP-over-Legacy (`lsp.handle`) erreichbar.
  ✗ = nur echter LSP-Server → braucht erst einen Share-CL im SDK
  (Vorlage: unser Inlay-Hint-CL `7c18d1fa0e5`, Plan `plans/2026-07-30-sdk-share-inlay-hint-handler.md`).
- Server-Support laut `pkg/analysis_server/tool/lsp_spec/README.md`.
- Gating nach dem #615-Prinzip: **kein Legacy-Pfad → global aktivieren, kein Flag;
  Legacy-Pfad vorhanden → Experimental-Flag als Legacy↔LSP-Schalter.**

## A. Erledigt / in Arbeit (nicht anfassen)

| Feature | LSP-Methode | Stand |
|---|---|---|
| Hover | `textDocument/hover` | ✅ merged (#291) |
| Go to Declaration (⌘B) | `textDocument/definition` | ✅ merged (#398 / PR #539) |
| Read/Write-Highlighting | `textDocument/documentHighlight` | ✅ merged (PR #552) |
| Go to Type Declaration (⌘⇧B) | `textDocument/typeDefinition` | PR #615 (helin24) offen; unser #618 als Duplikat geschlossen. Fixt #237 + #580 (Cross-Link fehlt noch) |
| Inlay Hints | `textDocument/inlayHint` | PR #617 (wir) ready for review; SDK-Gate 3.14.0-139.0.dev; Stufe 2: #622 + sdk#64101 |
| Code Actions | `textDocument/codeAction` + `executeCommand` + `applyEdit` | #520, helin24 PR #526 in Arbeit (Draft) |
| Diagnostics | `textDocument/publishDiagnostics` | PR #612 (helin24) offen, Review ausstehend (+#441); Issue #292 bereits 2026-05 geschlossen |
| Client Capabilities | `initialize`-Umbau | ✅ merged 2026-08-20 (PR #614) |
| Find Usages (`findReferences`) | `textDocument/references` | **NEU:** PR #623 (ranbeuer) offen seit 2026-08-21, mit Experimental-Flag-Gating; fixt #396 |

## B. Kandidaten OHNE Legacy-Pfad → global, kein Gating (geringste Review-Reibung)

| Feature (Customizer) | LSP-Methode(n) | LoL? | Issue | Bemerkung |
|---|---|---|---|---|
| Farb-Preview + Picker (`documentColor`) | `textDocument/documentColor`, `colorPresentation` | ✅ | — | Gutter-Farbchips für `Color(…)` — sichtbarer Flutter-Gewinn, JB-Client fertig |
| Go to Imports (`—`, custom) | `dart/textDocument/imports` | ✅ | **#582 existiert** | Custom-Methode: kein Customizer, braucht eigene Action im Plugin |
| Code Lens (`codeLens`) | `textDocument/codeLens` | ✅ | — | Server liefert derzeit v.a. Augmentation-Links; Usage-Count hat DanTup serverseitig abgelehnt (Dart-Code#1605) → Nutzen klein, aber Kosten auch |
| Document Links (`documentLink`) | `textDocument/documentLink` | **✗** | — | Braucht SDK-Share-CL; Nutzen moderat (klickbare URIs in Kommentaren) |

## C. Kandidaten MIT Legacy-Pfad → Experimental-Flag (Legacy↔LSP-Schalter)

| Feature (Customizer) | LSP-Methode(n) | LoL? | Issue | Legacy im Plugin | Bemerkung |
|---|---|---|---|---|---|
| Formatting (`formatting`, `onTypeFormatting`, `optimizeImports`) | `formatting`, `rangeFormatting`, `onTypeFormatting` | ✅ | — | `edit.format`, `edit.organizeDirectives` | Adressiert Formatter-Schmerzen #35/#152/#153/#309 |
| Parameter Info (`signatureHelp`) | `textDocument/signatureHelp` | ✅ | — | `ParameterInfoHandler` (PSI) | ⌘P-Popup |
| Structure View / Outline (`documentSymbol`) | `textDocument/documentSymbol` | ✅ | #402 | `analysis.outline`-Subscription | `dart/textDocument/publishOutline` (Notification) ist dagegen LSP-only + `initializationOptions` → nicht über LoL |
| Go to Symbol (`workspaceSymbol`) | `workspace/symbol` | ✅ | — | `search.findTopLevelDeclarations` u.a. | |
| ~~Find Usages (`findReferences`)~~ | `textDocument/references` | ✅ | #396 | `search.findElementReferences` | **Vergeben → Tabelle A:** PR #623 (ranbeuer) seit 2026-08-21. Unsere Verifikation: LSP- + PSI-Targets landen im „Choose target“-Popup |
| Call/Type Hierarchy (`callHierarchy`, `typeHierarchy`) | 6 Methoden (`prepare…`, `sub/supertypes`, `in/outgoingCalls`) | ✅ | #403 | `search.getTypeHierarchy` (21 Dateien) | Achtung Provider-Vorrang: Darts `language="Dart"` schlägt die LSP-Provider (`language=""`, `order="last"`) |
| Go to Implementation (⌘⌥B) | `textDocument/implementation` | ✅ | #404 | `DefinitionsScopedSearch` | **JB-Client hat KEINEN Customizer dafür** (nur `LspDynamicCapabilities`-Buchhaltung) → Client-Arbeit nötig (vendored patchen, Workflow #452) |
| Rename (`rename`) | `prepareRename`, `rename` | **✗** | #407 | `edit.getRefactoring` RENAME | SDK-Share-CL nötig; Vorsicht: Datei-Umbenennung bei Klassen-Rename (`renameFilesWithClasses` ist LSP-only-Config, s. Q13) |
| Completion (`completion`) | `completion`, `completionItem/resolve` | **✗** | #399 | `completion.getSuggestions2` + großes Ranking | Größter Brocken; Share-CL vermutlich nicht trivial (Doc-State) — Maintainer-Plan abwarten |
| Syntax-Highlighting (`semanticTokens`) | `semanticTokens/full`, `/range` | **✗** | #401 | `analysis.highlights`-Subscription | SDK-Share-CL nötig; Perf-relevant |
| Extend/Shrink Selection (`selectionRange`) | `textDocument/selectionRange` | **✗** | — | `DartWordSelectionHandler` + PSI (funktioniert heute) | Handler **mergen** (Plattform sammelt alle `ExtendWordSelectionHandler`) → Gating via `shouldAskServerForSelectionRange`; Gewinn = neue Syntax, die die IJ-Grammatik nicht kennt. Handler braucht nur `requireUnresolvedUnit` → Share-CL klein |
| Folding (`foldingRange`) | `textDocument/foldingRange` | **✗** | — | `DartFoldingBuilder` (PSI) | Share-CL nötig; würde Folding-Bugs #96/#150/#151/#161 wohl erledigen |

## D. Custom-Methoden / Datei-Operationen (kein Standard-Customizer)

| Feature | Methode | LoL? | Bemerkung |
|---|---|---|---|
| Go to Super (⌘U) | `dart/textDocument/super` | ✅ | Legacy via PSI/DAS vorhanden; kein JB-Customizer → eigene Action-Verdrahtung |
| Imports beim Verschieben/Umbenennen fixen | `workspace/willRenameFiles` | ✅ | Würde #95/#187 lösen; JB-Client-Support für File-Ops unklar → untersuchen |
| Closing Labels | `dart/textDocument/publishClosingLabels` (Notification) | **✗** | #400; gated über `initializationOptions` → über LoL unerreichbar, braucht SDK-Opt-in analog `d42063aac44`/sdk#64021 (Q13-Komplex) |
| Inline Values (Debugger) | `textDocument/inlineValue` | ✅ | Kein JB-Customizer → derzeit nicht nutzbar; nur notieren |

## E. Kein LSP-Äquivalent im Server (SDK-Arbeit zuerst oder streichen)

| Feature | Legacy | Bemerkung |
|---|---|---|
| Postfix-Templates (#405) | `edit.getPostfixCompletion` etc. | Fehlt in der LSP-README; VS Code hat das Feature nicht |
| Complete Statement (#406) | `edit.getStatementCompletion` | dito |

## Vorgeschlagene Reihenfolge (unser Vorschlag, im Call abgleichen)

1. **B ohne SDK-Arbeit:** `documentColor`, `dart/textDocument/imports` (#582) — global, kein Gating, sichtbarer Nutzen.
2. **C mit ✅-LoL, die Bugs schließen:** formatting, signatureHelp, documentSymbol, workspaceSymbol.
3. **Kleine SDK-Share-CLs als Pipeline** (Vorlage vorhanden): foldingRange, selectionRange, documentLink, semanticTokens, rename — je CL + Plugin-PR mit Versions-Gate.
4. Große Brocken nach Maintainer-Plan: completion, hierarchy, ~~find usages~~ (PR #623 offen), implementation (Client-Arbeit!).

## Fragen an Helin (Ergänzung zu OPEN-QUESTIONS-maintainers.md Q0–Q13)

- Gibt es eine interne Roadmap/Reihenfolge für die 16 offenen #207-Sub-Issues — wo helfen externe PRs am meisten, ohne zu kollidieren?
- Sollen Features ohne `[lsp]`-Issue (documentColor, folding, formatting, selectionRange, …) eigene Issues bekommen? Von uns angelegt oder von euch?
- Sind SDK-Share-CLs (Muster `7c18d1fa0e5`) willkommen, oder ist der Plan, mittelfristig auf echtes LSP umzustellen (dann lohnen sich Share-CLs nur für Kurzfristiges)?
- `initializationOptions`/`dart.*`-Config über LoL (Q13) — blockiert closing labels, publishOutline, rename-Config.
- Postfix/Complete Statement: SDK-Feature-Request oder beim LSP-Umstieg streichen?
