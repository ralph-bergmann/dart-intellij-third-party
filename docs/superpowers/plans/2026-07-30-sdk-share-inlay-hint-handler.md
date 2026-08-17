# Dart SDK task: Make `textDocument/inlayHint` a shared handler (LSP-over-Legacy)

> **Audience:** an AI coding agent (or human) working in a `dart-lang/sdk` checkout
> (e.g. `/Users/ralph.bergmann/development/projects/privat/sdk`). This document is
> self-contained — no other context is required.
>
> **Last verified against the real code and the issue trackers: 2026-08-17**
> (`dart-lang/sdk` main @ `144f5acb5db`). See "Verified facts" at the end for the evidence
> behind every claim in this document.

## Ground rules (apply to every step of this task)

These rules are binding. They exist so that the change is reviewable by the Dart team without
friction, and so that nothing in this document degrades into guesswork.

1. **Every decision and every change is validated.** Validation means: against an issue in the
   relevant repository, and/or against the actual source code in the checkout.
2. **Nothing is guessed, assumed by default, or invented.** If a fact cannot be validated, say so
   explicitly and ask, instead of writing something plausible. Statements in this document that
   *require* validation at execution time are marked **[verify]**.
3. **Follow the existing code, documentation and formatting.** Match the style of the surrounding
   files, not a personal preference. Reviewers should see something familiar and not have to learn a
   new idiom.
4. **Reuse existing code / classes / methods / test helpers where possible.** Do not add a new
   helper when the repository already has one.
5. Rules 3 and 4 hold **even when the existing code is suboptimal.** Do not "improve it on the way
   past". If you spot something that should change, write a separate markdown file with the
   proposal and leave the code alone.
6. **Commits must be authored by `Ralph Bergmann <ralph@dasralph.de>` only — no `Co-Authored-By:`
   trailer of any kind.** Google's CLA check treats a co-author as an additional contributor, and
   the check then fails because that contributor has not signed the CLA. Verify before committing:
   `git -C <sdk> config user.name` / `user.email`, and `git log -1 --format='%an <%ae> | %cn <%ce>'`
   afterwards.

Rationale for 3 and 4: consistency. Reviewers should recognise what they read instead of having to
work through something new.

## Status (2026-08-17)

* The conversion described here has **not** been done yet — `InlayHintHandler` is still an
  LSP-only handler on `dart-lang/sdk` main.
