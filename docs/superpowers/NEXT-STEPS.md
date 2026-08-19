# Nächste Schritte (Stand: 2026-08-19, spät)

## Wo wir stehen

| Was | Stand |
|---|---|
| Teil 1 — Rename `DartInlayHintsProvider` → `DartClosingLabelsInlayHintsProvider` | ✅ [#551](https://github.com/flutter/dart-intellij-third-party/pull/551), gemergt 2026-08-10 (`434c86f6`) |
| Teil 2 — LSP Read/Write-Highlighting | ✅ [#552](https://github.com/flutter/dart-intellij-third-party/pull/552), gemergt 2026-08-10 (`d3d9e7bf`), in Release 508.1.0 |
| SDK: `textDocument/inlayHint` als Shared Handler | ✅ dart-lang/sdk `7c18d1fa0e5` ([CL 536565](https://dart-review.googlesource.com/c/sdk/+/536565), schließt [sdk#64061](https://github.com/dart-lang/sdk/issues/64061)); erster Dev-Tag **`3.14.0-139.0.dev`** = `MIN_LSP_INLAY_HINTS_SDK_VERSION` |
| Teil 3 — LSP Inlay Hints (#159) | 🟡 [#617](https://github.com/flutter/dart-intellij-third-party/pull/617) **Ready for review** (Branch `lsp-inlay-hints` = `20b85b64`, 4 Commits auf upstream `main` `fb835401`) |
| Go to Type Declaration (#580) | ❌ **Duplikat** — helin24s [#615](https://github.com/flutter/dart-intellij-third-party/pull/615) (18.08., 20:16 UTC) war zuerst da; [#618](https://github.com/flutter/dart-intellij-third-party/pull/618) am 19.08. mit Entschuldigung geschlossen. Branch `lsp-type-definition` (`7519c92d`) bleibt vorerst liegen (Location-tolerantes Parsing + Tests als mögliches Follow-up zu #615 angeboten). |
| Scope-Analyse #207 + Fragenkatalog | ✅ `specs/2026-08-18-lsp-migration-scope-analysis.md`, `OPEN-QUESTIONS-maintainers.md` (Q0–Q13); alle Fragen am 2026-08-18 gepostet, Stacked-PR-Frage am 2026-08-19 zurückgezogen |

#617 ist durch: Unit-Tests (`com.jetbrains.lang.dart.lsp.*`), `verifyPlugin` (keine neuen
Baseline-Zeilen, die auf die Änderungen zurückgehen), Repo-Code-Review-Skill (0 MUST-FIX), Final-Review
mit Fix-Wave, erste Gemini-Runde (beantwortet), manueller Sandbox-Check:

- **#617:** mit Flutter `master` (Dart 3.14.0-143.0.dev) alle Hint-Kategorien sichtbar, Closing Labels
  daneben ohne Doppelung (bleiben auch bei Flag aus), Toggle an/aus, Scratch-Datei analysiert und
  gehintet, Hints nach IDE-Start sobald „Analyzing…" fertig ist, Flutter `stable` (Dart 3.13.0) → keine
  Hints (Versions-Gate), `idea.log` ohne ERROR und ohne `inlayHint failed`. Kein Screenshot im PR (der
  getestete Code ist nicht veröffentlichbar; das Snippet im PR-Text reproduziert es). Im PR-Text steht
  ausdrücklich: Feature ist per Default an (Schalter defaultet auf `true`), alle Kategorien, kein eigener
  Aus-Schalter — die Maintainer sollen das bewusst absegnen.
- **#618 (geschlossen):** war in der Sandbox verifiziert (Flag an → ⌃⇧B springt zum Typ, auch `.pub-cache`;
  Flag aus → nichts; Log sauber), ist aber inhaltlich #615 — helin24 schaltet Type Declaration dort
  **global** frei (kein Legacy-Pfad, daher kein experimenteller Schalter) und advertised `linkSupport`
  direkt in `RequestUtilities.java`. Nach dem Merge von #615 muss #617 rebased werden (Überlappung in
  `initialize()`, `LspMethod`, Descriptor-Imports, CHANGELOG).

## Als Nächstes

1. **Review-Runden begleiten** (#617 + die Fragen):
   - Gemini/Maintainer-Kommentare auf #617 prüfen — insbesondere die Antwort auf die Gating-Frage
     (Flag behalten / nur SDK-Gate / eigene Checkbox) und ggf. auf das Follow-up-Angebot in #618: `gh pr view <n> --repo flutter/dart-intellij-third-party --comments`
     bzw. `gh api repos/flutter/dart-intellij-third-party/pulls/<n>/comments`. Änderungen: eine nach der
     anderen, lokal testen, erst dann pushen (so wie heute). Antworten in Ich-Form.
   - Antworten auf die Fragen einsammeln: `gh issue view <n> --repo flutter/dart-intellij-third-party --comments`
     für #207, #396, #400, #402, #401, #403, #404, #399, #407, #405, #406, #441; Ergebnis in
     `OPEN-QUESTIONS-maintainers.md` eintragen (✅ + Datum + Kurzfassung). Bei „ja" zu Q3 (Closing Labels)
     ein dart-lang/sdk-Issue aufmachen (Vorlage: `plans/2026-07-30-sdk-share-inlay-hint-handler.md`).
   - Wenn #615 (oder #612/#614) gemergt ist: #617 auf `main` rebasen (textuelle Überlappung in
     `DartBridgeLspServer.kt`, `LspMethod.kt`, `DartLspServerDescriptor.kt`, `CHANGELOG.md`, Test-Datei).
2. **Nach den Antworten planen:** Stack 2 („Navigation family": #396, ggf. #404/#403) bzw. Stack 3
   („Push-Notifications über LSP-over-Legacy": #400, #402) — jeweils zuerst SDK-Issue/CL, dann Plugin;
   für SDK-Änderungen die Handoff-Vorlage kopieren. Da Stacked PRs aus einem Fork nicht gehen
   (github/gh-stack#46), bleibt es bei max. zwei unabhängigen PRs gleichzeitig.
3. **Arbeitsumgebung:** Worktree `.claude/worktrees/lsp-inlay-hints` (steht auf `lsp-inlay-hints`;
   `lsp-type-definition` nur noch als Fundus für ein Follow-up zu #615) und SDD-Ledger
   `.superpowers/sdd/2026-08-18-lsp-endpoint-stack-1/progress.md` bleiben, bis #617 gemergt ist.
   Sandbox-Log: `<Worktree>/third_party/.intellijPlatform/sandbox/Dart/IU-2026.1.3/log/idea.log`.
   Dev-SDK für Inlay-Hint-Tests: `/Users/ralph.bergmann/development/sdks/flutter/bin/cache/dart-sdk`
   (Flutter `master`). SDK-Checkout `~/development/projects/privat/dart-sdk/sdk` steht noch auf dem
   lokalen CL-Branch (`75e7d44de95`) — vor neuer SDK-Arbeit `git checkout main && git pull origin main`.

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
- **Kotlin: NIEMALS `!!` — auch nicht in Tests, auch nicht „weil der Nachbartest es so macht".**
  `.gemini/styleguide.md` (Zeile 47) verbietet es ohne Ausnahme, und Gemini flaggt es in jedem PR als
  `[MUST-FIX]` (#552-Review und erneut #617). Stattdessen `requireNotNull(x) { "…" }` (Vorbild:
  `DartBridgeLspServerTest.setUp`), `?.`, `?:` oder `if (x != null)`. Pläne dürfen keine Ausnahme
  „für Testcode" mehr formulieren; ein vorhandenes `!!` im Umfeld wird nicht kopiert, sondern ist ein
  Follow-up-Kandidat (requireNotNull-Sweep).
- **Vor jedem neuen Feature die PR-Liste neu ziehen — komplett, nicht nur die bekannten PRs:**
  `gh pr list --repo flutter/dart-intellij-third-party --state open` plus Assignees/Kommentare des
  Ziel-Issues. Lehre aus #618 (19.08.): helin24 hatte #580 am Abend vorher als #615 geöffnet; ich hatte
  nur #526/#612/#614 auf Merge-Stand geprüft und damit ein Duplikat gebaut und eröffnet. Also: erst
  gucken, dann bauen — auch wenn die Liste „gestern" schon geprüft wurde.
- **Max. zwei offene PRs.** Stacked PRs gehen aus einem Fork nicht („Cross-fork stacks are not
  supported", github/gh-stack#46), also unabhängige PRs; wer als Zweiter gemergt wird, rebased trivial.
- Nichts unter `third_party/thirdPartySrc/` ändern (Code-Review-Skill des Repos → `[MUST-FIX]`);
  Ausnahme nur mit Owner-Freigabe.
- **Auf GitHub als Person schreiben** („ich", nicht „wir"); PR-Änderungen eine nach der anderen, lokal
  getestet, erst dann pushen.
- **Lessons aus helin24s #615 (2026-08-19):**
  - *Gating:* Der experimentelle LSP-Schalter ist zum Umschalten zwischen Legacy und LSP da, nicht als
    allgemeiner Feature-Toggle. Gibt es einen Legacy-Pfad → Flag (+ Versions-Gate, wenn der Endpoint
    jung ist). Gibt es keinen → nur Versions-Gate, falls nötig, sonst gar kein Gate; `LspMethod`-Eintrag
    dann mit `isExperimental = false` (wie `TYPE_DEFINITION` in #615). Für #617 als Frage gestellt
    ([Kommentar](https://github.com/flutter/dart-intellij-third-party/pull/617#issuecomment-5345378240)):
    Flag behalten (= einziger Aus-Schalter), nur SDK-Gate, oder SDK-Gate + eigene Dart-Checkbox.
  - *PR-Text:* kurz wie #615 — Was/Warum in zwei Sätzen, Test-Snippet + erwartetes Ergebnis ganz oben;
    Trade-offs (migrate-das-to-lsp-Skill) bleiben, aber knapp.
  - *Revier:* Navigations-Familie (#539, #615 → vermutlich #396/#403/#404) macht helin24 selbst. Erst die
    Antwort auf „was soll ich übernehmen?" (#207) abwarten, dann bauen — Fragen allein reicht nicht.
  - *Tests:* nichts zu übernehmen — unser Umfang (Null-Ergebnis, gesendete Params) war größer; bleibt so,
    mit `requireNotNull`.
- Diese Regeln stehen als „Ground rules" auch am Anfang der SDK-Handoff-Doku, damit der dortige
  Agent sie ohne diesen Kontext hat.

## Offene Aufräumpunkte (nicht dringend)

- `requireNotNull`-Sweep über die vorbestehenden `!!` in `DartBridgeLspServerTest`
  (`testDiagnosticServerRequest`, `testDocumentHighlightRequest`) als kleiner eigener PR — Gemini
  gegenüber auf #617 angeboten.
- 2026.2-EAP-Inkompatibilität (`PsiTreeElementBase` weg, Structure View) als eigenes Issue/Fix —
  betrifft `main`, nicht meine PRs (würde durch #402/documentSymbol mittelfristig obsolet).
- `gradlew.bat`-Zeilenenden-Normalisierung als Upstream-Housekeeping.
- Ggf. Versions-Gate für documentHighlight analog `isLspNavigationEnabled` (Review-Idee aus #552).
- `docs/SCR-20260819-qcsg.png` liegt unversioniert im Planungs-Checkout (nicht committen; nicht
  veröffentlichbar).

## Verlauf (Kurzfassung)

- 2026-08-17: SDK-CL für Shared InlayHintHandler hochgeladen und gelandet; Branch `DartInlayHints`
  rebased, CLA-Hygiene auf allen Commits.
- 2026-08-18: Scope-Analyse #207 (16 offene Sub-Issues, drei Gruppen), Fragenkatalog Q0–Q13 (inkl.
  Server-Optionen aus `pkg/analysis_server/tool/lsp_spec/README.md`), Plan Stack 1; alle Fragen gepostet.
- 2026-08-19: Fork-`main` auf upstream `fb835401`; Stack 1 per subagent-driven-development umgesetzt
  (Implementer/Reviewer-Subagenten, Final-Review, Fix-Wave: DAS-Fehler bei `inlayHint` → „keine Hints"
  + Info-Log, weil der Inlay-Hint-Pfad des LSP-Clients Exceptions nicht fängt); beide PRs als Draft
  geöffnet; GitHub-Stacks scheitern aus dem Fork → #618 auf `main` entstapelt, Frage auf #207
  zurückgezogen; Gemini-Runde 1 abgearbeitet (`!!`→`requireNotNull`; `typeDefinition` akzeptiert
  `Location`-Antworten); Sandbox-Checks für beide PRs mit Ralph; beide PRs auf „Ready for review" —
  dann entdeckt, dass #618 helin24s #615 dupliziert → #618 geschlossen, Entschuldigung auf #618/#207,
  neue Regel „PR-Liste vor jedem Feature neu ziehen".
