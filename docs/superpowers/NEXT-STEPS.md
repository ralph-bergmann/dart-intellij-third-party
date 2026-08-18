# Nächste Schritte (Stand: 2026-08-18)

## Erledigt

- **Teil 1** (Rename `DartInlayHintsProvider` → `DartClosingLabelsInlayHintsProvider`):
  [PR #551](https://github.com/flutter/dart-intellij-third-party/pull/551) — **gemergt 2026-08-10**
  (`434c86f6`).
- **Teil 2** (LSP Read/Write-Highlighting): [PR #552](https://github.com/flutter/dart-intellij-third-party/pull/552)
  — **gemergt 2026-08-10** (`d3d9e7bf`). Ist bereits in Release 508.1.0.
- **SDK-Prerequisites für Teil 3 sind erledigt (2026-08-17):** unser CL ist gelandet —
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
  - **Nicht unsere:** codeAction #520 (helin24, PR #526), publishDiagnostics PR #612 (+ #441),
    setClientCapabilities PR #614, Analytics #374/#385, DAP #479.
- **Fragenkatalog** `OPEN-QUESTIONS-maintainers.md` (Q0–Q12, zum Einfügen in die Issues).
- **Plan für Stack 1** `plans/2026-08-18-lsp-endpoint-stack-1.md` (ersetzt Teil 3 des alten Plans;
  PR A = Inlay Hints auf `main`, PR B = typeDefinition gestapelt auf PR A).

## Als Nächstes

1. **Docs reviewen** (diese drei Dateien) und die Fragen posten — zuerst Q0 (Stacked PRs ok? was
   sollen wir übernehmen?) auf #207, dann Q1 (#396), Q2 (Usage Count — als Kommentar auf #396 oder
   neues Issue), Q3 (#400). Antworten in `OPEN-QUESTIONS-maintainers.md` eintragen (Status-Legende).
2. **Stack 1 umsetzen** (neue Claude-Session, `superpowers:subagent-driven-development`, Plan
   `plans/2026-08-18-lsp-endpoint-stack-1.md`):
   - Vorher prüfen, ob helin24s PRs #612/#614 gemergt sind (`gh pr view 612 --repo flutter/dart-intellij-third-party --json state,mergedAt`);
     wenn ja, `main` ziehen — beide berühren `DartBridgeLspServer.kt`/`DartBridgeLspServerTest.kt`.
     **PR B (typeDefinition) braucht #614** (`buildLspCapabilities`); Fallback steht im Plan (Task 5).
   - PR A: Tasks 1–3 (Bridge-Forwarding TDD → Version-Gate/LspMethod/Customizer/CHANGELOG →
     Sandbox mit Dev-SDK ≥ 3.14.0-139.0.dev, Verifier, Code-Review, PR).
   - PR B: Tasks 4–7, Branch von `lsp-inlay-hints`, `gh pr create --base lsp-inlay-hints`.
   - SDD-Ledger: `.superpowers/sdd/2026-07-30-dart-inlay-hints/progress.md` weiterführen (Teil 3 =
     PR A), für PR B einen eigenen Abschnitt anlegen.
   - Projektgedächtnis beachten: `lsp-feature-testing` (Cache-/DAS-Restart-Fallstricke!),
     `verifier-baselines`, `repo-git-quirks`, `build-environment`.
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
- Lokal nicht committet und bewusst liegen gelassen: `.gitignore` (+`.idea`) — ist Ralphs lokale
  Änderung, gehört nicht in die Planungs-Commits.

## Offene Aufräumpunkte

- 2026.2-EAP-Inkompatibilität (`PsiTreeElementBase` weg, Structure View) als eigenes Issue/Fix —
  betrifft `main`, nicht unsere PRs (und wäre durch #402/documentSymbol mittelfristig obsolet).
- `gradlew.bat`-Zeilenenden-Normalisierung als Upstream-Housekeeping.
- Follow-up-Ideen aus den Reviews: `requireNotNull`-Sweep in `DartBridgeLspServerTest`,
  TypeToken-Import-Konsolidierung (inzwischen importiert, s. `documentHighlight`), ggf. Versions-Gate
  für documentHighlight analog `isLspNavigationEnabled`.
- Branch `DartInlayHints` nach den heutigen Commits noch **nicht gepusht** (`origin/DartInlayHints`
  hinkt hinterher) — bei Gelegenheit `git push` (Fast-Forward, kein Force nötig).
