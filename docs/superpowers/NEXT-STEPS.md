# Nächste Schritte (Stand: 2026-08-19)

## Erledigt

- **Teil 1** (Rename `DartInlayHintsProvider` → `DartClosingLabelsInlayHintsProvider`):
  [PR #551](https://github.com/flutter/dart-intellij-third-party/pull/551) — **gemergt 2026-08-10**
  (`434c86f6`).
- **Teil 2** (LSP Read/Write-Highlighting): [PR #552](https://github.com/flutter/dart-intellij-third-party/pull/552)
  — **gemergt 2026-08-10** (`d3d9e7bf`). Ist bereits in Release 508.1.0.
- **SDK-Prerequisites für Teil 3 sind erledigt (2026-08-17):** der CL ist gelandet —
  dart-lang/sdk `7c18d1fa0e5` ([CL 536565](https://dart-review.googlesource.com/c/sdk/+/536565),
  schließt [sdk#64061](https://github.com/dart-lang/sdk/issues/64061)). Erster Dev-Tag, der den Commit
  enthält: **`3.14.0-139.0.dev`** (138 enthält ihn nicht) → das ist `MIN_LSP_INLAY_HINTS_SDK_VERSION`.
  Handoff-Doku `plans/2026-07-30-sdk-share-inlay-hint-handler.md` ist auf „DONE" gesetzt und bleibt
  als Vorlage für künftige SDK-Aufgaben. Achtung: der SDK-Checkout steht noch auf dem lokalen
  CL-Branch (`75e7d44de95`) — vor neuer SDK-Arbeit `git checkout main && git pull origin main`.
- **Scope-Analyse zu #207 (2026-08-18):** `specs/2026-08-18-lsp-migration-scope-analysis.md` —
  alle 16 offenen Sub-Issues von #207 klassifiziert (drei Gruppen, PR-Stack-Zuordnung, Belege im
  Anhang). Kernaussagen:
  - **Jetzt umsetzbar (Stack 1):** Inlay Hints (#159, Teil 3) + Go To Type Declaration (#580) — beide
    reine „Endpoint einschalten"-Änderungen ohne Legacy-Gating.
  - **Erst Maintainer fragen:** Find Usages #396 (PSI-vs-LSP-Target-Popup), Usage Count (DanTup
    lehnt eine Server-Codelens ab → IDE-seitiger Provider?), Closing Labels #400 (SDK sendet
    `publishClosingLabels` nicht über LSP-over-Legacy → Opt-in-Mechanismus fehlt), Outline #402,
    Hierarchie #403, Implementations #404, Completion #399, Semantic Tokens #401, Rename #407,
    Postfix/Complete-Statement #405/#406 (kein LSP-Protokoll).
  - **Nicht meine (Ralph):** codeAction #520 (helin24, PR #526), publishDiagnostics PR #612 (+ #441),
    setClientCapabilities PR #614, Analytics #374/#385, DAP #479.
- **Fragenkatalog** `OPEN-QUESTIONS-maintainers.md` (Q0–Q13, zum Einfügen in die Issues). Q13 und
  die Ergänzungen in Q3/Q4/Q8/Q9/Q12 decken die Server-Optionen aus
  `pkg/analysis_server/tool/lsp_spec/README.md` ab (Initialization Options wie `closingLabels`/
  `outline`, `dart.*`-Konfiguration wie `renameFilesWithClasses`, `inlayHints`-Kategorien) — beides
  ist über LSP-over-Legacy nicht erreichbar, daher zwei Fragen: SDK-Transportweg und IntelliJ-Settings-UI.
- **Plan für Stack 1** `plans/2026-08-18-lsp-endpoint-stack-1.md` (ersetzt Teil 3 des alten Plans;
  PR A = Inlay Hints auf `main`, PR B = typeDefinition gestapelt auf PR A).

## Als Nächstes

1. ✅ **Fragen sind gepostet (2026-08-18, 13 Kommentare, alle in Ich-Form):** zwei auf #207
   (Scope/Stacked PRs/Usage Count/untracked Endpoints; Server-Optionen & Settings-UI, cc DanTup) und
   je einer auf #396, #400 (cc DanTup), #402, #401, #403, #404, #399, #407, #405, #406, #441 — Links
   stehen in den Überschriften von `OPEN-QUESTIONS-maintainers.md`. **Jetzt: Antworten abwarten**,
   regelmäßig `gh issue view <n> --repo flutter/dart-intellij-third-party --comments` prüfen und die
   Antworten in `OPEN-QUESTIONS-maintainers.md` eintragen (✅ + Datum + Kurzfassung). Bei einem „ja“ zu
   Q3 folgt ein dart-lang/sdk-Issue (Vorlage: `plans/2026-07-30-sdk-share-inlay-hint-handler.md`).
2. ✅ **Stack 1 ist implementiert und als zwei Draft-PRs offen (2026-08-19):**
   - **PR A** [#617](https://github.com/flutter/dart-intellij-third-party/pull/617) „Show LSP inlay
     hints for Dart as an experimental feature" — Branch `lsp-inlay-hints` (`1826e3cc`, 3 Commits auf
     upstream `main` `fb835401`). Tasks 1–3 erledigt inkl. Verifier (keine neuen Baseline-Zeilen),
     Repo-Code-Review (0 MUST-FIX), Final-Review + Fix-Wave (Fehlerantworten von DAS für
     `textDocument/inlayHint` werden jetzt zu „keine Hints" + Info-Log statt IDE-Fehler, weil der
     Inlay-Hint-Pfad des LSP-Clients Exceptions nicht fängt). **Offen: der manuelle Sandbox-Check
     (Task 3 Step 1)** mit einem Dev-SDK ≥ 3.14.0-139.0.dev — danach Screenshots in den PR-Body,
     Draft aufheben. Im PR-Body steht explizit: Feature ist per Default an (experimenteller
     LSP-Schalter defaultet auf `true`), alle Kategorien, kein eigener Aus-Schalter — Maintainer
     sollen sagen, ob sie das so wollen.
   - **PR B** [#618](https://github.com/flutter/dart-intellij-third-party/pull/618) „Go to Type
     Declaration via LSP typeDefinition" — Branch `lsp-type-definition` (`ca95e6d0`, 3 Commits auf
     `lsp-inlay-hints`), **abhängiger PR gegen `main`** (enthält A's Commits bis A gemergt ist).
     Tasks 4, 6, 7 (Step 2–3) erledigt. **Offen: Task 5** (`typeDefinition.linkSupport` in
     `buildLspCapabilities` + Test) — **blockiert auf helin24s #614**; bis dahin ist das Feature
     nicht funktionsfähig (DAS liefert ohne linkSupport ein nacktes `Location`, der Bridge-Code
     erwartet `List<LocationLink>`) und der Code-Kommentar in `typeDefinition(...)` verweist auf das
     noch nicht existierende `buildLspCapabilities`. Nach #614: `main` ziehen, beide Branches
     rebasen, Task 5 committen, Sandbox-Check (Task 7 Step 1), Draft aufheben.
   - **GitHub-Stacks gehen nicht aus einem Fork** („Cross-fork stacks are not supported",
     [Docs](https://docs.github.com/en/pull-requests/reference/stacked-pull-requests)) — deshalb klassischer
     abhängiger PR; Korrektur ist auf #207 gepostet
     ([Kommentar](https://github.com/flutter/dart-intellij-third-party/issues/207#issuecomment-5341395635)).
   - Worktree: `.claude/worktrees/lsp-inlay-hints` (aktuell auf `lsp-type-definition`). SDD-Ledger:
     `.superpowers/sdd/2026-08-18-lsp-endpoint-stack-1/progress.md` (bleibt bis Task 5 + Sandbox
     erledigt sind). Sandbox-Fallstricke: Projektgedächtnis `lsp-feature-testing`.
   - Beim Rebase nach #612 (publishDiagnostics) aufpassen: berührt `DartBridgeLspServer(.kt/Test.kt)`.
3. **Nach den Antworten:** Stack 2 („Navigation family": #396, ggf. #404/#403) bzw. Stack 3
   („Push-Notifications über LoL": #400, #402) planen — jeweils zuerst SDK-Issue/CL, dann Plugin.
   Für SDK-Änderungen die Handoff-Vorlage `plans/2026-07-30-sdk-share-inlay-hint-handler.md` kopieren.

## Regeln (gelten für Plugin **und** SDK-Arbeit)

- **Alle Entscheidungen und Änderungen werden validiert** — gegen Issues des jeweiligen Repos
  und/oder gegen den echten Code. Es wird nichts geraten, pauschal angenommen oder erfunden.
- **An vorhandenem Code, Doku und Formatierung orientieren** und vorhandene Klassen/Methoden/Helper
  wiederverwenden — auch dann, wenn Vorhandenes suboptimal ist. In dem Fall: eigene md-Datei mit
  Änderungsvorschlägen schreiben, den Code aber in Ruhe lassen. Grund: Reviewer sollen Bekanntes
  sehen und sich nicht in Neues einarbeiten müssen.
- **Commits nur von `Ralph Bergmann <ralph@dasralph.de>`**, insbesondere **keine
  `Co-Authored-By:`-Trailer** — Googles CLA-Bot behandelt Co-Autoren als eigene Contributor, der
  Check schlägt dann fehl. (Übersteuert Commit-Vorlagen in den Plan-Dokumenten.)
- `git add` immer mit expliziten Dateien (gradlew.bat-CRLF-Falle), Gradle nur mit `JAVA_HOME` = IntelliJ-JBR.
- **Max. zwei offene PRs**, ähnliche Änderungen als Stacked PRs (PR B mit `--base <Branch von PR A>`),
  damit Reviewer ein Muster einmal sehen. Nach dem Merge von A: `gh pr edit <B> --base main` + Rebase.
- Nichts unter `third_party/thirdPartySrc/` ändern (Code-Review-Skill des Repos → `[MUST-FIX]`);
  Ausnahme nur mit Owner-Freigabe (siehe Plan Task 5 Fallback).
- Diese Regeln stehen als „Ground rules" auch am Anfang der SDK-Handoff-Doku, damit der dortige
  Agent sie ohne diesen Kontext hat.

## Am 2026-08-18 erledigt (Session-Log)

- SDK-Checkout gefetcht (`origin/main` = `00c42422917`), Landung des CL und ersten Dev-Tag verifiziert.
- Issue #207 samt aller 27 Sub-Issues (16 offen) und der Kommentare gelesen; helin24s Draft-PRs
  #526/#612/#614 (Dateilisten) und die gemergten #539/#552 als Muster ausgewertet.
- SDK-Handler-Listen (`handler_states.dart` bei `origin/main`, 3.3.0, 3.5.0) und den
  publishDiagnostics-Präzedenzfall `d42063aac44` (sdk#64021) analysiert.
- Plattform-Verhalten in den IntelliJ-Community-Quellen verifiziert (Implicit-Reference-Vorrang bei
  Go-to-(Type-)Declaration; Find-Usages-Target-Popup bei PSI + LSP; Hierarchy-Provider-Vorrang).
- Vier Dokumente geschrieben/aktualisiert (Spec, Fragen, Plan Stack 1, Status in den alten Plänen).
- Alle Fragen auf GitHub gepostet (als ralph-bergmann, Wortlaut für GitHub angepasst: Ich-Form, keine
  internen Verweise; Querverweise zeigen auf die jeweiligen Kommentare). Q0 enthält jetzt den Hinweis,
  dass Stacked PRs seit 2026-07-30 in Public Preview sind (GitHub-Changelog + Docs-Link).
- Nachtrag auf Ralphs Hinweis: `pkg/analysis_server/tool/lsp_spec/README.md` (bei `origin/main`)
  ausgewertet — Initialization Options, `dart.*`-Konfiguration, Method-Status-Tabelle, Client
  Commands (`dart.goToLocation`) — und als Q13 plus Ergänzungen in Q2/Q3/Q4/Q8/Q9/Q12 eingearbeitet;
  Spec §2.1/§7/§10 entsprechend erweitert.
- `.gitignore` (+`.idea`) hat Ralph selbst committet (`5c4c5977`).
- 2026-08-19: Fork-`main` auf upstream `fb835401` fast-forwarded; Stack 1 per
  subagent-driven-development umgesetzt (Implementer/Reviewer-Subagenten, Final-Review, Fix-Wave);
  beide PRs als Draft geöffnet; Kommentare/Korrektur auf #207 gepostet.

## Offene Aufräumpunkte

- 2026.2-EAP-Inkompatibilität (`PsiTreeElementBase` weg, Structure View) als eigenes Issue/Fix —
  betrifft `main`, nicht meine PRs (und wäre durch #402/documentSymbol mittelfristig obsolet).
- `gradlew.bat`-Zeilenenden-Normalisierung als Upstream-Housekeeping.
- Follow-up-Ideen aus den Reviews: `requireNotNull`-Sweep in `DartBridgeLspServerTest`,
  TypeToken-Import-Konsolidierung (inzwischen importiert, s. `documentHighlight`), ggf. Versions-Gate
  für documentHighlight analog `isLspNavigationEnabled`.
- Branch `DartInlayHints` nach den heutigen Commits noch **nicht gepusht** (`origin/DartInlayHints`
  hinkt hinterher) — bei Gelegenheit `git push` (Fast-Forward, kein Force nötig).