* No `dart-lang/sdk` issue asking for this exists yet, so **Step 0 is still required**.
* On the plugin side, Parts 1 and 2 of the inlay-hints plan have shipped:
  [#551](https://github.com/flutter/dart-intellij-third-party/pull/551) and
  [#552](https://github.com/flutter/dart-intellij-third-party/pull/552), both merged 2026-08-10.
  Part 3 (the plugin-side inlay hints) is blocked on this SDK change.

## Why

The Dart IntelliJ plugin (github.com/flutter/dart-intellij-third-party) talks to the Dart Analysis
Server via the **legacy protocol** and reaches LSP features through the `lsp.handle` request
("LSP-over-Legacy"). Only LSP handlers registered as *shared* handlers are reachable that way.

The inlay hint handler is currently **LSP-only**, so the IntelliJ plugin cannot request inlay hints
(type hints, parameter name hints, …) while it is being migrated to LSP. Calling
`textDocument/inlayHint` via `lsp.handle` today returns `MethodNotFound`.

This is the same situation `textDocument/definition` / `textDocument/references` were in until
2026-07-24, when they were converted with exactly the change described below:

* Issue: https://github.com/dart-lang/sdk/issues/63884 — *"Make textDocument/definition and
  textDocument/references shared handlers so they're available over DTD/Legacy"*, rationale:
  *"This would allow IntelliJ to use them via LoL while being migrated to LSP."*
* Fix: https://dart-review.googlesource.com/c/sdk/+/527540 (commit `c6d728bfd33`), 6 files,
  +121/−5. Commit message: *"Everything these require has already been abstracted to the base
  server now, so this was a trivial change of types (plus some basic tests) with no implementation
  changes."*

A second, slightly different precedent (an LSP-only handler made callable over LSP-over-Legacy) is
commit `eb362d426`, *"[analysis_server] Support diagnostic server request for LSP-over-Legacy"*
(2026-05-06). It is one of the reference commits listed in
https://github.com/dart-lang/sdk/issues/63878.

The inlay hint handler has the same property: everything it uses is already server-agnostic —
`pathContext`, `extractDocumentVersion`, `fileHasBeenModified`, `pathOfDoc` and
`requireResolvedUnit` all live on `HandlerHelperMixin<S extends AnalysisServer>`, `isDartDocument`
is a top-level function, and `lspClientConfiguration` is an abstract getter on the base
`AnalysisServer` that `LegacyAnalysisServer` already implements. The conversion is a pure type
change. (Evidence: "Verified facts" §2.)

Motivating downstream issue: https://github.com/flutter/dart-intellij-third-party/issues/159
("Add Inlay Hints to Dart code: variable type, parameter name, and more.") — still open.

## Scope boundary (decided upstream — do not widen)

This task is **only** the handler/registration type change plus a test. In particular, do **not**
extend `DartInlayHintComputer` to emit closing labels as inlay hints, and do not file an issue
proposing it. That direction was explicitly considered and parked by the analysis server
maintainer, DanTup, in the review of
[flutter/dart-intellij-third-party#551](https://github.com/flutter/dart-intellij-third-party/pull/551)
(merged 2026-08-10):

> *"FWIW I wanted to do this, but currently VS Code doesn't support custom kinds of inlay hints, so
> the styling of Closing Labels would become the same as other Inlay Hints and the customisation
> would be lost. […] If VS Code and LSP gain support for custom kinds (that we can customise the
> style of in the client), and IntelliJ could support the same, we should revisit it."*

The blocker he references is https://github.com/microsoft/vscode/issues/151920. Closing labels stay
on the custom `dart/textDocument/publishClosingLabels` notification until that changes.

## Repository conventions you must follow

Read these before writing any code; they are the repo's own instructions, and they take precedence
over anything in this document:

* **`.agents/rules/add_license_header.md`** — every newly created `.dart` file needs the standard
  license header on line 1, with the current year, followed by a blank line.
  ⚠️ The rule file renders the header with **two** spaces after `authors.`, but every one of the
  948 files under `pkg/analysis_server/test/` uses **one** space. Follow the surrounding files
  (one space, as in the snippet in Step 3) — per ground rule 3.
* **`.agents/skills/`** — repo skills (`read_gerrit_cl`, `find_release`, …). Use `read_gerrit_cl`
  when you need to read review comments on the uploaded CL.
* **`pkg/analysis_server/doc/implementation/coding_style.md`** — analysis server coding style.
  (Note: `pkg/analysis_server/doc/implementation/handlers.md` exists but is an empty stub — do not
  expect guidance there.)
* **`docs/Commit-Message-Best-Practices.md`** — component prefix in the subject, blank line before
  the body, present tense, full GitHub URL for issue links, GitHub auto-close notation.
* **`CONTRIBUTING.md`** (repo root) — CLA and the Gerrit workflow; see Step 6.

## Step 0: File the GitHub issue

**[verify]** first that no such issue has appeared in the meantime:

```bash
gh api -X GET search/issues --raw-field q='repo:dart-lang/sdk inlay in:title' \
  --jq '.items[] | "\(.number) [\(.state)] \(.title)"'
```

As of 2026-08-17 the results are #61292, #61841, #60145, #48972 — none of them is about sharing the
handler. If that is still the case, file a new issue on https://github.com/dart-lang/sdk
(area-devexp / devexp-lsp, the labels #63884 carries). Suggested text:

> **Title:** Make textDocument/inlayHint a shared handler so it's available over DTD/Legacy
>
> **Body:**
> Similar to #63884: `InlayHintHandler` is currently registered in
> `InitializedLspStateMessageHandler.lspHandlerGenerators` and therefore not available to
> LSP-over-Legacy/DTD clients.
>
> Making it a `SharedMessageHandler` would allow the Dart IntelliJ plugin to show inlay hints
> (types, parameter names) via LSP-over-Legacy while it is being migrated to LSP
> (flutter/dart-intellij-third-party#159).
>
> Like the handlers converted in #63884, `InlayHintHandler` only uses base-server facilities
> (`requireResolvedUnit`, document versions, `lspClientConfiguration`), so this should again be a
> trivial change of types plus a test.

Record the issue number — the commit message references it (`Fixes https://…/issues/NNNNN`).

## Step 1: Convert the handler

File: `pkg/analysis_server/lib/src/lsp/handlers/handler_inlay_hint.dart`

Change the base class and opt out of the trusted-caller restriction (mirror commit `c6d728bfd33`,
which did the same for `DefinitionHandler`/`ReferencesHandler`):

```dart
// Before:
class InlayHintHandler
    extends LspMessageHandler<InlayHintParams, List<InlayHint>> {
  new(super.server);
  @override
  Method get handlesMessage => Method.textDocument_inlayHint;

// After:
class InlayHintHandler
    extends SharedMessageHandler<InlayHintParams, List<InlayHint>> {
  new(super.server);
  @override
  Method get handlesMessage => Method.textDocument_inlayHint;

  @override
  bool get requiresTrustedCaller => false;
```

The `requiresTrustedCaller` override is **not optional**: `LspMessageHandler` supplies a default of
`true`, but `SharedMessageHandler` does not declare the member at all, so it stays abstract from
`MessageHandler` and the class will not compile without it. Returning `false` is what makes the
handler callable by untrusted DTD clients as well; the converted definition, references and
documentHighlights handlers all return `false`.

Keep the rest of the file unchanged — in particular do not touch `InlayHintRegistrations`, which
governs LSP capability registration and is unrelated to LSP-over-Legacy.

## Step 2: Move the registration

File: `pkg/analysis_server/lib/src/lsp/handlers/handler_states.dart`

* Remove `InlayHintHandler.new,` from `InitializedLspStateMessageHandler.lspHandlerGenerators`
  (line 105 as of `144f5acb5db`).
* Add `InlayHintHandler.new,` to `InitializedStateMessageHandler.sharedHandlerGenerators`,
  keeping the list's alphabetical order — between `IncomingCallHierarchyHandler.new` (line 147) and
  `InlineValueHandler.new` (line 148).

## Step 3: Add an LSP-over-Legacy test

New file: `pkg/analysis_server/test/lsp_over_legacy/inlay_hint_test.dart`

Model it on `definition_test.dart` / `references_test.dart` in the same directory — those two files
were added by `c6d728bfd33`, i.e. by exactly the CL this task mirrors. Note what they do and copy
it: `TestCode.parse`, `newFile(testFilePath, code.code)`, `await waitForTasksFinished()`, and a
doc comment pointing at the fuller LSP-suite test.

Helpers that are available here (**verified**, see §3): `getInlayHints(Uri uri, Range range)` from
`LspRequestHelpersMixin`, `TestCode.parse`, and `code.range.range` via
`../utils/test_code_extensions.dart`.
Helper that is **not** available here: `rangeOfWholeContent` — it lives on
`LspAnalysisServerTestMixin`, which `LspOverLegacyTest` does not mix in. Use a `[!…!]` marked range
from `TestCode` instead; that is the idiom the neighbouring tests use for ranges.

```dart
// Copyright (c) 2026, the Dart project authors. Please see the AUTHORS file
// for details. All rights reserved. Use of this source code is governed by a
// BSD-style license that can be found in the LICENSE file.

import 'package:analysis_server/lsp_protocol/protocol.dart';
import 'package:analyzer/src/test_utilities/test_code_format.dart';
import 'package:test/test.dart';
import 'package:test_reflective_loader/test_reflective_loader.dart';

import '../utils/test_code_extensions.dart';
import 'abstract_lsp_over_legacy.dart';

void main() {
  defineReflectiveSuite(() {
    defineReflectiveTests(InlayHintTest);
  });
}

/// More complete tests for textDocument/inlayHint are in
/// 'test/lsp/inlay_hint_test.dart'.
@reflectiveTest
class InlayHintTest extends LspOverLegacyTest {
  Future<void> test_variableType() async {
    var contents = '''
[!final a = 1;!]
''';

    var code = TestCode.parse(contents);
    newFile(testFilePath, code.code);
    await waitForTasksFinished();

    var hints = await getInlayHints(testFileUri, code.range.range);

    expect(hints, hasLength(1));
    expect(hints.single.kind, InlayHintKind.Type);
  }
}
```

**[verify] the expectations by running the test (Step 5) and adjust them to what the server
actually reports — do not guess.** What is already verified: `final a = 1;` produces at least one
hint (`test/lsp/inlay_hint_test.dart`, `test_documentUpdates` asserts `isNotEmpty` for exactly this
content), and the analogous class-field case `final i1 = 1;` renders as `final (Type:int) i1 = 1;`,
i.e. a single `Type` hint. The exact count for the top-level case is what you must confirm.

Also note (verified in the handler source): the current handler **ignores `params.range`** and
always computes hints for the whole unit. Pass a real range anyway — the test should express the
intended contract, not depend on that detail.

## Step 4: Register the test

File: `pkg/analysis_server/test/lsp_over_legacy/test_all.dart`

Add the import alphabetically — between `implementation_test.dart` and `references_test.dart`:

```dart
import 'inlay_hint_test.dart' as inlay_hint;
```

and, in the same position inside `main()`:

```dart
inlay_hint.main();
```

## Step 5: Verify

`pkg/analysis_server/CONTRIBUTING.md` documents these commands (run from the repo root):

```bash
dart test pkg/analysis_server/test/lsp_over_legacy/inlay_hint_test.dart
dart test pkg/analysis_server/test/lsp_over_legacy/test_all.dart   # full LSP-over-Legacy suite
dart analyze pkg/analysis_server
```

All tests must pass; `dart analyze` must be clean.

⚠️ `pkg/analysis_server/CONTRIBUTING.md` notes that `dart` may need to point at a Dart SDK built
from source, depending on the change. The full suite is normally run via
`./tools/test.py -mrelease pkg/analysis_server/test/`, which needs a built SDK.
⚠️ **Environment caveat:** the checkout at `../sdk` is a plain `git clone`. `CONTRIBUTING.md` states:
*"You must use the `gclient` tool (`fetch`), using `git clone` will not get you a functional
environment!"* If the tests cannot be run in this checkout, **[verify]** how to proceed and report
back — do not skip verification and do not claim the tests passed.

## Step 6: Submit

Follow `CONTRIBUTING.md` in the SDK root. The CLA must be signed by the commit author, which is why
ground rule 6 exists.

Two documented paths:

1. **Gerrit (the main flow).** Requires depot_tools:
   `git new-branch <name>` → commit → `git cl upload` → `git cl web`. See
   `docs/Code-review-workflow-with-GitHub-and-Gerrit.md`.
2. **GitHub PR.** `CONTRIBUTING.md` states that the repo "occasionally take[s] pull requests" and
   that a PR "will be automatically converted into a Gerrit change list (a 'CL') by a
   copybara-service bot", with the CL link posted as a comment. This is the fallback if depot_tools
   is unavailable in this checkout.

**[verify] which path to use before uploading** — this is a code change, not a doc tweak, and the
main flow is Gerrit.

Suggested commit message, mirroring `c6d728bfd33` and `docs/Commit-Message-Best-Practices.md`:

```
[analysis_server] Make textDocument/inlayHint a shared handler

This allows IntelliJ to call it via LSP-over-Legacy while being
migrated to LSP.

InlayHintHandler only uses base-server facilities, so this is a change
of types plus a test, with no implementation changes.

Fixes https://github.com/dart-lang/sdk/issues/NNNNN
```

(Replace `NNNNN` with the issue number from Step 0. `Fixes` — not `Closes` — matches `c6d728bfd33`
and is the more common form in `pkg/analysis_server` history: 90 vs 52 occurrences in the last 500
commits touching that package. No `Co-Authored-By` trailer — see ground rule 6.)

## Step 7: Report back (needed by the IntelliJ plugin)

After the CL lands, determine the **first dev version tag that contains the commit**:

```bash
git fetch origin
git tag --contains <landed-commit-sha> | sort -V | head -1
```

The result (format like `3.14.0-65.0.dev`) is required by the IntelliJ plugin as
`MIN_LSP_INLAY_HINTS_SDK_VERSION` — see Part 3 of
`docs/superpowers/plans/2026-07-30-dart-inlay-hints.md` in the `dart-intellij-third-party`
repository.

Context on why a version constant is the right mechanism: in #63884 DanTup asked whether the plugin
detects availability by checking SDK versions or whether the server needs to expose a capability.
**That question was never answered in the issue** (the issue has no comments and was closed by the
fix). The plugin's established practice answers it in code — there are two precedents:

* `MIN_LSP_NAVIGATION_SDK_VERSION = "3.14.0-65.0.dev"` — for the #63884 conversion
  (`DartAnalysisServerService.java:184`).
* `MIN_LSP_DIAGNOSTIC_SERVER_SDK_VERSION = "3.13.0-106.0.dev"` — for the `eb362d426` conversion
  (`AnalysisServerDiagnosticsAction.java:27`).

---

## Verified facts (2026-08-17, `dart-lang/sdk` @ `144f5acb5db`)

Everything above rests on these checks. Re-run them if significant time has passed.

**§1 — The change is still needed**

* `handler_inlay_hint.dart`: `class InlayHintHandler extends LspMessageHandler<InlayHintParams,
  List<InlayHint>>` — still LSP-only.
* `handler_states.dart:105` — `InlayHintHandler.new,` inside `lspHandlerGenerators` (which begins at
  line 86). `sharedHandlerGenerators` begins at line 122 and contains
  `IncomingCallHierarchyHandler.new` (147) and `InlineValueHandler.new` (148).
* `pkg/analysis_server/test/lsp_over_legacy/` contains no `inlay_hint_test.dart`.
* No `dart-lang/sdk` issue requests this conversion (search: `inlay in:title`, `inlayHint in:title`,
  `"shared handler" LSP`).

**§2 — All dependencies are server-agnostic (the premise of "pure type change")**

* `handlers.dart:81` — `mixin HandlerHelperMixin<S extends AnalysisServer>`, spanning lines 81–339,
  contains `pathContext` (82), `extractDocumentVersion` (103), `fileHasBeenModified` (118),
  `pathOfDoc` (161), `requireResolvedUnit` (284).
* `mapping.dart:680` — `bool isDartDocument(lsp.TextDocumentIdentifier doc)`, a top-level function.
* `analysis_server.dart:516` — `lsp.LspClientConfiguration get lspClientConfiguration;` (abstract,
  on the base server); `legacy_analysis_server.dart:318` holds the concrete field, constructed at
  line 407.
* `handlers.dart:577` — `abstract class SharedMessageHandler<P, R> extends MessageHandler<P, R,
  AnalysisServer>`; it does **not** declare `requiresTrustedCaller`, which is abstract at
  `handlers.dart:403`. `LspMessageHandler` (343) defaults it to `true` at line 351.

**§3 — Test infrastructure**

* `abstract_lsp_over_legacy.dart:23` — `abstract class LspOverLegacyTest extends
  PubPackageAnalysisServerTest with LspRequestHelpersMixin, LspReverseRequestHelpersMixin,
  LspEditHelpersMixin, ClientCapabilitiesHelperMixin, LspVerifyEditHelpersMixin,
  LspNotificationsMixin, AnalyticsTestMixin`. `LspAnalysisServerTestMixin` is **not** among them.
* `test/lsp/request_helpers_mixin.dart:680` — `Future<List<InlayHint>> getInlayHints(Uri uri, Range
  range)`; it sends `Method.textDocument_inlayHint`, which `LspOverLegacyTest` routes through
  `lsp.handle`.
* `test/lsp/server_abstract.dart:1355` — `rangeOfWholeContent`, inside `LspAnalysisServerTestMixin`
  (which starts at line 865) → not reachable from `LspOverLegacyTest`.
* `test/lsp/request_helpers_mixin.dart:217` — `startOfDocRange` is a zero-length range at (0,0);
  it is reachable, but it is not a range covering content.
* `definition_test.dart` / `references_test.dart` (both added by `c6d728bfd33`, header year 2026)
  establish the file shape used in Step 3.
* `test_all.dart` import order: … `implementation_test.dart`, `references_test.dart` … → the new
  import and `main()` call belong between those two.
* License header in `pkg/analysis_server/test/`: 948 files use `authors. Please see`, 0 files use
  `authors.  Please see`.

**§4 — Behaviour relevant to the plugin (Part 3)**

* Inlay hints are **on by default when the client sends no configuration**:
  `LspClientInlayHintsConfiguration` (`lsp/client_configuration.dart:195`) resolves every sub-option
  with `boolean ?? true` when `userPreference` is `null` (lines 203–217). So an LSP-over-Legacy
  client gets hints without sending `workspace/configuration`.
* The handler ignores `params.range` — it reads only `params.textDocument` and calls
  `DartInlayHintComputer(pathContext, result, config).compute()` for the whole unit.
* Hint kinds emitted are `Type` and `Parameter` only (`computer_inlay_hint.dart`); closing labels
  are a separate custom notification, not inlay hints.

**§5 — Contribution process**

* `CONTRIBUTING.md`: CLA required; Gerrit is the main flow (`git cl upload`); GitHub PRs are
  accepted and converted to CLs by a copybara bot; `gclient`/`fetch` is required for a functional
  build environment.
* `docs/Commit-Message-Best-Practices.md`: component prefix, blank line before body, present tense,
  full GitHub issue URL, GitHub auto-close notation.
* `.agents/rules/add_license_header.md` and `.agents/skills/{read_gerrit_cl,find_release,…}` exist
  and apply to agents working in this repo.
* Reference commits for handler work are listed in https://github.com/dart-lang/sdk/issues/63878
  ("Add a skill for implementing a new LSP request/notification handler", open).
