# Contribution 1: Make `IGNORED` URLs failures more visible in the summary

**Contribution Number:** 1  
**Student:** Samiul Saimon  
**Issue:** [lycheeverse/lychee#997](https://github.com/lycheeverse/lychee/issues/997)  
**Status:** Phase II Complete (reproduced + planned)

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

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
