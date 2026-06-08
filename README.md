# Contribution 1: Make `IGNORED` URLs failures more visible in the summary

**Contribution Number:** 1  
**Student:** Samiul Saimon  
**Issue:** [lycheeverse/lychee#997](https://github.com/lycheeverse/lychee/issues/997)  
**Status:** Phase I Complete

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

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

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
