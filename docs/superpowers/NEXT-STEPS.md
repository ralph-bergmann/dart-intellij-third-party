# Nächste Schritte (Stand: 2026-08-17)

## Erledigt

- **Teil 1** (Rename `DartInlayHintsProvider` → `DartClosingLabelsInlayHintsProvider`):
  [PR #551](https://github.com/flutter/dart-intellij-third-party/pull/551) — **gemergt 2026-08-10**
  (`434c86f6`). Review: helin24 erst CHANGES_REQUESTED (erwartete, dass die Klasse mit Inlay Hints
  wegfällt), nach der verifizierten Antwort approved; pq hat gemergt.
- **Teil 2** (LSP Read/Write-Highlighting): [PR #552](https://github.com/flutter/dart-intellij-third-party/pull/552)
  — **gemergt 2026-08-10** (`d3d9e7bf`).
- **SDK-Handoff-Doku** `plans/2026-07-30-sdk-share-inlay-hint-handler.md`: am 2026-08-17 vollständig
  gegen `dart-lang/sdk` main (`144f5acb5db`), gegen die Issues beider Repos und gegen die
  Review-Diskussionen der gemergten PRs neu verifiziert und überarbeitet. Jede Aussage ist jetzt mit
  Datei:Zeile / Issue / Commit belegt (Anhang „Verified facts"); noch offene Punkte sind mit
  `[verify]` markiert.
- Branch `DartInlayHints` am 2026-08-17 auf `origin/main` (`232dd0b7`) rebased — vorher 45 Commits
  im Rückstand.

## Als Nächstes

1. **SDK-Prerequisites (P1/P2), im `../sdk`-Checkout:**
   - P1: `InlayHintHandler` als Shared Handler — Anleitung: `plans/2026-07-30-sdk-share-inlay-hint-handler.md`.
     Bestätigt am 2026-08-17: die Änderung steht noch aus, und es gibt noch **kein** SDK-Issue dafür
     (Step 0 der Anleitung ist also weiterhin nötig).
   - ⚠️ `../sdk` ist ein **plain git clone ohne gclient/depot_tools**. `CONTRIBUTING.md` sagt
     ausdrücklich, dass ein reiner `git clone` keine funktionsfähige Umgebung ergibt → für Tests
     (Step 5) erst `gclient`-Setup (zieht viele GB; Plattenplatz prüfen). Als dokumentierte
     Alternative zum Gerrit-Upload akzeptiert das Repo GitHub-PRs, die ein copybara-Bot in CLs
     umwandelt.
   - P2: nach Landung erste Dev-Version bestimmen:
     `git -C ../sdk tag --contains <sha> | sort -V | head -1` → Wert für `MIN_LSP_INLAY_HINTS_SDK_VERSION`.
     Präzedenz im Plugin: `MIN_LSP_NAVIGATION_SDK_VERSION = "3.14.0-65.0.dev"`
     (`DartAnalysisServerService.java:184`) und `MIN_LSP_DIAGNOSTIC_SERVER_SDK_VERSION = "3.13.0-106.0.dev"`
     (`AnalysisServerDiagnosticsAction.java:27`).
2. **Teil 3 umsetzen** (neue Claude-Session, `superpowers:subagent-driven-development`):
   - SDD-Ledger: `.superpowers/sdd/2026-07-30-dart-inlay-hints/progress.md` (weiterführen, nicht neu anlegen).
   - Task 3.1 (Bridge-Forwarding, TDD) geht schon **vor** P2; Merge erst, wenn P2 die Versionskonstante liefert.
   - Projektgedächtnis beachten: `lsp-feature-testing` (Cache-/DAS-Restart-Fallstricke!), `verifier-baselines`, `repo-git-quirks`.

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
- Diese Regeln stehen als „Ground rules" auch am Anfang der SDK-Handoff-Doku, damit der dortige
  Agent sie ohne diesen Kontext hat.

## Am 2026-08-17 erledigt

- **CLA-Hygiene auf allen 5 Branch-Commits hergestellt**: vier Commits waren als
  `Ralph Bergmann <ralph.bergmann@adesso.de>` committet und trugen zusätzlich einen
  `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>`-Trailer. Autor und Committer sind jetzt
  überall `Ralph Bergmann <ralph@dasralph.de>`, alle Co-Author-Trailer sind entfernt, die
  ursprünglichen Autor-Daten (2026-07-30) und die Dateiinhalte unverändert.
- **Branches der gemergten PRs restlos entfernt**: Worktrees, lokale Branches und die
  Fork-Branches `lsp-document-highlight` / `rename-closing-labels-provider`. Vorher verifiziert:
  die Tips `088ae11f` / `cd9c4174` waren exakt die Head-SHAs der gemergten PRs #552 / #551. Beide
  PRs wurden squash-gemergt (`d3d9e7bf` / `434c86f6`), die Original-SHAs stehen also nicht in
  `main`, bleiben aber über die PRs auf GitHub einsehbar. Der SDD-Workspace bleibt bis Teil 3
  fertig ist.
- Branch nach dem History-Rewrite mit `--force-with-lease` gepusht; `origin/DartInlayHints` und
  lokal sind wieder synchron. Im Fork liegen jetzt nur noch `DartInlayHints` und `main`.

## Offene Aufräumpunkte

- Keine — der nächste Schritt ist inhaltlich (SDK-P1, siehe oben).
- 2026.2-EAP-Inkompatibilität (`PsiTreeElementBase` weg, Structure View) als eigenes Issue/Fix —
  betrifft `main`, nicht unsere PRs.
- `gradlew.bat`-Zeilenenden-Normalisierung als Upstream-Housekeeping.
- Follow-up-Ideen aus den Reviews: `requireNotNull`-Sweep in `DartBridgeLspServerTest`,
  TypeToken-Import-Konsolidierung, ggf. Versions-Gate für documentHighlight analog `isLspNavigationEnabled`.
