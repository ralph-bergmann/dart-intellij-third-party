# LSP Endpoint Stack 1 (Inlay Hints + Type Definition) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development
> (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use
> checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship two LSP endpoints through the DAS→LSP bridge as one stacked pair of PRs: **PR A**
inlay hints (`textDocument/inlayHint`, issue #159) and **PR B** Go To Type Declaration
(`textDocument/typeDefinition`, issue #580), PR B based on PR A's branch.

**Architecture:** Both features ride the existing `DartBridgeLspServer` (`lsp.handle` tunnel) and are
rendered by the bundled JetBrains LSP client (`com.intellij.platform.dartlsp`): the bridge advertises
the capability and forwards the request, `LspMethod` gets an entry (feeds the settings checkbox
label), `DartLspServerDescriptor` switches the customizer on behind *Turn on experimental LSP
features*. Inlay hints additionally gate on `MIN_LSP_INLAY_HINTS_SDK_VERSION = "3.14.0-139.0.dev"`
(the first dev SDK whose `InlayHintHandler` is a shared handler — dart-lang/sdk `7c18d1fa0e5`);
typeDefinition needs no version gate (shared since Dart 3.3.0, like hover). No new provider classes,
no `plugin.xml` changes, no legacy code to gate (there is no Dart legacy inlay-hint provider besides
the separate closing labels, and no Dart `TypeDeclarationProvider` at all).

**Tech Stack:** Kotlin/Java, IntelliJ Platform Gradle plugin (Gradle from `third_party/`, `JAVA_HOME`
= IntelliJ JBR), lsp4j, existing test base `DartBridgeLspServerTest` (`DartCodeInsightFixtureTestCase`).

**Spec:** `docs/superpowers/specs/2026-08-18-lsp-migration-scope-analysis.md` (decisions S1–S3, S6)
and, for inlay-hint behaviour/trade-offs, `docs/superpowers/specs/2026-07-30-dart-inlay-hints-and-highlighting-design.md` §4.
This plan **supersedes Part 3** of `2026-07-30-dart-inlay-hints.md` (same tasks, prerequisite values
filled in, stacked-PR mechanics added).

## Global Constraints

- Remotes: `origin` = `ralph-bergmann/dart-intellij-third-party` (fork). PRs target
  `flutter/dart-intellij-third-party`; PR A base `main`, ~~PR B base `lsp-inlay-hints` (stacked)~~
  **2026-08-19: PR B base `main` as well** — GitHub stacks need all branches incl. the trunk in one
  repository ("Cross-fork stacks are not supported", github/gh-stack#46), so the PRs are independent;
  `lsp-type-definition` was rebased onto `main` (#618 = `0718d326`). Task 7 Step 4 (retarget) is N/A;
  whichever PR merges second gets a trivial rebase.
- Feature branches are cut from a fresh `main` (`git checkout main && git pull origin main`),
  preferably in a worktree under `.claude/worktrees/` (superpowers:using-git-worktrees). The planning
  docs (`docs/superpowers/**`) exist only on the `DartInlayHints` branch — never include them in a
  feature branch. Neutralize the CRLF trap per worktree:
  `git update-index --assume-unchanged third_party/gradlew.bat`.
- **Before starting each PR, check helin24's in-flight PRs** and rebase onto `main` after they merge:
  #612 (publishDiagnostics; touches `DartBridgeLspServer.kt`, `DartBridgeLspServerTest.kt`,
  `DartAnalysisServerService.java`) and #614 (`buildLspCapabilities`; PR B depends on it — see
  Task 5). `gh pr view 612 --repo flutter/dart-intellij-third-party --json state,mergedAt`.
- NEVER modify files under `third_party/thirdPartySrc/` (the repo code-review skill rejects it with
  `[MUST-FIX]`; the only in-scope exception is described in Task 5's fallback).
- Kotlin: **no `!!` anywhere — tests included** (`.gemini/styleguide.md`: "NEVER use the double-bang
  `!!` operator"; Gemini flags it as `[MUST-FIX]` — it did on #552 and #617). Use
  `requireNotNull(x) { "…" }` (as `DartBridgeLspServerTest.setUp` does), `?.`, `?:` or `if (x != null)`;
  do not copy the `!!` from the pre-existing `testDocumentHighlightRequest`. Prefer `val`, imports
  instead of fully qualified names (the bridge already imports `com.google.gson.reflect.TypeToken` —
  use it unqualified, as `documentHighlight` does). Java: mirror the surrounding style (2-space indent,
  `final` locals as in `isLspNavigationEnabled`).
  *(2026-08-19: the earlier "test code may mirror the existing `!!`" exemption in this plan's task
  briefs was wrong and is revoked — the test snippets in Tasks 1 and 4 must use `requireNotNull`.)*
- All Gradle commands run from `third_party/` with
  `export JAVA_HOME="$HOME/Applications/IntelliJ IDEA.app/Contents/jbr/Contents/Home"`. Long runs
  (`test`, `verifyPlugin`, `runIde`) exceed the 10-minute tool limit — run in the background and poll.
- CHANGELOG entries go under `## Unreleased` / `### Added` in `third_party/CHANGELOG.md`, phrased
  like existing entries (gerund/noun style, e.g. "Highlighting …"), with the PR number filled in after
  the PR exists (amend + `git push --force-with-lease`).
- Commits: short imperative subject; author/committer **`Ralph Bergmann <ralph@dasralph.de>`**
  (verify `git config user.email` in the worktree — the shared `.git/config` sets it); **no
  `Co-Authored-By:` trailer of any kind** (Google CLA bot fails otherwise). Stage with **explicit
  paths, never `git add -A`**. Verify with `git show --stat HEAD`.
- PR bodies follow the repo template (behaviour, how to test manually, functional differences per the
  migrate-das-to-lsp skill, contribution checklist), reference the issue, and end with
  `🤖 Generated with [Claude Code](https://claude.com/claude-code)`.
- **Code review before every PR:** run `.agents/skills/code-review/SKILL.md` on `git diff main...HEAD`
  (all passes incl. thirdPartySrc check, correctness, resource management, `.gemini/styleguide.md`)
  and fix every `[MUST-FIX]`/`[CONCERN]`. This is in addition to the subagent-driven-development
  reviewers.
- **migrate-das-to-lsp checklist** (`.agents/skills/migrate-das-to-lsp/SKILL.md`) applies to both PRs:
  clean sandbox (`./gradlew clean prepareSandbox --no-build-cache`), files open at startup AND opened
  later, external files (`.pub-cache`, `dart:io`), settings-toggle lifecycle, `./gradlew verifyPlugin`
  judged via the CI baseline comparison (`/usr/bin/grep '^\*' report.md | /usr/bin/grep -v dartlsp | sort`
  vs `third_party/tool/baseline/<ver>/verifier-baseline.txt`; the pre-existing IU-262
  `PsiTreeElementBase` failure and the 3 lambda-index lines are known noise; **never run
  `tool/update_baselines.sh` locally**). Its "Step 1: gate the legacy provider" is N/A for both
  features — state that in each PR body.
- Sandbox-testing pitfalls (project memory `lsp-feature-testing`): swapping the SDK on disk does not
  restart the running DAS (use *Restart Dart Analysis Server*), toggling the flag restarts only the
  bridge, client caches survive until the document is modified — type a character before judging.

---

## Prerequisites (all verified 2026-08-18)

- SDK: `InlayHintHandler` shared handler landed as `7c18d1fa0e5`
  ([CL 536565](https://dart-review.googlesource.com/c/sdk/+/536565), fixes dart-lang/sdk#64061);
  `git -C ../dart-sdk/sdk tag --contains 7c18d1fa0e5 | sort -V | head -1` → **`3.14.0-139.0.dev`**
  (`3.14.0-138.0.dev` does not contain it). This is the value of `MIN_LSP_INLAY_HINTS_SDK_VERSION`.
- Test fixtures in `third_party/src/test/java/com/jetbrains/lang/dart/lsp/DartBridgeLspServerTest.kt`:
  `bridgeServer: DartBridgeLspServer`, `capturedRequests: CopyOnWriteArrayList<JsonObject>`,
  `capturedListener: ResponseListener`, mock request id `"123"`; existing model test
  `testDocumentHighlightRequest`.
- Bridge helpers in `DartBridgeLspServer.kt`: `private fun <T> forwardRequest(method: String,
  params: Any?, responseType: Type): CompletableFuture<T>`; capabilities in `initialize()`
  currently `setHoverProvider(true)`, `setDefinitionProvider(true)`, `setDocumentHighlightProvider(true)`.
- Version-gate precedent in `DartAnalysisServerService.java`: constant
  `MIN_LSP_NAVIGATION_SDK_VERSION` (line 183), methods `isDartSdkVersionSufficientForLspNavigation` /
  `isLspNavigationEnabled` (lines 582–593).
- End-to-end inlay-hint testing needs a Dart SDK ≥ `3.14.0-139.0.dev` (dev channel,
  https://dart.dev/get-dart/archive → "Dev channel"; e.g.
  `https://storage.googleapis.com/dart-archive/channels/dev/release/3.14.0-139.0.dev/sdk/dartsdk-macos-arm64-release.zip`)
  or a Flutter master/beta channel that bundles one. A source build from `../dart-sdk/sdk` is *not*
  needed anymore.

---

# PR A — "Show LSP inlay hints for Dart as an experimental feature" (branch `lsp-inlay-hints`, base `main`)

Issue: [#159](https://github.com/flutter/dart-intellij-third-party/issues/159).

### Task 1: Forward `textDocument/inlayHint` in the bridge (TDD)

**Files:**
- Test: `third_party/src/test/java/com/jetbrains/lang/dart/lsp/DartBridgeLspServerTest.kt`
- Modify: `third_party/src/main/java/com/jetbrains/lang/dart/lsp/DartBridgeLspServer.kt`

**Interfaces:**
- Produces: `DartBridgeLspServer.inlayHint(params: InlayHintParams): CompletableFuture<List<InlayHint>>`;
  `initialize()` advertises `inlayHintProvider`.
- Consumes: existing `forwardRequest(method, params, responseType)` and the test fixtures above.

- [ ] **Step 1: Create the branch (worktree)**

```bash
git checkout main && git pull origin main
git worktree add .claude/worktrees/lsp-inlay-hints -b lsp-inlay-hints main
cd .claude/worktrees/lsp-inlay-hints && git update-index --assume-unchanged third_party/gradlew.bat
git config user.email   # must print ralph@dasralph.de
```

- [ ] **Step 2: Write the failing test**

Add to `DartBridgeLspServerTest.kt` after `testDocumentHighlightRequest` (imports to add, keeping
the alphabetical import block: `org.eclipse.lsp4j.InlayHintKind`, `org.eclipse.lsp4j.InlayHintParams`,
`org.eclipse.lsp4j.Range`):

```kotlin
    fun testInlayHintRequest() {
        val params = InlayHintParams().apply {
            textDocument = TextDocumentIdentifier("file://test.dart")
            range = Range(Position(0, 0), Position(10, 0))
        }

        val future = bridgeServer.inlayHint(params)

        val jsonObject = capturedRequests.find { it.get("method")?.asString == "lsp.handle" }
        assertNotNull("An lsp.handle request should be sent to DAS", jsonObject)
        assertEquals("123", jsonObject!!.get("id").asString)

        val lspMessage = jsonObject.getAsJsonObject("params").getAsJsonObject("lspMessage")
        assertEquals("123", lspMessage.get("id").asString)
        assertEquals("textDocument/inlayHint", lspMessage.get("method").asString)

        val responseJson = """
            {
              "id": "123",
              "result": {
                "lspResponse": {
                  "jsonrpc": "2.0",
                  "id": "123",
                  "result": [
                    {"position": {"line": 0, "character": 5}, "label": "String", "kind": 1},
                    {"position": {"line": 2, "character": 8}, "label": [{"value": "name:"}], "kind": 2}
                  ]
                }
              }
            }
        """.trimIndent()

        capturedListener.onResponse(responseJson)

        val result = future.get(5, TimeUnit.SECONDS)
        assertEquals(2, result.size)
        assertEquals(InlayHintKind.Type, result[0].kind)
        assertEquals("String", result[0].label.left)
        assertEquals(InlayHintKind.Parameter, result[1].kind)
        assertEquals("name:", result[1].label.right[0].value)
    }
```

(The `!!` after `assertNotNull` mirrors `testDocumentHighlightRequest`; test code is exempt from the
no-`!!` rule.)

- [ ] **Step 3: Run the test — expect compile failure**

```bash
cd third_party && ./gradlew test --tests "com.jetbrains.lang.dart.lsp.DartBridgeLspServerTest"
```

Expected: FAILS — `bridgeServer.inlayHint` is not overridden (unresolved reference / lsp4j default
throws `UnsupportedOperationException`).

- [ ] **Step 4: Implement the bridge forwarding**

`DartBridgeLspServer.kt` — imports to add (alphabetical block): `org.eclipse.lsp4j.InlayHint`,
`org.eclipse.lsp4j.InlayHintParams`.

In `initialize()`:

```kotlin
        val capabilities = ServerCapabilities().apply {
            setHoverProvider(true)
            setDefinitionProvider(true)
            setDocumentHighlightProvider(true)
            setInlayHintProvider(true)
            // Add other capabilities as we support them.
        }
```

Directly below `documentHighlight(...)`:

```kotlin
    override fun inlayHint(params: InlayHintParams): CompletableFuture<List<InlayHint>> {
        val type = object : TypeToken<List<InlayHint>>() {}.type
        return forwardRequest<List<InlayHint>>("textDocument/inlayHint", params, type)
    }
```

- [ ] **Step 5: Run the test — expect pass**

```bash
./gradlew test --tests "com.jetbrains.lang.dart.lsp.DartBridgeLspServerTest"
```

Expected: PASS (all tests in the class).

- [ ] **Step 6: Commit**

```bash
git add third_party/src/main/java/com/jetbrains/lang/dart/lsp/DartBridgeLspServer.kt \
        third_party/src/test/java/com/jetbrains/lang/dart/lsp/DartBridgeLspServerTest.kt
git commit -m "Forward textDocument/inlayHint in DartBridgeLspServer"
git show --stat HEAD   # exactly these two files, author ralph@dasralph.de
```

### Task 2: SDK version gate, `LspMethod` entry, customizer, changelog

**Files:**
- Modify: `third_party/src/main/java/com/jetbrains/lang/dart/analyzer/DartAnalysisServerService.java`
  (constant near line 183, methods near lines 582–593)
- Modify: `third_party/src/main/java/com/jetbrains/lang/dart/lsp/LspMethod.kt`
- Modify: `third_party/src/main/java/com/jetbrains/lang/dart/lsp/DartLspServerDescriptor.kt`
- Modify: `third_party/CHANGELOG.md`
- Test (must stay green): `third_party/src/test/java/com/jetbrains/lang/dart/lsp/DartLspExperimentalFeaturesTest.java`

**Interfaces:**
- Produces: `DartAnalysisServerService.MIN_LSP_INLAY_HINTS_SDK_VERSION`,
  `isDartSdkVersionSufficientForLspInlayHints(String)`, `isLspInlayHintsEnabled(Project)`;
  `LspMethod.INLAY_HINT`; enabled `inlayHintCustomizer`.
- Consumes: `DartBridgeLspServer.inlayHint` (Task 1); `DartConfigurable.isExperimentalLspFeaturesEnabled`,
  `DartSdk.getDartSdk`, `DartSdkUpdateChecker.compareDartSdkVersions` (all existing).

- [ ] **Step 1: Add the version gate**

In `DartAnalysisServerService.java`, directly after
`public static final String MIN_LSP_NAVIGATION_SDK_VERSION = "3.14.0-65.0.dev";`:

```java
  public static final String MIN_LSP_INLAY_HINTS_SDK_VERSION = "3.14.0-139.0.dev";
```

Directly after `isLspNavigationEnabled(...)`:

```java
  public static boolean isDartSdkVersionSufficientForLspInlayHints(@NotNull String sdkVersion) {
    return DartSdkUpdateChecker.compareDartSdkVersions(sdkVersion, MIN_LSP_INLAY_HINTS_SDK_VERSION) >= 0;
  }

  public static boolean isLspInlayHintsEnabled(final @NotNull Project project) {
    if (!DartConfigurable.isExperimentalLspFeaturesEnabled(project)) {
      return false;
    }
    final DartSdk sdk = DartSdk.getDartSdk(project);
    return sdk != null && isDartSdkVersionSufficientForLspInlayHints(sdk.getVersion());
  }
```

- [ ] **Step 2: Add the `LspMethod` entry**

`LspMethod.kt` — keep the entries alphabetical, so between `INITIALIZE` and `SHUTDOWN`:

```kotlin
    INITIALIZE("initialize"),
    INLAY_HINT("textDocument/inlayHint", isExperimental = true, presentableName = "inlay hints"),
    SHUTDOWN("shutdown");
```

(The settings checkbox label is assembled in `DartConfigurable.java:117` from
`LspMethod.getExperimentalFeatures()`, so "inlay hints" appears there automatically.)

- [ ] **Step 3: Enable the customizer**

`DartLspServerDescriptor.kt` — replace `override val inlayHintCustomizer = LspInlayHintDisabled` with:

```kotlin
        override val inlayHintCustomizer: LspInlayHintCustomizer
            get() = if (DartAnalysisServerService.isLspInlayHintsEnabled(project)) {
                LspInlayHintSupport()
            } else {
                LspInlayHintDisabled
            }
```

Imports to add (alphabetical block): `com.intellij.platform.dartlsp.api.customization.LspInlayHintCustomizer`,
`com.intellij.platform.dartlsp.api.customization.LspInlayHintSupport`.

- [ ] **Step 4: Compile and run the LSP test suites**

```bash
./gradlew compileKotlin compileJava && ./gradlew test --tests "com.jetbrains.lang.dart.lsp.*"
```

Expected: BUILD SUCCESSFUL, all green (`DartLspExperimentalFeaturesTest` only checks the flag
persistence; the settings checkbox label is built from `LspMethod.getExperimentalFeatures()` and picks
up "inlay hints" automatically).

- [ ] **Step 5: CHANGELOG**

Under `## Unreleased` / `### Added` in `third_party/CHANGELOG.md`:

```markdown
- Inlay hints for types and parameter names (JetBrains LSP, experimental feature; requires Dart SDK 3.14.0-139.0.dev or newer) (#PR)
```

- [ ] **Step 6: Commit**

```bash
git add third_party/src/main/java/com/jetbrains/lang/dart/analyzer/DartAnalysisServerService.java \
        third_party/src/main/java/com/jetbrains/lang/dart/lsp/LspMethod.kt \
        third_party/src/main/java/com/jetbrains/lang/dart/lsp/DartLspServerDescriptor.kt \
        third_party/CHANGELOG.md
git commit -m "Show LSP inlay hints for Dart as an experimental feature

Forwards textDocument/inlayHint through the DAS bridge and enables the
JetBrains LSP inlay hint rendering when the experimental-LSP setting is
on and the SDK is recent enough to serve inlayHint over LSP-over-Legacy
(dart-lang/sdk 7c18d1fa0e5, first dev tag 3.14.0-139.0.dev)."
```

### Task 3: Verify end-to-end, review, open PR A

**Files:** none new (verification + PR).

- [ ] **Step 1: Sandbox verification (migrate-das-to-lsp checklist)**

Point the sandbox project at a Dart SDK ≥ `3.14.0-139.0.dev` (Prerequisites). Then:

```bash
./gradlew clean prepareSandbox --no-build-cache && ./gradlew runIde   # background, poll
```

In the sandbox IDE with *Settings | Languages & Frameworks | Dart | Turn on experimental LSP
features* ON, open a Dart file containing:

```dart
void main() {
  var a = '';
  print(int.parse('1', radix: 10));
  final list = [1, 2, 3].map((e) => e * 2).toList();
}
```

Expect a `String` type hint after `a`, a `List<int>` hint after `list`, and parameter-name hints on
positional arguments (e.g. `source:` before `'1'` — the exact set is the server's default
configuration; there is no per-category IDE settings UI, document that in the PR). Check:

- a file already open at IDE startup AND a file opened afterwards,
- an external file (`.pub-cache` dependency and `dart:io`),
- closing labels still render alongside the new hints (no duplication),
- toggling the setting off removes the hints cleanly,
- with an old SDK (< `3.14.0-139.0.dev`) no hints appear and `idea.log`
  (`third_party/.intellijPlatform/sandbox/Dart/IU-*/log/idea.log`) shows no errors,
- after any SDK swap: *Restart Dart Analysis Server*, then edit the file once before judging.

Take screenshots (flag off / flag on) for the PR body.

- [ ] **Step 2: Plugin verifier + baseline comparison**

```bash
./gradlew verifyPlugin   # background; exit code 1 is expected (baselined IU-262 problem)
for d in third_party/build/reports/pluginVerifier/IU-*; do echo "== $d"; \
  /usr/bin/grep '^\*' "$d/report.md" | /usr/bin/grep -v dartlsp | sort > /tmp/new.txt; \
  v=$(basename "$d" | sed 's/IU-\([0-9]*\).*/\1/'); \
  diff <(sort third_party/tool/baseline/$v/verifier-baseline.txt) /tmp/new.txt; done
```

Expected: no new lines except the known 3 lambda-index renumbering lines. Do NOT update baselines.

- [ ] **Step 3: Repository code review**

Run `.agents/skills/code-review/SKILL.md` on `git diff main...HEAD` (all passes). Fix every
`[MUST-FIX]`/`[CONCERN]`, re-run the unit tests, amend/commit as appropriate.

- [ ] **Step 4: Push and open PR A**

```bash
git push -u origin lsp-inlay-hints
gh pr create --repo flutter/dart-intellij-third-party --base main \
  --title "Show LSP inlay hints for Dart as an experimental feature" \
  --body "Implements #159 via the LSP bridge: the Dart Analysis Server's inlay hints (variable types, parameter names, return types, type arguments, dot-shorthand types) render through the bundled LSP client. Gated by the experimental-LSP setting plus \`MIN_LSP_INLAY_HINTS_SDK_VERSION = \"3.14.0-139.0.dev\"\` — the SDK gained \`textDocument/inlayHint\` over LSP-over-Legacy in dart-lang/sdk@7c18d1fa0e5 (dart-lang/sdk#64061).

**Manual test:** experimental LSP on, project SDK ≥ 3.14.0-139.0.dev, open a Dart file with \`var a = ''; print(int.parse('1'));\` — type hint after \`a\`, parameter-name hints on positional arguments. Toggle the setting off → hints disappear; older SDK → no hints, no errors.

**Functional differences / trade-offs:** hint categories follow the server defaults — no per-category settings UI yet (\`workspace/didChangeConfiguration\` is not available over the legacy protocol; IntelliJ's Inlay Hints settings tree only lists declarative providers). Labels longer than 42 chars are truncated (framework default). Closing labels are unaffected (separate mechanism, see #400/#551).

Per the migrate-das-to-lsp skill: there is no legacy provider to gate — inlay hints are a new feature.

<screenshots>

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

Then replace `#PR` in the CHANGELOG line with the new PR number:
`git commit --amend --no-edit && git push --force-with-lease`.

---

# PR B — "Go to Type Declaration via LSP typeDefinition" (branch `lsp-type-definition`, base `lsp-inlay-hints`)

Issue: [#580](https://github.com/flutter/dart-intellij-third-party/issues/580). Stacked on PR A so
that both "enable an endpoint" changes are reviewed as one pattern; after PR A merges, retarget PR B
to `main` (`gh pr edit <B> --base main`) and rebase.

### Task 4: Forward `textDocument/typeDefinition` in the bridge (TDD)

**Files:**
- Test: `third_party/src/test/java/com/jetbrains/lang/dart/lsp/DartBridgeLspServerTest.kt`
- Modify: `third_party/src/main/java/com/jetbrains/lang/dart/lsp/DartBridgeLspServer.kt`

**Interfaces:**
- Produces: `DartBridgeLspServer.typeDefinition(params: TypeDefinitionParams): CompletableFuture<Either<List<Location>, List<LocationLink>>>`
  (right side always used, like `definition`); `initialize()` advertises `typeDefinitionProvider`.
- Consumes: `forwardRequest`, test fixtures.

- [ ] **Step 1: Create the branch on top of PR A**

```bash
cd .claude/worktrees/lsp-inlay-hints && git checkout -b lsp-type-definition   # or a new worktree from lsp-inlay-hints
```

- [ ] **Step 2: Write the failing test**

Add to `DartBridgeLspServerTest.kt` after `testInlayHintRequest` (import to add:
`org.eclipse.lsp4j.TypeDefinitionParams`):

```kotlin
    fun testTypeDefinitionRequest() {
        val params = TypeDefinitionParams().apply {
            textDocument = TextDocumentIdentifier("file://test.dart")
            position = Position(3, 7)
        }

        val future = bridgeServer.typeDefinition(params)

        val jsonObject = capturedRequests.find { it.get("method")?.asString == "lsp.handle" }
        assertNotNull("An lsp.handle request should be sent to DAS", jsonObject)
        assertEquals("123", jsonObject!!.get("id").asString)

        val lspMessage = jsonObject.getAsJsonObject("params").getAsJsonObject("lspMessage")
        assertEquals("123", lspMessage.get("id").asString)
        assertEquals("textDocument/typeDefinition", lspMessage.get("method").asString)

        val responseJson = """
            {
              "id": "123",
              "result": {
                "lspResponse": {
                  "jsonrpc": "2.0",
                  "id": "123",
                  "result": [
                    {
                      "originSelectionRange": {"start": {"line": 3, "character": 6}, "end": {"line": 3, "character": 9}},
                      "targetUri": "file:///lib/foo.dart",
                      "targetRange": {"start": {"line": 10, "character": 0}, "end": {"line": 20, "character": 1}},
                      "targetSelectionRange": {"start": {"line": 10, "character": 6}, "end": {"line": 10, "character": 9}}
                    }
                  ]
                }
              }
            }
        """.trimIndent()

        capturedListener.onResponse(responseJson)

        val result = future.get(5, TimeUnit.SECONDS)
        assertTrue(result.isRight)
        assertEquals(1, result.right.size)
        assertEquals("file:///lib/foo.dart", result.right[0].targetUri)
        assertEquals(10, result.right[0].targetSelectionRange.start.line)
    }
```

- [ ] **Step 3: Run the test — expect failure**

```bash
cd third_party && ./gradlew test --tests "com.jetbrains.lang.dart.lsp.DartBridgeLspServerTest"
```

Expected: FAILS (`typeDefinition` not overridden).

- [ ] **Step 4: Implement the bridge forwarding**

`DartBridgeLspServer.kt` — import to add: `org.eclipse.lsp4j.TypeDefinitionParams` (`Location`,
`LocationLink`, `Either` are already imported).

`initialize()` gains `setTypeDefinitionProvider(true)` after `setInlayHintProvider(true)`.

Directly below `definition(...)` (same shape, same guarantee comment):

```kotlin
    // Note: We advertise linkSupport: true for typeDefinition in server.setClientCapabilities
    // (see DartAnalysisServerService.buildLspCapabilities) so DAS returns List<LocationLink>.
    override fun typeDefinition(params: TypeDefinitionParams): CompletableFuture<Either<List<Location>, List<LocationLink>>> {
        val type = object : TypeToken<List<LocationLink>>() {}.type
        return forwardRequest<List<LocationLink>>("textDocument/typeDefinition", params, type).thenApply { links ->
            Either.forRight(links ?: emptyList())
        }
    }
```

- [ ] **Step 5: Run the test — expect pass, then commit**

```bash
./gradlew test --tests "com.jetbrains.lang.dart.lsp.DartBridgeLspServerTest"
git add third_party/src/main/java/com/jetbrains/lang/dart/lsp/DartBridgeLspServer.kt \
        third_party/src/test/java/com/jetbrains/lang/dart/lsp/DartBridgeLspServerTest.kt
git commit -m "Forward textDocument/typeDefinition in DartBridgeLspServer"
```

### Task 5: Advertise `typeDefinition.linkSupport` in the client capabilities (TDD)

**Files:**
- Modify: `third_party/src/main/java/com/jetbrains/lang/dart/analyzer/DartAnalysisServerService.java`
  (`buildLspCapabilities`, introduced by PR #614)
- Test: `third_party/src/test/java/com/jetbrains/lang/dart/lsp/DartBridgeLspServerTest.kt`

**Interfaces:**
- Produces: `buildLspCapabilities(sdkVersion)` JSON contains
  `textDocument.typeDefinition.linkSupport = true` (next to the existing `textDocument.definition.linkSupport`).
- Consumes: `DartAnalysisServerService.buildLspCapabilities(@NotNull String sdkVersion): JsonObject`
  from #614.

**Precondition check:** `gh pr view 614 --repo flutter/dart-intellij-third-party --json state,mergedAt`.
`buildLspCapabilities` exists only after #614 merged (rebase `lsp-inlay-hints` and this branch onto
the new `main` first). **Fallback if #614 is closed without merging:** the capability JSON still lives
in `third_party/thirdPartySrc/analysisServer/com/google/dart/server/internal/remote/utilities/RequestUtilities.java`
(`generateClientCapabilities`, the block that adds `definition.linkSupport`); editing that file needs
the repository-owner override the code-review skill mentions — ask helin24 on the PR (precedent: PR
#539 edited exactly that block) before touching it. Do not proceed silently.

- [ ] **Step 1: Write the failing test**

Add to `DartBridgeLspServerTest.kt` (imports: `com.jetbrains.lang.dart.analyzer.DartAnalysisServerService`
is already imported):

```kotlin
    fun testLspCapabilitiesAdvertiseTypeDefinitionLinkSupport() {
        val lspCapabilities = DartAnalysisServerService.buildLspCapabilities("3.14.0-139.0.dev")
        val textDocument = lspCapabilities.getAsJsonObject("textDocument")
        assertEquals(true, textDocument.getAsJsonObject("definition").get("linkSupport").asBoolean)
        assertEquals(true, textDocument.getAsJsonObject("typeDefinition").get("linkSupport").asBoolean)
    }
```

- [ ] **Step 2: Run — expect failure**

```bash
./gradlew test --tests "com.jetbrains.lang.dart.lsp.DartBridgeLspServerTest"
```

Expected: FAILS with a `NullPointerException`/assertion on the missing `typeDefinition` object.

- [ ] **Step 3: Implement**

In `DartAnalysisServerService.buildLspCapabilities`, directly after the three lines that build and
add `definition`:

```java
    JsonObject typeDefinition = new JsonObject();
    typeDefinition.addProperty("linkSupport", true);
    textDocument.add("typeDefinition", typeDefinition);
```

- [ ] **Step 4: Run — expect pass, commit**

```bash
./gradlew test --tests "com.jetbrains.lang.dart.lsp.DartBridgeLspServerTest"
git add third_party/src/main/java/com/jetbrains/lang/dart/analyzer/DartAnalysisServerService.java \
        third_party/src/test/java/com/jetbrains/lang/dart/lsp/DartBridgeLspServerTest.kt
git commit -m "Advertise typeDefinition linkSupport to the analysis server"
```

### Task 6: Enable Go To Type Declaration (LspMethod + customizer + changelog)

**Files:**
- Modify: `third_party/src/main/java/com/jetbrains/lang/dart/lsp/LspMethod.kt`
- Modify: `third_party/src/main/java/com/jetbrains/lang/dart/lsp/DartLspServerDescriptor.kt`
- Modify: `third_party/CHANGELOG.md`

**Interfaces:**
- Produces: `LspMethod.TYPE_DEFINITION`; enabled `goToTypeDefinitionCustomizer`.
- Consumes: `DartBridgeLspServer.typeDefinition` (Task 4), `DartConfigurable.isExperimentalLspFeaturesEnabled`.

- [ ] **Step 1: `LspMethod` entry** (alphabetical: after `SHUTDOWN` — it becomes the last entry, so
move the `;` and turn the previous entry's `;` into `,`):

```kotlin
    SHUTDOWN("shutdown"),
    TYPE_DEFINITION("textDocument/typeDefinition", isExperimental = true, presentableName = "go to type declaration");
```

- [ ] **Step 2: Customizer** — in `DartLspServerDescriptor.kt` replace
`override val goToTypeDefinitionCustomizer = LspGoToTypeDefinitionDisabled` with:

```kotlin
        override val goToTypeDefinitionCustomizer: LspGoToTypeDefinitionCustomizer
            get() = if (DartConfigurable.isExperimentalLspFeaturesEnabled(project)) {
                LspGoToTypeDefinitionSupport()
            } else {
                LspGoToTypeDefinitionDisabled
            }
```

Imports to add: `com.intellij.platform.dartlsp.api.customization.LspGoToTypeDefinitionCustomizer`,
`com.intellij.platform.dartlsp.api.customization.LspGoToTypeDefinitionSupport`.

(No SDK version gate: `TypeDefinitionHandler` has been a shared handler since Dart 3.3.0 — same
category as hover, spec decision S2.)

- [ ] **Step 3: Compile + LSP tests**

```bash
./gradlew compileKotlin compileJava && ./gradlew test --tests "com.jetbrains.lang.dart.lsp.*"
```

- [ ] **Step 4: CHANGELOG** under `## Unreleased` / `### Added`:

```markdown
- Go To Type Declaration for Dart via LSP textDocument/typeDefinition (JetBrains LSP, experimental feature) (#PR)
```

- [ ] **Step 5: Commit**

```bash
git add third_party/src/main/java/com/jetbrains/lang/dart/lsp/LspMethod.kt \
        third_party/src/main/java/com/jetbrains/lang/dart/lsp/DartLspServerDescriptor.kt \
        third_party/CHANGELOG.md
git commit -m "Enable Go To Type Declaration via LSP typeDefinition

The bundled LSP client's LspImplicitReferenceProvider answers the
GotoTypeDeclarationAction with textDocument/typeDefinition results; the
plugin registers no TypeDeclarationProvider, so there is no legacy path
to gate. Gated by the experimental-LSP setting only (the handler has been
shared over LSP-over-Legacy since Dart 3.3.0)."
```

### Task 7: Verify end-to-end, review, open PR B

- [ ] **Step 1: Sandbox verification**

```bash
./gradlew clean prepareSandbox --no-build-cache && ./gradlew runIde   # background
```

Flag ON, any supported SDK: in `void main() { final list = <String>['a']; final s = list.first; }`
put the caret on `list` and invoke *Navigate | Type Declaration* (⌘⇧B / Ctrl+Shift+B) → the editor
jumps to `class List` in the SDK; on `s` → `class String`. Also: caret on a variable whose type is
declared in a `.pub-cache` package; file open at startup vs opened later; flag OFF → the action does
nothing for Dart (today's behaviour — there is no legacy implementation); no errors in `idea.log`.
Note the known platform limitation for the PR body: Ctrl+hover does not underline type-declaration
targets in LSP-backed files (`LspImplicitReferenceProvider` only reacts to the explicit action).

- [ ] **Step 2: Verifier + code review** — same commands as Task 3 Steps 2–3, on `git diff lsp-inlay-hints...HEAD`.

- [ ] **Step 3: Push and open PR B (stacked)**

```bash
git push -u origin lsp-type-definition
gh pr create --repo flutter/dart-intellij-third-party --base lsp-inlay-hints \
  --title "Go To Type Declaration via LSP typeDefinition" \
  --body "Implements #580 with the same pattern as #539/#552 and the inlay-hints PR this is stacked on: the bridge forwards \`textDocument/typeDefinition\` (advertising \`typeDefinition.linkSupport\` like \`definition\`), \`LspMethod.TYPE_DEFINITION\` is added, and \`goToTypeDefinitionCustomizer\` is enabled behind the experimental-LSP setting. The bundled client's \`LspImplicitReferenceProvider\`/\`LspSymbolTypeProvider\` drive *Navigate | Type Declaration*.

**Stacked on** the inlay-hints PR (base branch \`lsp-inlay-hints\`); only the last commits are new. Will be retargeted to \`main\` once that PR merges.

**Manual test:** flag on, caret on a variable → ⌘⇧B jumps to the type's declaration (SDK, package, project). Flag off → unchanged (no Dart implementation existed).

**Functional differences:** none to gate — the plugin never registered a \`TypeDeclarationProvider\`. Platform limitation: Ctrl+hover does not underline type-declaration targets for LSP-backed files. No SDK version gate: the handler is shared over LSP-over-Legacy since Dart 3.3.0.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

Fill in `#PR` in the CHANGELOG line, amend, `git push --force-with-lease`.

- [ ] **Step 4: After PR A merges** — `gh pr edit <B> --base main`, rebase `lsp-type-definition`
onto `main`, `git push --force-with-lease`, re-run the LSP tests.

---

## Self-review checklist (done during plan writing)

- Spec coverage: S1 (Tasks 1–7), S2 (Task 6 Step 2 — no version gate), S3 (Task 5 — linkSupport in
  `buildLspCapabilities`, dependency on #614 spelled out with a non-silent fallback), S6 (rebase notes
  in Global Constraints and Task 5), S8 (no SDK task). Usage count/#396/#400 intentionally absent
  (spec S4/S7 — waiting for maintainer answers).
- Placeholders: the only deferred values are the PR numbers, resolved by their own steps.
- Type consistency: `forwardRequest<T>(String, Any?, Type)` matches the private API;
  `typeDefinition` returns `Either<List<Location>, List<LocationLink>>` exactly like `definition`;
  `LspMethod` constructor `(method, isExperimental, presentableName)`; `buildLspCapabilities(String):
  JsonObject` as introduced by #614.
