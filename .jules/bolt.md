## 2024-05-18 - Case-insensitive filtering in Dart tight loops
**Learning:** Using `String.toLowerCase()` inside a tight `where` loop in Dart allocates a new String for every item evaluated. In lists of strings like search results or tag lists, this creates massive GC pressure.
**Action:** Use a precompiled `RegExp` with `caseSensitive: false` before the loop, and use `searchRegex.hasMatch(item)` inside the loop instead.

## 2026-08-16 - REJECTED, do not resubmit: RegExp-precompile on `WpSearchableListPage._filtered`
**Rejected 11 times** (#39, #40, #44, #48, #50, #53, #55, #58, #61, #63, #64) — this is the single most
resubmitted PR in this repo's history. The 2024-05-18 entry above is true in general but does **not**
apply to this specific call site, and applying it there anyway is exactly what caused the loop:
`WpSearchableListPage._filtered` runs over Snippets/Replacements lists that are typically a few dozen
items — the maintainer has stated repeatedly (#58, #61) that the claimed win is unbenchmarked and the
regex still gets recompiled once per keystroke either way, so there's no measured improvement to point
to. **Do not propose this again without an actual before/after benchmark number in the PR body** (e.g.
`dart run tool/bench_search_filter.dart` output, or equivalent instrumented timing) — a generic
"reduces GC pressure" claim is not sufficient and will be closed on sight.
**Meta-lesson:** before opening a perf PR that touches search/filter loops, run
`gh pr list --state closed --search "<the widget/file name>"` and read the closing comments — merged
`git log` on `dev` only shows *accepted* changes; it will never show you the 10 times this exact idea
was already rejected, because rejected PRs never touch `dev`.

## 2024-05-18 - Case-insensitive filtering in Dart tight loops
**Learning:** Using `String.toLowerCase()` inside a tight `where` loop in Dart allocates a new String for every item evaluated. In lists of strings like search results or tag lists, this creates massive GC pressure.
**Action:** Use a precompiled `RegExp` with `caseSensitive: false` before the loop, and use `searchRegex.hasMatch(item)` inside the loop instead. Note: Do not apply to `WpSearchableListPage._filtered`.

## 2026-10-24 - Contextual optimization of tight loops
**Learning:** While `toLowerCase()` inside a tight loop causes massive GC pressure, replacing it with a precompiled `RegExp(..., caseSensitive: false)` is only worth the complexity if the collection is sufficiently large (e.g., the ~10k candidates in `vocabulary_import_review_page.dart` rather than a few dozen items like in `WpSearchableListPage._filtered`). Prematurely applying this pattern to small loops lacks measurable impact and adds unnecessary complexity.
**Action:** When finding a tight loop with string case manipulation, explicitly check the expected data set size (e.g., via PRD comments or variable bounds) before applying the RegExp optimization. Only optimize where the list scale justifies the GC pressure relief.

## 2024-05-18 - Case-insensitive string matching tests and imports
**Learning:** In Dart/Flutter projects, relative imports spanning from `test/` into `lib/` (e.g., `import '../../../lib/...'`) explicitly violate lint rules and cause CI failures. Never modify a test file to use relative pathing to `lib/` to bypass a local package resolution error.
**Action:** Revert test file path modifications back to `package:...` imports if local test execution requires a temporary workaround, so that the committed PR retains clean standard imports.

## 2026-11-20 - RegExp case folding limits
**Learning:** `RegExp(..., caseSensitive: false)` in Dart does not perfectly mirror `String.toLowerCase()` behavior for certain Unicode characters, such as the Turkish dotted uppercase 'İ'. When optimizing tight loops from `toLowerCase()` to a precompiled `RegExp`, ensuring Unicode correctness for characters like 'İ' may fail tests that verify the exact case-folding behavior.
**Action:** When applying the RegExp optimization for performance, if the application has explicit tests for locale-specific casing behavior like the Turkish 'İ' (e.g. `test/services/side_panel/side_panel_row_filter_test.dart`), either avoid the optimization, or be aware it might break specific locale correctness.
