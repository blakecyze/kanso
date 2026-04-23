---
name: kanso-audit
description: Use when the user asks for a code review, audit, pre-PR check, quality sweep, or pattern analysis of a diff, branch, module, or codebase. Also use when the user asks to "check" or "look over" their code for issues.
argument-hint: "[scope: diff|branch|module-path|all]"
context: fork
agent: Explore
allowed-tools: Bash(git *) Bash(gh *) Bash(rg *) Bash(find *) Bash(wc *)
---

# kanso-audit

Read-only code review. Produces a structured findings report. Makes no edits and suggests no file modifications beyond the report itself.

Runs in a forked subagent so findings don't pollute the working session. The principles from `kanso-principles` apply throughout.

## Resolve the scope

`$ARGUMENTS` is one of:

- `diff` → the unstaged + staged working tree (default if empty)
- `branch` → commits on the current branch vs its upstream or `main`
- a path like `src/billing/` → everything under that path
- `all` → the whole repo (warn the user this will be noisy on anything larger than a small service)

If `$ARGUMENTS` is empty, default to `diff`. If the diff is empty, fall back to the last commit and tell the user.

Gather context before reviewing:

- Diff: `!git diff HEAD` or `!git diff main...HEAD` depending on scope
- Changed files: `!git diff --name-only`
- Recent commits for voice calibration: `!git log -n 20 --oneline`
- Existing conventions: check for `AGENTS.md`, `CLAUDE.md`, `.cursor/rules/`, `CONTRIBUTING.md`, linter configs

Read enough of the surrounding codebase to calibrate repo conventions. A finding that contradicts existing repo style is noise, not signal.

## The review framework

Findings sort into four categories (Google's pillars) and three tiers (signal priority).

### Categories

**Correctness** — Will this cause incorrect behaviour? Logic errors, off-by-ones, race conditions, missing error handling, security holes, auth bypasses, injection vectors, concurrency problems. Also: the anti-dilution pattern *defensive programming theatre*, which looks safe but hides failures.

**Clarity** — Can another developer read this quickly? Over-long functions, dense nesting, single-character or zero-entropy names, comments explaining *what* instead of *why*, over-engineering for imagined future needs.

**Consistency** — Does it match the surrounding codebase? Naming (mixed `get`/`fetch`/`load`), error handling style, decomposition patterns, API conventions, data-type choices. Anything a linter could catch is *not* a human review finding — note the missing linter rule instead.

**Architecture** — Does this code belong here? Does it introduce new coupling? Does it break backward compatibility? Does this add a dependency that deserves scrutiny? Does it anticipate a problem that doesn't exist yet? The senior progression: "does it work" → "is it good" → "will it survive".

### Tiers

**Tier 1 — Blocker.** Would cause production failures or material harm if merged. Logic bugs, breaking API changes, security holes, race conditions, data loss risks.

**Tier 2 — Important.** Would cause maintainability or scale issues. Architectural violations, missing tests for critical paths, performance regressions, increased coupling, anti-dilution patterns that will metastasise.

**Tier 3 — Polish.** Subjective or stylistic. Minor naming suggestions, small clarity improvements, nits.

### The signal rule

If Tier 1 + Tier 2 findings are fewer than 60% of total findings, the audit is generating noise. Cut Tier 3 findings aggressively until the ratio is above 60%, or omit the Tier 3 section entirely on a clean diff.

## The adversarial lens

Google's rule: think like an adversary, but be polite about it. Explicitly try to construct inputs that break the code:

- What happens with empty input, null, zero, negative, huge?
- What happens under concurrent access?
- What happens if the user is malicious? If the upstream service is down?
- What happens on retry? Is the operation idempotent?
- What happens if this is called a million times?

Surface the adversarial cases that *aren't* handled. Don't invent ones that don't matter.

## Anti-dilution patterns to flag specifically

From `kanso-principles`. Call these out by name so they're searchable:

- **Tautological comment** — comment restates the code
- **Step-marker comment** — `// Step 1:`, `// Now do X`
- **Defensive programming theatre** — nested try/catch returning None/null on failure
- **Filler variable** — `result = expr; return result`
- **Zero-entropy name** — `userDataProcessingResult`, `helperFunction`
- **Premature abstraction** — factory for one implementation
- **Fake test coverage** — tests assert on mocks, not behaviour
- **Structural erosion** — god function absorbing new requirements
- **Phantom bug** — guard against impossible state
- **Vanilla reimplementation** — hand-rolled version of a stdlib function

## The report format

Emit a single markdown document. No preamble, no "Let me know if you'd like me to…", no apology. Structure:

```markdown
# Audit: <short scope description>

**Scope:** <what was reviewed>
**Files:** <count>
**Findings:** <tier 1 count> blocker, <tier 2 count> important, <tier 3 count> polish

## Summary

<Two or three sentences. Lead with the highest-severity finding. If the diff is clean, say so.>

## Blocker findings

### [1] <short title>
**Location:** `path/to/file.ts:142`
**Category:** Correctness
**Pattern:** <anti-dilution pattern name if applicable>

<One paragraph: what is wrong, why it matters, evidence.>

**Recommendation:** <Specific fix. Not "improve this" but "extract the DB call to a repository method and remove the try/catch".>

### [2] ...

## Important findings

(same structure)

## Polish findings

(same structure, but omit entirely if the signal ratio would drop below 60%)

## What's good

<One or two sentences. Not flattery. Note anything genuinely well-handled that a future modifier should preserve.>
```

## Framing

- Critique the code, not the author. Use "this code" not "you".
- Every finding includes a recommendation. A finding without a path forward is an alarm, not a report.
- Offer alternatives on architecture disagreements rather than declaring a verdict.
- If a finding is for learning rather than blocking, mark it FYI.
- Don't flag preference-only issues as findings. A style preference that isn't in a style guide goes in "what's good" or nowhere.
- Don't repeat findings. One entry per pattern per location.

## What this skill never does

- Edit files. The audit is read-only.
- Run tests, build the project, or execute the code being reviewed.
- Check out branches or pull remote changes.
- Flag things a linter already catches, without also noting the missing linter rule.
- Generate findings for padding. If there are three Tier 1 findings and nothing else, the report has three findings.

## Failure modes to avoid

- Reviewing without reading the PR description, commit messages, or repo conventions. Context first, then findings.
- Flagging AI-sounding code that is fine. Not every verbose function is slop.
- Treating this as a style-guide compliance scan. The anti-dilution taxonomy is about harm, not taste.
- Producing a review so long the author skims it. Density matters more than coverage.
