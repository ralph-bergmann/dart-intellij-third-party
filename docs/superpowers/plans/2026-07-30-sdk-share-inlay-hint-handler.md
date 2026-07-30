# Dart SDK task: Make `textDocument/inlayHint` a shared handler (LSP-over-Legacy)

> **Audience:** an AI coding agent (or human) working in a `dart-lang/sdk` checkout
> (e.g. `/Users/ralph.bergmann/development/projects/privat/sdk`). This document is
> self-contained — no other context is required.

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

The inlay hint handler has the same property: its dependencies (`requireResolvedUnit`,
`extractDocumentVersion`, `fileHasBeenModified` — all on the generic
`HandlerHelperMixin<S extends AnalysisServer>` — and the abstract `lspClientConfiguration` getter
on the base `AnalysisServer`) are already server-agnostic. The conversion is a pure type change.

Motivating downstream issue: https://github.com/flutter/dart-intellij-third-party/issues/159
("Add Inlay Hints to Dart code: variable type, parameter name, and more.").

## Step 0: File the GitHub issue

File a new issue on https://github.com/dart-lang/sdk (area-analysis-server). Suggested text:

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

Record the issue number — the commit message references it (`Fixes #NNNNN`).

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

(`requiresTrustedCaller => false` is what makes the handler callable by untrusted DTD clients as
well; the converted definition/references/documentHighlights/codeLens handlers all return `false`.
Keep the rest of the file unchanged.)

## Step 2: Move the registration

File: `pkg/analysis_server/lib/src/lsp/handlers/handler_states.dart`

* Remove `InlayHintHandler.new,` from `InitializedLspStateMessageHandler.lspHandlerGenerators`.
* Add `InlayHintHandler.new,` to `InitializedStateMessageHandler.sharedHandlerGenerators`,
  keeping the list's alphabetical order (between `IncomingCallHierarchyHandler.new` and
  `InlineValueHandler.new`).

## Step 3: Add an LSP-over-Legacy test

New file: `pkg/analysis_server/test/lsp_over_legacy/inlay_hint_test.dart`

Model it on the existing `document_highlights_test.dart` in the same directory. The shared test
helpers already provide `getInlayHints(Uri uri, Range range)` (in
`test/lsp/request_helpers_mixin.dart`) and `rangeOfWholeContent(String content)`:

```dart
// Copyright (c) 2026, the Dart project authors. Please see the AUTHORS file
// for details. All rights reserved. Use of this source code is governed by a
// BSD-style license that can be found in the LICENSE file.

import 'package:analysis_server/lsp_protocol/protocol.dart';
import 'package:test/test.dart';
import 'package:test_reflective_loader/test_reflective_loader.dart';

import 'abstract_lsp_over_legacy.dart';

void main() {
  defineReflectiveSuite(() {
    defineReflectiveTests(InlayHintTest);
  });
}

@reflectiveTest
class InlayHintTest extends LspOverLegacyTest {
  Future<void> test_inlayHints() async {
    var content = '''
var a = '';
''';
    newFile(testFilePath, content);
    var results = await getInlayHints(
      testFileUri,
      rangeOfWholeContent(content),
    );
    // Expect the Type hint for the inferred type of `a` (`String`).
    expect(results, hasLength(1));
    expect(results.single.kind, InlayHintKind.Type);
  }
}
```

Adjust imports/expectations to whatever the analyzer actually reports (run the test — see Step 5);
the shape above mirrors the converted handlers' tests. If `rangeOfWholeContent` is not visible from
`LspOverLegacyTest`, construct the range explicitly:
`Range(start: Position(line: 0, character: 0), end: Position(line: 1, character: 0))`.

## Step 4: Register the test

File: `pkg/analysis_server/test/lsp_over_legacy/test_all.dart`

Add (alphabetically):

```dart
import 'inlay_hint_test.dart' as inlay_hint;
```

and inside `main()`:

```dart
inlay_hint.main();
```

## Step 5: Verify

```bash
cd pkg/analysis_server
dart test test/lsp_over_legacy/inlay_hint_test.dart
dart test test/lsp_over_legacy/   # full LSP-over-Legacy suite
dart analyze .
```

All tests must pass; `dart analyze` must be clean.

## Step 6: Submit

Follow `CONTRIBUTING.md` in the SDK root (CLA, Gerrit via `git cl upload`). Suggested commit
message, mirroring `c6d728bfd33`:

```
[analysis_server] Make textDocument/inlayHint a shared handler

This allows IntelliJ to request inlay hints via LSP-over-Legacy while
being migrated to LSP.

InlayHintHandler only uses base-server facilities, so this is a change
of types plus a test, with no implementation changes.

Fixes https://github.com/dart-lang/sdk/issues/NNNNN
```

(Replace `NNNNN` with the issue number from Step 0.)

## Step 7: Report back (needed by the IntelliJ plugin)

After the CL lands, determine the **first dev version tag that contains the commit**:

```bash
git fetch origin
git tag --contains <landed-commit-sha> | sort -V | head -1
```

The result (format like `3.14.0-65.0.dev`) is required by the IntelliJ plugin as
`MIN_LSP_INLAY_HINTS_SDK_VERSION` — see Part 3 of
`docs/superpowers/plans/2026-07-30-dart-inlay-hints.md` in the
`dart-intellij-third-party` repository.
