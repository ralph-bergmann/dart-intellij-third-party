# Nächste Schritte (Stand: Feierabend 2026-07-30)

## Erledigt heute

- **Teil 1** (Rename): [PR #551](https://github.com/flutter/dart-intellij-third-party/pull/551) — offen, CLA grün, 1 Commit `3c1ca547`.
- **Teil 2** (LSP Read/Write-Highlighting): [PR #552](https://github.com/flutter/dart-intellij-third-party/pull/552) — offen, CLA grün, 3 Commits (HEAD `8fbea7fc`). Sandbox-Checkliste vollständig verifiziert (Kernfall mit Dart 3.13.0-282.3.beta).
- Alle Reviews (Task-Reviews, Whole-Branch-Reviews, Repo-Review-Skill) sauber; Verifier: keine neuen Issues vs. Baseline (Exit 1 kommt vom vorbestehenden, baselinierte 262-EAP-Problem).

## Morgen / als Nächstes

1. **PR-Feedback**: Reviews auf #551/#552 beantworten bzw. einarbeiten (Claude-Session: `superpowers:receiving-code-review`; Worktrees liegen noch unter `.claude/worktrees/`).
2. **Teil 3 vorbereiten — SDK-Prerequisites (P1/P2, macht Ralph im `../sdk`-Checkout):**
   - P1: `InlayHintHandler` als Shared Handler, nach `docs/superpowers/plans/2026-07-30-sdk-share-inlay-hint-handler.md`.
   - ⚠️ `../sdk` ist ein **plain git clone ohne gclient/depot_tools** — zum Bauen/Testen erst `gclient`-Setup (zieht viele GB; Plattenplatz prüfen, aktuell ~37 GiB frei).
   - P2: nach Landung erste Dev-Version bestimmen:
     `git -C ../sdk tag --contains <sha> | sort -V | head -1` → Wert für `MIN_LSP_INLAY_HINTS_SDK_VERSION`.
3. **Teil 3 umsetzen** (neue Claude-Session, `superpowers:subagent-driven-development`):
   - SDD-Ledger: `.superpowers/sdd/2026-07-30-dart-inlay-hints/progress.md` (weiterführen, nicht neu anlegen).
   - Task 3.1 (Bridge-Forwarding, TDD) geht schon **vor** P2; Merge erst, wenn P2 die Versionskonstante liefert.
   - Projektgedächtnis beachten: `lsp-feature-testing` (Cache-/DAS-Restart-Fallstricke!), `verifier-baselines`, `repo-git-quirks`.
4. **Nach Merge von #551/#552**: Worktrees + lokale Branches aufräumen; SDD-Workspace erst nach Teil 3 löschen.

## Wichtige Regeln (heute gelernt, gelten auch für Teil 3)

- **Keine `Co-Authored-By: Claude …`-Trailer** in diesem Repo — Googles CLA-Bot behandelt Co-Autoren als Contributor → Check schlägt fehl. (Übersteuert die Commit-Vorlage im Plan!)
- Author/Committer: `ralph@dasralph.de` (repo-lokal gesetzt; globale Config hat die adesso-Adresse).
- `git add` immer mit expliziten Dateien (gradlew.bat-CRLF-Falle), Gradle nur mit `JAVA_HOME` = IntelliJ-JBR.

## Offene Kleinigkeiten (optional)

- 2026.2-EAP-Inkompatibilität (`PsiTreeElementBase` weg, Structure View) als eigenes Issue/Fix — betrifft `main`, nicht unsere PRs.
- `gradlew.bat`-Zeilenenden-Normalisierung als Upstream-Housekeeping.
- Follow-up-Ideen aus den Reviews: `requireNotNull`-Sweep in `DartBridgeLspServerTest`, TypeToken-Import-Konsolidierung, ggf. Versions-Gate für documentHighlight analog `isLspNavigationEnabled`.
