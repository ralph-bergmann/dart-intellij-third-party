# Dart Inlay Hints & Read/Write Highlighting Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development
> (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use
> checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enable LSP inlay hints and LSP read/write occurrence highlighting in the Dart IntelliJ
plugin, plus a clarifying rename — delivered as **three independent branches/PRs**.

**Architecture:** All features ride the existing DAS→LSP bridge (`DartBridgeLspServer` forwards
LSP requests through the legacy `lsp.handle` request), gated by the *"Turn on experimental LSP
features"* setting; inlay hints additionally gate on an SDK minimum version. No new provider
classes — the bundled JetBrains LSP client framework (`com.intellij.platform.dartlsp`) already
renders inlay hints and document highlights once the customizers are enabled.

**Tech Stack:** Kotlin/Java, IntelliJ Platform Gradle plugin (run Gradle from `third_party/`),
lsp4j, existing test bases `DartBridgeLspServerTest` / `DartCodeInsightFixtureTestCase`.

**Spec:** `docs/superpowers/specs/2026-07-30-dart-inlay-hints-and-highlighting-design.md`

## Global Constraints

- Repository remote setup: `origin` = `ralph-bergmann/dart-intellij-third-party` (fork). PRs target
  `flutter/dart-intellij-third-party`, base `main`.
- Each part starts from a fresh branch off `main` (`git checkout main && git pull origin main`).
  The planning docs (`docs/superpowers/**`) live only on the `DartInlayHints` planning branch —
  do NOT include them in the feature branches.
- NEVER modify files under `third_party/thirdPartySrc/` (bulk-copied upstream sources).
- Kotlin: no `!!`, prefer `val`, imports instead of fully qualified names — except mirror the
  existing inline `com.google.gson.reflect.TypeToken` usage in `DartBridgeLspServer`.
- All Gradle commands run from `third_party/`.
- CHANGELOG entries go under `## Unreleased` in `third_party/CHANGELOG.md`, phrased like the
  existing entries (gerund/noun style, e.g. "Highlighting …", not "Add …"), with the PR number.
- Commits: short imperative subject, ending with
  `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>`.
- PR bodies: describe behavior + trade-offs (per the migrate-das-to-lsp skill: "Always explicitly
  document any functional differences"), reference the issues named per part, and end with
  `🤖 Generated with [Claude Code](https://claude.com/claude-code)`.

---

# Part 1 — PR "Rename closing-labels provider" (branch `rename-closing-labels-provider`)

Independent of everything else; mergeable immediately. Not user-facing → **no CHANGELOG entry**.

### Task 1.1: Rename `DartInlayHintsProvider` → `DartClosingLabelsInlayHintsProvider`

**Files:**
- Rename: `third_party/src/main/java/com/jetbrains/lang/dart/hints/DartInlayHintsProvider.kt`
  → `third_party/src/main/java/com/jetbrains/lang/dart/hints/DartClosingLabelsInlayHintsProvider.kt`
- Modify: `third_party/src/main/java/com/jetbrains/lang/dart/analyzer/DartClosingLabelManager.java`
  (import on line 8, `PROVIDER_ID` usages on lines 19 and 28)
- Modify: `third_party/src/main/resources/META-INF/plugin.xml` (line ~157, `implementationClass`)

**Interfaces:**
- Produces: class `com.jetbrains.lang.dart.hints.DartClosingLabelsInlayHintsProvider` with unchanged
  `companion object { const val PROVIDER_ID: String = "dart.closing.labels" }`.
- Consumes: nothing from other tasks.

- [ ] **Step 1: Create the branch**

```bash
git checkout main && git pull origin main
git checkout -b rename-closing-labels-provider
```

- [ ] **Step 2: Rename the file and class**

```bash
git mv third_party/src/main/java/com/jetbrains/lang/dart/hints/DartInlayHintsProvider.kt \
       third_party/src/main/java/com/jetbrains/lang/dart/hints/DartClosingLabelsInlayHintsProvider.kt
```

In the renamed file change only the class name (body stays identical):

```kotlin
class DartClosingLabelsInlayHintsProvider : InlayHintsProvider {
```

- [ ] **Step 3: Update references**

`DartClosingLabelManager.java`: change the import to
`com.jetbrains.lang.dart.hints.DartClosingLabelsInlayHintsProvider` and both usages to
`DartClosingLabelsInlayHintsProvider.PROVIDER_ID`.

`plugin.xml`: in the `<codeInsight.declarativeInlayProvider …>` registration change
`implementationClass` to `com.jetbrains.lang.dart.hints.DartClosingLabelsInlayHintsProvider`.
All other attributes (`providerId="dart.closing.labels"`, `nameKey`, `group`, …) stay unchanged.

- [ ] **Step 4: Verify no stale references remain**

```bash
grep -rn "DartInlayHintsProvider" third_party/src third_party/gen
```

Expected: no matches.

- [ ] **Step 5: Compile and run the unit test suite**

```bash
cd third_party && ./gradlew compileKotlin compileJava && ./gradlew test --tests "com.jetbrains.lang.dart.*"
```

Expected: BUILD SUCCESSFUL, tests green.

- [ ] **Step 6: Commit, push, open PR**

```bash
git add -A && git commit -m "Rename DartInlayHintsProvider to DartClosingLabelsInlayHintsProvider

The class only renders closing labels. With LSP-provided inlay hints
coming (issue #159), the old name would be misleading. providerId
\"dart.closing.labels\" is unchanged, so user settings are unaffected.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
git push -u origin rename-closing-labels-provider
gh pr create --repo flutter/dart-intellij-third-party --base main \
  --title "Rename DartInlayHintsProvider to DartClosingLabelsInlayHintsProvider" \
  --body "The class only renders closing labels; with LSP inlay hints planned (#159) the old name becomes misleading. \`providerId\` (\"dart.closing.labels\") and all registration attributes are unchanged, so persisted user settings are unaffected. No functional change.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

---

# Part 2 — PR "Read/write highlighting via LSP" (branch `lsp-document-highlight`)

Independent of Part 1 and Part 3; no SDK change needed (`textDocument/documentHighlight` is already
a shared handler; read/write kinds exist since dart-lang/sdk#62929). PR references
flutter/dart-intellij-third-party#546 (lists this endpoint as required) and #92 (object-pattern
highlights, fixed in LSP mode).

### Task 2.1: Forward `textDocument/documentHighlight` in the bridge (TDD)

**Files:**
- Test: `third_party/src/test/java/com/jetbrains/lang/dart/lsp/DartBridgeLspServerTest.kt`
- Modify: `third_party/src/main/java/com/jetbrains/lang/dart/lsp/DartBridgeLspServer.kt`

**Interfaces:**
- Produces: `DartBridgeLspServer.documentHighlight(params): CompletableFuture<List<DocumentHighlight>>`;
  `initialize()` advertises `documentHighlightProvider`.
- Consumes: existing `forwardRequest(method, params, responseType)` and the test fixtures
  `capturedRequests` / `capturedListener` already present in `DartBridgeLspServerTest`.

- [ ] **Step 1: Create the branch**

```bash
git checkout main && git pull origin main
git checkout -b lsp-document-highlight
```

- [ ] **Step 2: Write the failing test**

Add to `DartBridgeLspServerTest.kt` (imports to add:
`org.eclipse.lsp4j.DocumentHighlightKind`, `org.eclipse.lsp4j.DocumentHighlightParams`):

```kotlin
fun testDocumentHighlightRequest() {
    val params = DocumentHighlightParams().apply {
        textDocument = TextDocumentIdentifier("file://test.dart")
        position = Position(1, 2)
    }

    val future = bridgeServer.documentHighlight(params)

    val jsonObject = capturedRequests.find { it.get("method")?.asString == "lsp.handle" }
    assertNotNull("An lsp.handle request should be sent to DAS", jsonObject)

    val lspMessage = jsonObject!!.getAsJsonObject("params").getAsJsonObject("lspMessage")
    assertEquals("123", lspMessage.get("id").asString)
    assertEquals("textDocument/documentHighlight", lspMessage.get("method").asString)

    val responseJson = """
        {
          "id": "123",
          "result": {
            "lspResponse": {
              "jsonrpc": "2.0",
              "id": "123",
              "result": [
                {"range": {"start": {"line": 0, "character": 4}, "end": {"line": 0, "character": 5}}, "kind": 3},
                {"range": {"start": {"line": 2, "character": 2}, "end": {"line": 2, "character": 3}}, "kind": 2}
              ]
            }
          }
        }
    """.trimIndent()

    capturedListener.onResponse(responseJson)

    val result = future.get(5, TimeUnit.SECONDS)
    assertEquals(2, result.size)
    assertEquals(DocumentHighlightKind.Write, result[0].kind)
    assertEquals(DocumentHighlightKind.Read, result[1].kind)
}
```

- [ ] **Step 3: Run the test — expect compile failure**

```bash
cd third_party && ./gradlew test --tests "com.jetbrains.lang.dart.lsp.DartBridgeLspServerTest"
```

Expected: FAILS — `documentHighlight` is not overridden (unresolved reference / lsp4j default
throws `UnsupportedOperationException`).

- [ ] **Step 4: Implement the bridge forwarding**

In `DartBridgeLspServer.kt` (imports to add: `org.eclipse.lsp4j.DocumentHighlight`,
`org.eclipse.lsp4j.DocumentHighlightParams`):

In `initialize()`, extend the capabilities block:

```kotlin
val capabilities = ServerCapabilities().apply {
    setHoverProvider(true)
    setDefinitionProvider(true)
    setDocumentHighlightProvider(true)
    // Add other capabilities as we support them.
}
```

Below `definition(...)`, add (mirroring its `TypeToken` style):

```kotlin
override fun documentHighlight(params: DocumentHighlightParams): CompletableFuture<List<DocumentHighlight>> {
    val type = object : com.google.gson.reflect.TypeToken<List<DocumentHighlight>>() {}.type
    return forwardRequest<List<DocumentHighlight>>("textDocument/documentHighlight", params, type)
}
```

- [ ] **Step 5: Run the test — expect pass**

```bash
./gradlew test --tests "com.jetbrains.lang.dart.lsp.DartBridgeLspServerTest"
```

- [ ] **Step 6: Commit**

```bash
git add -A && git commit -m "Forward textDocument/documentHighlight in DartBridgeLspServer

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

### Task 2.2: Enable the feature (descriptor + LspMethod + changelog)

**Files:**
- Modify: `third_party/src/main/java/com/jetbrains/lang/dart/lsp/DartLspServerDescriptor.kt`
- Modify: `third_party/src/main/java/com/jetbrains/lang/dart/lsp/LspMethod.kt`
- Modify: `third_party/CHANGELOG.md`
- Test: `third_party/src/test/java/com/jetbrains/lang/dart/lsp/DartLspExperimentalFeaturesTest.java`
  (existing — must stay green)

**Interfaces:**
- Consumes: `DartBridgeLspServer.documentHighlight` from Task 2.1;
  `DartConfigurable.isExperimentalLspFeaturesEnabled(project)` (existing).
- Produces: `LspMethod.DOCUMENT_HIGHLIGHT`; an enabled `documentHighlightsCustomizer`.

- [ ] **Step 1: Add the LspMethod entry**

In `LspMethod.kt`, after `DIAGNOSTIC_SERVER`:

```kotlin
DOCUMENT_HIGHLIGHT("textDocument/documentHighlight", isExperimental = true, presentableName = "read/write highlighting"),
```

- [ ] **Step 2: Enable the customizer in the descriptor**

In `DartLspServerDescriptor.kt` replace
`override val documentHighlightsCustomizer = LspDocumentHighlightsDisabled` with:

```kotlin
override val documentHighlightsCustomizer: LspDocumentHighlightsCustomizer
    get() = if (DartConfigurable.isExperimentalLspFeaturesEnabled(project)) {
        object : LspDocumentHighlightsSupport() {
            // The default implementation only serves plain-text/TextMate files.
            override fun shouldAskServerForDocumentHighlights(psiFile: PsiFile): Boolean = true
        }
    } else {
        LspDocumentHighlightsDisabled
    }
```

Imports to add: `com.intellij.platform.dartlsp.api.customization.LspDocumentHighlightsCustomizer`,
`com.intellij.platform.dartlsp.api.customization.LspDocumentHighlightsSupport`,
`com.intellij.psi.PsiFile`.

- [ ] **Step 3: Run the LSP test suites**

```bash
./gradlew test --tests "com.jetbrains.lang.dart.lsp.*"
```

Expected: PASS (the experimental-features label test picks up the new presentable name
automatically; if it asserts the literal feature list, update the expected string).

- [ ] **Step 4: Add the CHANGELOG entry**

Under `## Unreleased` / `### Added`:

```markdown
- Highlighting read vs write variable occurrences (JetBrains LSP, experimental feature) (#PR)
```

(Replace `#PR` with the actual PR number after opening the PR — amend the commit.)

- [ ] **Step 5: Manual sandbox verification**

```bash
./gradlew clean prepareSandbox --no-build-cache && ./gradlew runIde
```

In the sandbox IDE: enable *Settings | Languages & Frameworks | Dart | Turn on experimental LSP
features*; open a Dart file with `var a = ''; a = 'x'; print(a);`; caret on `a` — the assignment
occurrence must use the write color ("Write identifier under caret"), the `print(a)` occurrence the
read color. Also verify a file from `.pub-cache` and toggling the setting off restores today's
uniform highlighting.

- [ ] **Step 6: Commit, push, open PR**

```bash
git add -A && git commit -m "Highlight read vs write occurrences via LSP documentHighlight

Enables the JetBrains LSP documentHighlight feature (experimental flag):
the Dart Analysis Server returns DocumentHighlightKind Read/Write/Text
(dart-lang/sdk#62929), which the platform maps to the read/write caret
colors. Older SDKs without kinds degrade to the current all-read look.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
git push -u origin lsp-document-highlight
gh pr create --repo flutter/dart-intellij-third-party --base main \
  --title "Highlight read vs write occurrences via LSP documentHighlight" \
  --body "Enables LSP \`textDocument/documentHighlight\` through the bridge behind the experimental-LSP setting. With current SDKs the server sets \`DocumentHighlightKind\` Read/Write/Text (dart-lang/sdk#62929), so writes get the write caret color — matching Java/Kotlin behavior. Contributes one of the endpoints listed in #546; object-pattern highlights (#92) work in LSP mode.

Trade-offs: on SDKs without highlight kinds everything renders as read (today's behavior). Find Usages read/write classification is not covered by LSP documentHighlight (would need a \`ReadWriteAccessDetector\`, see design doc).

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

Then replace `#PR` in the CHANGELOG entry with the created PR number and
`git commit --amend --no-edit && git push --force-with-lease`.

---

# Part 3 — PR "Inlay hints via LSP" (branch `lsp-inlay-hints`)

**Prerequisites (external, done by Ralph in the `../sdk` checkout):**

- P1: The SDK change from `docs/superpowers/plans/2026-07-30-sdk-share-inlay-hint-handler.md`
  has landed on dart-lang/sdk `main`.
- P2: The first dev version tag containing it is known
  (`git -C ../sdk tag --contains <sha> | sort -V | head -1`, format `3.14.0-NNN.0.dev`).
  This value is referred to as `INLAY_HINTS_MIN_SDK` below.

Tasks 3.1 can be implemented and reviewed before P1/P2; the PR must not merge before P2 fills in
the version constant. PR references flutter/dart-intellij-third-party#159.

### Task 3.1: Forward `textDocument/inlayHint` in the bridge (TDD)

**Files:**
- Test: `third_party/src/test/java/com/jetbrains/lang/dart/lsp/DartBridgeLspServerTest.kt`
- Modify: `third_party/src/main/java/com/jetbrains/lang/dart/lsp/DartBridgeLspServer.kt`

**Interfaces:**
- Produces: `DartBridgeLspServer.inlayHint(params): CompletableFuture<List<InlayHint>>`;
  `initialize()` advertises `inlayHintProvider`.
- Consumes: existing `forwardRequest` + test fixtures (as in Task 2.1).

- [ ] **Step 1: Create the branch**

```bash
git checkout main && git pull origin main
git checkout -b lsp-inlay-hints
```

- [ ] **Step 2: Write the failing test**

Add to `DartBridgeLspServerTest.kt` (imports to add: `org.eclipse.lsp4j.InlayHintKind`,
`org.eclipse.lsp4j.InlayHintParams`, `org.eclipse.lsp4j.Range`):

```kotlin
fun testInlayHintRequest() {
    val params = InlayHintParams().apply {
        textDocument = TextDocumentIdentifier("file://test.dart")
        range = Range(Position(0, 0), Position(10, 0))
    }

    val future = bridgeServer.inlayHint(params)

    val jsonObject = capturedRequests.find { it.get("method")?.asString == "lsp.handle" }
    assertNotNull("An lsp.handle request should be sent to DAS", jsonObject)

    val lspMessage = jsonObject!!.getAsJsonObject("params").getAsJsonObject("lspMessage")
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

- [ ] **Step 3: Run the test — expect failure**

```bash
cd third_party && ./gradlew test --tests "com.jetbrains.lang.dart.lsp.DartBridgeLspServerTest"
```

- [ ] **Step 4: Implement the bridge forwarding**

In `DartBridgeLspServer.kt` (imports to add: `org.eclipse.lsp4j.InlayHint`,
`org.eclipse.lsp4j.InlayHintParams`):

`initialize()` capabilities block gains:

```kotlin
setInlayHintProvider(true)
```

Below `documentHighlight(...)` (or `definition(...)` if Part 2 is not merged yet), add:

```kotlin
override fun inlayHint(params: InlayHintParams): CompletableFuture<List<InlayHint>> {
    val type = object : com.google.gson.reflect.TypeToken<List<InlayHint>>() {}.type
    return forwardRequest<List<InlayHint>>("textDocument/inlayHint", params, type)
}
```

- [ ] **Step 5: Run the test — expect pass, then commit**

```bash
./gradlew test --tests "com.jetbrains.lang.dart.lsp.DartBridgeLspServerTest"
git add -A && git commit -m "Forward textDocument/inlayHint in DartBridgeLspServer

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

### Task 3.2: SDK version gate + enable the feature

**Files:**
- Modify: `third_party/src/main/java/com/jetbrains/lang/dart/analyzer/DartAnalysisServerService.java`
- Modify: `third_party/src/main/java/com/jetbrains/lang/dart/lsp/LspMethod.kt`
- Modify: `third_party/src/main/java/com/jetbrains/lang/dart/lsp/DartLspServerDescriptor.kt`
- Modify: `third_party/CHANGELOG.md`

**Interfaces:**
- Produces: `DartAnalysisServerService.isLspInlayHintsEnabled(Project)`,
  `MIN_LSP_INLAY_HINTS_SDK_VERSION`; `LspMethod.INLAY_HINT`; enabled `inlayHintCustomizer`.
- Consumes: `DartBridgeLspServer.inlayHint` from Task 3.1; prerequisite value
  `INLAY_HINTS_MIN_SDK` from P2.

- [ ] **Step 1: Add the version gate to `DartAnalysisServerService`**

Next to `MIN_LSP_NAVIGATION_SDK_VERSION` (line ~184) add the constant, filling in the value from
prerequisite P2:

```java
public static final String MIN_LSP_INLAY_HINTS_SDK_VERSION = "<INLAY_HINTS_MIN_SDK from P2>";
```

Next to `isDartSdkVersionSufficientForLspNavigation` / `isLspNavigationEnabled` (lines ~583–594)
add, mirroring them exactly:

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

- [ ] **Step 2: Add the LspMethod entry**

```kotlin
INLAY_HINT("textDocument/inlayHint", isExperimental = true, presentableName = "inlay hints"),
```

- [ ] **Step 3: Enable the customizer in the descriptor**

Replace `override val inlayHintCustomizer = LspInlayHintDisabled` with:

```kotlin
override val inlayHintCustomizer: LspInlayHintCustomizer
    get() = if (DartAnalysisServerService.isLspInlayHintsEnabled(project)) {
        LspInlayHintSupport()
    } else {
        LspInlayHintDisabled
    }
```

Imports to add: `com.intellij.platform.dartlsp.api.customization.LspInlayHintCustomizer`,
`com.intellij.platform.dartlsp.api.customization.LspInlayHintSupport`.

- [ ] **Step 4: Run the LSP test suites**

```bash
./gradlew test --tests "com.jetbrains.lang.dart.lsp.*"
```

- [ ] **Step 5: Add the CHANGELOG entry**

Under `## Unreleased` / `### Added`:

```markdown
- Inlay hints for types and parameter names (JetBrains LSP, experimental feature; requires a recent Dart SDK) (#PR)
```

- [ ] **Step 6: End-to-end verification**

Requires an SDK containing the upstream change — either the dev-channel SDK ≥ `INLAY_HINTS_MIN_SDK`
(https://dart.dev/get-dart/archive, dev channel) or a locally built `../sdk` with the patch.

```bash
./gradlew clean prepareSandbox --no-build-cache && ./gradlew runIde
```

In the sandbox IDE (experimental LSP flag ON, project SDK = the new dev SDK): open a Dart file with
`var a = ''; print(int.parse('1'));` — expect a `String` type hint after `a` and a `radix:`-style
parameter-name situation on calls with positional args (e.g. `name:` hints). Verify closing labels
still render, and that an old SDK (< `INLAY_HINTS_MIN_SDK`) shows no hints and logs no errors.

- [ ] **Step 7: Commit, push, open PR**

```bash
git add -A && git commit -m "Show LSP inlay hints for Dart as an experimental feature

Forwards textDocument/inlayHint through the DAS bridge and enables the
JetBrains LSP inlay hint rendering when the experimental-LSP setting is
on and the SDK is recent enough to serve inlayHint over LSP-over-Legacy.

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
git push -u origin lsp-inlay-hints
gh pr create --repo flutter/dart-intellij-third-party --base main \
  --title "Show LSP inlay hints for Dart as an experimental feature" \
  --body "Implements #159 via the LSP bridge: the Dart Analysis Server's inlay hints (variable types, parameter names, return types, type arguments, dot-shorthand types) render through the bundled LSP client. Gated by the experimental-LSP setting plus \`MIN_LSP_INLAY_HINTS_SDK_VERSION\` (the SDK gained \`textDocument/inlayHint\` over LSP-over-Legacy in dart-lang/sdk#NNNNN).

Trade-offs (documented in the design doc): hint categories follow server defaults — no per-category settings UI yet (\`workspace/didChangeConfiguration\` is not available over the legacy protocol); labels truncate at 42 chars (framework default). Closing labels are unaffected.

🤖 Generated with [Claude Code](https://claude.com/claude-code)"
```

(Replace `NNNNN` with the SDK issue number; update `#PR` in the CHANGELOG as in Part 2.)

---

## Self-review checklist (done during plan writing)

- Spec coverage: Part 1 = spec §7; Part 2 = spec §5; Part 3 = spec §4 + the SDK handoff doc;
  spec §6 (usage count) is deliberately design-only — no task, per decision D3.
- Type consistency: `forwardRequest<T>(String, Any?, Type)` matches the existing private API;
  lsp4j overrides mirror the proven `definition(...)` pattern (Kotlin drops Java wildcards).
- Placeholders: the only deferred value is `INLAY_HINTS_MIN_SDK`, produced by prerequisite P2 with
  an exact procedure; `#PR`/`#NNNNN` are resolved by their own steps.
