# Contribution 1: Make `IGNORED` URLs failures more visible in the summary

**Contribution Number:** 1  
**Student:** Samiul Saimon  
**Issue:** [lycheeverse/lychee#997](https://github.com/lycheeverse/lychee/issues/997)  
**Status:** Phase IV Complete — PR merged ✅

---

## Why I Chose This Issue

[lychee](https://github.com/lycheeverse/lychee) is a fast, async link checker written in Rust, and scans Markdown, HTML, websites, and emails to find broken URLs or dead links. This issue asks for ignored/unsupported URLs (e.g. malformed links like `hhttps://...`) to be listed explicitly in the run summary, instead of silently disappearing. Right now, they only show up as a mismatch between the "Total" count and the sum of the other categories, which can be difficult to notice in a repo with thousands of links. 

I chose it because it's a self-contained, well-scoped issue where the maintainer has already pointed to the relevant file to change with implementation guidance, so I can focus on learning the codebase and Rust output formatting rather than untangling a sprawling bug. The issue also aligns with my interests in software engineering and developer tools while giving me an opportunity to gain hands-on experience with Rust, a language I have limited experience contributing in. The maintainer ([@mre](https://github.com/lycheeverse/lychee/issues/997)) has already confirmed I can take it.

---

## Understanding the Issue

### Problem Description

When lychee checks links, some URLs are reported as `IGNORED` (internally an *unsupported* status - for example a malformed `hhttps://` URL that can't be turned into a request). These ignored links are counted in the `Total` but are not broken out anywhere in the printed summary. As a result, the only signal that something was ignored is that `Total` no longer equals the sum of the visible categories (Successful, Errors, etc.), which is easy to miss, and an opportunity to investigate the logs is lacking.

### Expected Behavior

The summary should explicitly list the ignored/unsupported links (similar to how Errors, Timeouts, Redirects, and Suggestions are already broken out per input), so a user can immediately see which URLs were ignored and investigate malformed links.

### Current Behavior

Ignored/unsupported links are tallied into the total count but never displayed individually, so they "vanish" from the summary and the user is left accounting for the differences by hand.

### Affected Components

- **Primary:** `lychee-bin/src/formatters/stats/markdown.rs` - the maintainer pointed here; it calls `write_stats_per_input(...)` for `error_map`, `timeout_map`, `redirect_map`, and `suggestion_map`, and a new block is needed for ignored links.
- **The other stat formatters** in `lychee-bin/src/formatters/stats/` (e.g. the compact/detailed text formatters) "where it makes sense."
- **Possible Additional Component: `ResponseStats`** (in `lychee-bin/src/formatters/stats/response.rs`) - Based on an initial review, it appears to track an unsupported counter but may not store the underlying ignored responses needed for formatter output. This will be investigated further during Phase II.

---

## Reproduction Process

### Environment Setup

lychee is a pure Rust project. [CONTRIBUTING.md](https://github.com/lycheeverse/lychee/blob/master/CONTRIBUTING.md) keeps setup minimal: install Rust via [rustup](https://rustup.rs/), then `cargo test` (for running tests) and `cargo clippy` (for linting code) should pass. The repo also ships a `.devcontainer/` (VS Code + Docker), but I did **not** use it because installing Rust directly is simpler and it is the project's documented path. Docker would have been a heavier/unnecessary dependency for building a CLI.

**Setup steps I followed (macOS):**

1. Installed the Rust toolchain (includes `cargo`, the build tool):
   ```sh
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
   source "$HOME/.cargo/env"
   ```
   The repo pins `channel = "stable"` in `rust-toolchain.toml`, so rustup automatically uses the stable toolchain.
2. Verified the toolchain: `rustc --version` → `rustc 1.96.0`, `cargo --version` → `cargo 1.96.0`.
3. Built the project from the repo root: `cargo build`. The first build compiles all dependencies and took several minutes; later builds are incremental and fast.
4. Confirmed a green baseline before changing anything by running the existing stats-formatter tests:
   ```sh
   cargo test --bin lychee formatters::stats
   ```
   → `test result: ok. 13 passed; 0 failed ...`.

**Challenges / notes:**
- After installing rustup, `cargo` isn't on the `PATH` in the current shell until you run `source "$HOME/.cargo/env"` (or open a new terminal). Forgetting this gives a `command not found: cargo` error.
- lychee excludes `example.com` and similar reserved domains by default, so my first reproduction file used `example.com` links that showed up as `EXCLUDED` instead of the status I wanted. I switched to a real host (`rust-lang.org`) plus deliberately malformed/unsupported links to get clean `IGNORED` results.
- The per-link live progress lines like `[IGNORED] ...` are printed to **stderr** during execution while the actual final summary **report** is printed to **stdout**. Redirecting stderr (`2>/dev/null`) (like a digital trash can) isolates the report, making the bug (the ignored URLs are missing from the summary) obvious.

### Steps to Reproduce

These steps reproduce the bug on a clean clone. They assume Rust is installed (see Environment Setup above).

1. Build lychee from the repo root:
   ```sh
   cargo build
   ```
2. Create a test Markdown file containing one valid link and two links that become `IGNORED`/`Unsupported` (a typo'd scheme and an unsupported URI scheme):
   ```sh
   # creates test folder
   mkdir -p /tmp/lychee-repro
   # creates test file - Reproduction for lychee issue 997
   cat > /tmp/lychee-repro/links.md << 'EOF'

   A working link: <https://www.rust-lang.org>

   A malformed link with a bad scheme (becomes IGNORED/Unsupported):
   [typo https](hhttps://www.rust-lang.org)

   An unsupported URI scheme (also IGNORED/Unsupported):
   [slack deep-link](slack://channel?team=T123)
   EOF
   ```
3. Run lychee with the Markdown formatter (the formatter the maintainer pointed to), isolating the report on stdout:
   ```sh
   ./target/debug/lychee --no-progress --format markdown /tmp/lychee-repro/links.md 2>/dev/null
   ```
4. **Observed result:** the summary table reports `⛔ Unsupported | 2`, and "Redirects" gets its own broken-out section, but there is **no section listing the two ignored URLs** (`hhttps://www.rust-lang.org` and `slack://channel?team=T123`). The count is present and the URLs have vanished from the report. The same gap appears in the compact (default) and detailed (`--format detailed`) formatters.

**Expected result:** an "Unsupported per input" (or "Ignored") section listing each ignored URL and its source, mirroring how Errors, Timeouts, Redirects, and Suggestions are already broken out.

I reproduced this consistently across multiple runs and across the markdown, compact, and detailed formatters.

### Reproduction Evidence

- **Branch in my fork:** https://github.com/samiuls25/lychee/tree/fix-issue-997
- **Observed output (markdown report, stderr hidden):**
  ```
  # Summary

  | Status         | Count |
  |----------------|-------|
  | 🔍 Total       | 3     |
  | 🔗 Unique      | 3     |
  | ✅ Successful  | 1     |
  | ⏳ Timeouts    | 0     |
  | 🔀 Redirected  | 1     |
  | 👻 Excluded    | 0     |
  | ❓ Unknown     | 0     |
  | 🚫 Errors      | 0     |
  | ⛔ Unsupported | 2     |

  ## Redirects per input

  ### Redirects in /tmp/lychee-repro/links.md

  * https://www.rust-lang.org/ --[301]--> https://rust-lang.org/
  ```
  The report claims 2 unsupported links but lists none of them.
- **My findings:** The root cause is on the *data* side, not just the formatting side. lychee correctly **counts** unsupported responses but never **stores** them, so by the time any formatter runs there is no data to list. Details in Solution Approach below.

---

## Solution Approach

### Analysis

The bug has a **data layer** and a **formatting layer**, and the root cause is in the data layer.

Tracing the flow in `lychee-bin/src/formatters/stats/response.rs`:

- **Counting works.** `ResponseStats::increment_status_counters` (`response.rs`) handles `Status::Unsupported(_) => self.unsupported += 1`, so the count is always correct. ("IGNORED" is just the display label for `Status::Unsupported`; see `lychee-lib/src/types/status.rs`, where `Status::Unsupported(_) => "IGNORED"`.)
- **Storing does not work.** `ResponseStats::add_response_status` routes each response into a per-input map (`timeout_map`, `error_map`, `excluded_map`, `success_map`) via an `if / else if` chain. An `Unsupported` response matches **none** of those branches and hits the final `else { return; }`, so the `ResponseBody` is dropped. There is also **no `unsupported_map` field** on the `ResponseStats` struct to hold it.
- **Formatting can't show what wasn't stored.** `markdown.rs` calls `write_stats_per_input(...)` for `error_map`, `timeout_map`, `redirect_map`, and `suggestion_map`. Even if I added an "Unsupported" block there, there would be no map to read from.

So the count and the URLs come from two different mechanisms. Only the counter path covers unsupported links. **Adding a formatter block alone would not work** because the data has to be captured first.

Secondary finding: `detailed.rs` has a latent typo: the "⛔ Unsupported" row prints `stats.errors` instead of `stats.unsupported`. It's small and directly adjacent to this work, so I plan to mention it in the same PR message (to confirm with the maintainer that fixing it is acceptable while keeping the original PR changes clean and addressing the main issue).

### Proposed Solution

1. Add an `unsupported_map: HashMap<InputSource, HashSet<ResponseBody>>` field to `ResponseStats`, mirroring `error_map`/`timeout_map`.
2. Populate it in `add_response_status` by adding a branch for unsupported responses (using the existing `status.is_unsupported()` helper in `lychee-lib`).
3. Print it in the formatters: add an "Unsupported" (labeled "Ignored") block in `markdown.rs`, and surface the per-input list in `compact.rs` / `detailed.rs` where it fits, mirroring how errors/timeouts are already shown.
4. Add/extend unit tests so the new map and rendered output are covered.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** When lychee can't turn a URL into a request (e.g. a malformed scheme like `hhttps://` or an unsupported scheme like `slack://`), it marks the response `Unsupported` and displays it as `IGNORED`. The summary report shows a *count* of these but never lists *which* URLs they were, so they effectively disappear. The only hint is that `Total` no longer equals the sum of the visible categories. The fix is to list each ignored URL per input, the way Errors, Timeouts, Redirects, and Suggestions already are.

**Match:** The codebase already does exactly this pattern for other statuses, so I can copy it rather than invent anything:
- In `response.rs`, `error_map`/`timeout_map`/`excluded_map`/`success_map` are `HashMap<InputSource, HashSet<ResponseBody>>` populated inside `add_response_status`. I'll add `unsupported_map` the same way.
- In `markdown.rs`, `write_stats_per_input(f, "Errors", &stats.error_map, ...)` renders a per-input section. I'll add a similar call for the unsupported map.
- In `compact.rs`/`detailed.rs`, error and timeout responses are iterated per source. I'll include unsupported entries alongside them where it makes the most sense.
- `lychee-lib`'s `Status::is_unsupported()` already exists, so the new branch in `add_response_status` has a ready-made predicate/check.

**Plan:** Step-by-step:
1. `response.rs`: add `unsupported_map: HashMap<InputSource, HashSet<ResponseBody>>` to `ResponseStats`, and add a branch in `add_response_status` (`else if status.is_unsupported()`) that inserts into it. Confirm whether it should populate always or only under `detailed_stats` (errors/timeouts populate always - I'll match that so the data is visible by default).
2. `markdown.rs`: add a `write_stats_per_input(f, "Ignored", &stats.unsupported_map, ...)` block reusing the existing `markdown_response` closure.
3. `compact.rs` / `detailed.rs`: include the unsupported map in the per-input output where it makes sense. Fix the `stats.errors` → `stats.unsupported` typo in `detailed.rs` (pending upon response in initial PR).
4. Update tests: extend the `get_dummy_stats()` fixture in `mod.rs` with an unsupported entry, and update the `test_render_summary` / `test_detailed_formatter` / `test_formatter` snapshot expectations. Add a `response.rs` unit test asserting an unsupported response lands in `unsupported_map`.

**Implement:** Work happens on branch [`fix-issue-997`](https://github.com/samiuls25/lychee/tree/fix-issue-997) in my fork (Phase III). Commit links will be added here as I push.

**Review:** Before opening the PR I will: run `cargo test` and `make lint` (`cargo clippy`) per CONTRIBUTING.md and ensure both are clean, keep the change minimal and consistent with the existing map/formatter pattern, double-check the maintainer's guidance on the issue, and review lychee's commit-message/PR conventions (recent history uses Conventional Commits, e.g. `fix:` / `feat:`) so my PR title matches.

**Evaluate:** Verify by (a) re-running the reproduction command above and confirming the two ignored URLs now appear in a dedicated section of the report; (b) `cargo test --bin lychee formatters::stats` passing with the updated snapshots; (c) `cargo clippy` clean; and (d) confirming the `Total` count still matches with the now-visible categories.

---

## Testing Strategy

The project tests these formatters with in-crate unit tests and exact-output "snapshot" assertions (e.g. `test_render_summary`), so I followed that same pattern rather than adding new integration-test files.

### Unit Tests

- [x] **Data layer — `response.rs::test_unsupported_stats`:** asserts that an `Unsupported` (IGNORED) response is both counted (`stats.unsupported == 1`) *and* stored in the new `unsupported_map`. This is the test that would have caught the original bug (responses counted but never stored).
- [x] **Markdown — `markdown.rs::test_render_summary_with_ignored`:** exact-snapshot assertion on the full rendered summary, confirming a new `## Ignored per input` section is grouped by source and lists the ignored URL with its `[IGNORED]` reason.
- [x] **JSON snapshot — `json.rs::test_json_formatter`:** updated the expected JSON to include the new serialized `unsupported_map` field (adding the field changes the serialized struct, so this snapshot had to move with it).
- [x] **Compact — `compact.rs::test_formatter_lists_ignored_only_input`:** exact-snapshot (color codes stripped) confirming an input whose *only* finding is an ignored link is still listed under its source.
- [x] **Detailed — `detailed.rs::test_detailed_formatter_lists_ignored_only_input`:** exact-snapshot confirming a dedicated `Ignored in <source>` section (not mislabeled under "Errors").
- [x] **Count fix — `detailed.rs::test_detailed_formatter`:** updated snapshot after correcting the `⛔ Unsupported` summary row to use `stats.unsupported` instead of `stats.errors`.
- [x] **Regression:** all 13 pre-existing stats-formatter tests still pass. Final total: 17 stats tests passing; full `cargo test --bin lychee` green (82 tests); `cargo clippy` clean.

### Integration Tests

- No new integration-test files were needed: the formatters are unit-tested in-crate (matching the project's existing approach), and end-to-end behavior is validated manually via the CLI (below).

### Manual Testing

Rebuilt the binary and re-ran the exact Phase II reproduction. **Before**, the markdown report showed `⛔ Unsupported | 2` with no list of which URLs. **After the fix**, the same command produces a new `## Ignored per input` section listing both ignored URLs and their reasons, so the count now reconciles with a visible list:

```
## Ignored per input

### Ignored in /tmp/lychee-repro/links.md

* [IGNORED] <hhttps://www.rust-lang.org> (at 6:1) | Unsupported: Failed to create HTTP request client: builder error for url (hhttps://www.rust-lang.org)
* [IGNORED] <slack://channel?team=T123> (at 9:1) | Unsupported: Failed to create HTTP request client: builder error for url (slack://channel?team=T123)
```

I also verified the compact (default) and detailed (`--format detailed`) formatters end-to-end: both now list ignored links per input, including for an input whose only finding is ignored links. After the count fix, the detailed formatter's `⛔ Unsupported` row correctly shows the number of ignored links (previously it printed the error count). `cargo clippy --bin lychee` is clean.

---

## Implementation Notes

### Week 3 Progress (Phase III)

Synced my branch with the latest `upstream/master` (rebased — no conflicts; confirmed none of the upstream commits touched the `formatters/stats/` files I'm changing), then implemented the fix in small, independently-tested commits. Completed so far:

1. **Data layer (commit 1):** added the `unsupported_map` field to `ResponseStats` and a branch in `add_response_status` so ignored/unsupported responses are stored (not just counted). This was the actual root cause — previously these responses fell through to `else { return; }` and were discarded.
2. **Markdown formatter (commit 2):** added an `## Ignored per input` section, reusing the existing `write_stats_per_input` helper and `markdown_response` closure.

Challenges faced and how I solved them:
- **`ErrorKind` equality gotcha:** my first `response.rs` test used `Status::Unsupported(ErrorKind::UnsupportedUriType("slack"))`, and the assertion failed even though the debug output looked identical. Cause: `ErrorKind`'s hand-written `PartialEq` has no arm for `UnsupportedUriType`, so it falls to `_ => false` — two identical values are never equal. Fixed by using `ErrorKind::InvalidUrlHost`, which *is* handled (`=> true`) and is a real cause of an `Unsupported` status.
- **JSON serialization coupling:** `ResponseStats` derives `Serialize`, so simply adding the new field changed the JSON formatter's output and broke `test_json_formatter`. The fix is correct and intended, so the snapshot update belongs in the same commit as the field.

Still to do (next session): compact + detailed formatters (commits 3–4), with full parity so an input whose only problem is ignored links still lists them.

### Week 4 Progress (Phase III)

Re-synced with `upstream/master` and finished the remaining formatters, keeping the same small-commit discipline:

3. **Compact formatter (commit 3):** chained `unsupported_map` into the per-input listing *and* the "Issues found in N inputs" count, so ignored links show under their source — including for inputs whose only finding is ignored links ("full parity").
4. **Detailed formatter (commit 4):** added a dedicated `Ignored in <source>` section. I gave it its own block (rather than chaining into the existing `Errors in <source>` loop) so ignored links aren't mislabeled as errors, and so an input with only ignored links still appears.
5. **Count fix (commit 5):** while in `detailed.rs` I found the `⛔ Unsupported` summary row printed `stats.errors` instead of `stats.unsupported`, so it showed the error count under the Unsupported label. Fixed it as its own `fix:` commit since it's directly on-theme for #997.

Decision I revisited: I had originally planned to leave the count typo for a follow-up. On reflection, since it's the same file *and* the same "make unsupported links accurate/visible" theme as #997, I included it — but as a **separate commit** so it stays cleanly attributable and easy to split out if the maintainer prefers. I'll surface this explicitly in the PR description.

All five commits are pushed to the branch; the full suite is green (17 stats tests, 82 bin tests) and clippy is clean. Ready to open the PR (Phase IV).

### Code Changes

- **Files modified (all in `lychee-bin/src/formatters/stats/`):** `response.rs` and `json.rs` (commit 1); `markdown.rs` (commit 2); `compact.rs` (commit 3); `detailed.rs` (commits 4 and 5).
- **Branch:** https://github.com/samiuls25/lychee/tree/fix-issue-997
- **Key commits:**
  - `feat(stats): collect ignored (unsupported) responses` — adds + populates `unsupported_map`, with a unit test and the JSON snapshot update.
  - `feat(stats): list ignored links in the Markdown summary` — adds the `## Ignored per input` section + an exact-snapshot test.
  - `feat(stats): list ignored links in the compact summary` — lists ignored links per input (full parity) + test.
  - `feat(stats): list ignored links in the detailed summary` — adds an `Ignored in <source>` section + test.
  - `fix(stats): use the unsupported count in the detailed summary` — corrects the `stats.errors` → `stats.unsupported` typo + snapshot updates.
- **Approach decisions:**
  - **One atomic commit per layer/formatter,** each leaving `cargo test` green, so the history is bisectable and easy to review.
  - **Local test fixtures** (instead of editing the shared `get_dummy_stats`) so each formatter commit is self-contained and doesn't ripple snapshot churn into the others — the same approach `junit.rs` already uses.
  - **Exact-snapshot assertions** to match the rest of the file's test style.
  - **Labeled the section "Ignored"** to match the user-facing `[IGNORED]` status text and the issue title.
  - **Compact "Issues found in N" count includes ignored links** so the header matches the number of detail blocks shown; ignored links don't affect the exit code. Note in the PR as a reversible choice.
  - **Included the `detailed.rs` count typo fix** as a separate `fix:` commit (originally planned as a follow-up) since it's on-theme for #997; kept separable in case the maintainer wants it split.

---

## Pull Request

**PR Link:** https://github.com/lycheeverse/lychee/pull/2243

**PR Description:** Resolves #997 by making ignored (`IGNORED`/unsupported) links visible in lychee's run summaries. lychee previously *counted* unsupported links (malformed or unsupported-scheme URLs) but never *stored* them, so they were never listed — the only hint was that `Total` didn't match the sum of the visible categories. The PR adds an `unsupported_map` to `ResponseStats` and lists ignored links per input in the markdown, compact, and detailed formatters (and exposes the new field in the JSON output), mirroring how Errors/Timeouts/Redirects are already broken out. It also includes a small related fix (the detailed formatter's `Unsupported` count printed the error count due to a copy-paste typo) as a separate commit. Split into 5 atomic commits, each green on `cargo test`; full suite (82 bin tests) and `cargo clippy`/`cargo fmt` all pass.

**Submission checklist (verified before opening):** branch targets `lycheeverse:master` ← `samiuls25:fix-issue-997`; no merge conflicts (`git merge-tree` clean against latest `master`); exact CI commands pass locally (`cargo fmt --all --check`, `cargo clippy --all-targets --all-features -- -D warnings`); no DCO/CLA required; "Allow edits by maintainers" enabled.

**Maintainer Feedback:**
- _2026-06-24:_ PR opened and awaiting review. The maintainer (@mre) had already confirmed I could take the issue.
- _2026-06-29:_ @mre reviewed, **approved, and merged** the PR (merge commit `2761062`) into `lycheeverse:master`; all 7 CI checks passed, and issue #997 was auto-closed. He left one non-blocking observation: the codebase mixes the related-but-distinct terms "unsupported" (the internal status) and "ignored" (the user-facing label), which could be aligned in the future. This was a deliberate choice in my change - user-facing "Ignored" to match the `[IGNORED]` label, internal `unsupported_map`/`is_unsupported()` naming. Since the note was non-blocking and the PR was already merged, I acknowledged the merge with an emoji reaction and am leaving the contribution here as completed.

**Status:** ✅ Merged (shipping in lychee v0.25.0)

---

## Learnings & Reflections

### Technical Skills Gained

- **Reading an unfamiliar Rust codebase to find a root cause.** I traced the bug from the formatter output back through `ResponseStats::add_response_status` and learned the difference between where lychee *counts* a status (`increment_status_counters`) and where it *stores* the response body for later display. The real fix was in the data layer first, and then the formatter the issue pointed at.
- **Rust specifics:** working with `HashMap<InputSource, HashSet<ResponseBody>>`, the `Display` trait and `fmt::Formatter`, iterator chaining (`.chain(...)`) to merge maps, struct-update syntax (`..Default::default()`), and how `#[derive(Serialize)]` means adding a field changes JSON output.
- **The project's testing style:** in-crate unit tests with exact-output "snapshot" assertions, and how to generate a snapshot by running the test once and freezing the real output.
- **Professional Git/GitHub workflow:** fork + `upstream`/`origin` remotes, `git fetch`/`rebase` to stay current, atomic Conventional Commits, `git merge-tree` to detect conflicts before opening a PR, and matching CI locally (`cargo fmt --all --check`, `cargo clippy --all-targets --all-features -- -D warnings`).

### Challenges Overcome

- **The issue's guidance was outdated.** The maintainer's 2023 comment said to "add a block to iterate over the ignored links," but it assumed the data was already collected — it wasn't (it fell through an `else { return; }`). Realizing the listed fix couldn't work without first capturing the data was the key insight.
- **A subtle `PartialEq` bug in a test.** My first unit test compared two `Status::Unsupported(ErrorKind::UnsupportedUriType(...))` values that looked identical but failed `assert_eq!`. The cause was that `ErrorKind`'s hand-written `PartialEq` has no arm for that variant, so it falls through to `_ => false`. I fixed it by using `ErrorKind::InvalidUrlHost`, which is handled and is a real cause of an `Unsupported` status. This taught me not to assume derived equality.
- **Keeping commits atomic without breaking other tests.** Editing the shared `get_dummy_stats()` fixture would have rippled snapshot changes across several formatter tests at once. I used local per-test fixtures instead (the same pattern `junit.rs` uses), so each commit stayed self-contained and green.
- **stdout vs stderr.** Early on it looked like ignored links *were* shown, until I realized those `[IGNORED]` lines are live progress on stderr, while the actual report (the thing the issue is about) goes to stdout. Isolating it with `2>/dev/null` made the bug more obvious.

### What I'd Do Differently Next Time

- **Decide on adjacent fixes earlier.** I went back and forth on whether to fix the `detailed.rs` count typo. Settling a clear rule up front ("fix on-theme adjacent bugs, but in their own commit") would have avoided the churn.
- **Confirm a fixture's equality semantics before writing assertions**, rather than discovering the `PartialEq` quirk through a failing test.
- Overall the small-commit, test-as-you-go approach worked well, and I'd keep it.

---

## Resources Used

- [lychee issue #997](https://github.com/lycheeverse/lychee/issues/997) - the issue itself, including the maintainer's implementation pointer and the original reporter's minimal reproduction repo.
- [lychee CONTRIBUTING.md](https://github.com/lycheeverse/lychee/blob/master/CONTRIBUTING.md) - setup and the `cargo test` / `cargo clippy` expectations.
- [lychee CI workflow](https://github.com/lycheeverse/lychee/blob/master/.github/workflows/ci.yml) - to find the exact checks CI enforces (`cargo fmt --all --check`, `cargo clippy --all-targets --all-features -- -D warnings`) and match them locally.
- [rustup.rs](https://rustup.rs/) - installing the Rust toolchain.
- [Rust `std::fmt` docs](https://doc.rust-lang.org/std/fmt/) - the `Display` trait and `Formatter`, for the formatter changes.
- [Conventional Commits](https://www.conventionalcommits.org/) - commit message format (the project uses `release-plz`, which relies on it).
- Existing code in the same area as my own best reference - `markdown.rs`/`compact.rs`/`detailed.rs`/`junit.rs` showed the patterns to mirror for both the formatter blocks and the test fixtures.
