---
name: kanso-audit
description: Use when the user asks for a code review, audit, pre-PR check, quality sweep, or pattern analysis of a diff, branch, module, or codebase. Also use when the user asks to "check" or "look over" their code for issues.
argument-hint: "[scope: diff|branch|module-path|all]"
allowed-tools: Bash(git *) Bash(gh *) Bash(rg *) Bash(find *) Bash(wc *)
---

# kanso-audit

Code review that reports findings, proposes concrete fixes, and — on approval — either hands cleanup off to `/kanso-refactor` or applies the fix in place. The principles from `kanso-principles` govern both the findings and the fixes.

Three phases:

- **Phase A — Investigation.** Read-only. Gather context and produce the findings report using the framework below. The skill's `allowed-tools` deliberately exclude Edit/Write so the investigation cannot modify files.
- **Phase B — Proposal.** Translate Tier 1 and Tier 2 findings into a numbered list. Tag each entry by *shape*: `refactor` (behaviour-preserving) or `behaviour-change` (correctness, security, architecture). Show the list in the approval block and wait.
- **Phase C — Apply.**
  - For `refactor`-shaped fixes → hand off to `/kanso-refactor audit-report`. That skill already enforces the behaviour-preserving rule, has the refactor taxonomy, and knows how to split work into commit-able groups. Don't reimplement it here.
  - For `behaviour-change` fixes → apply in place in the main session, one at a time, re-reading each file immediately before editing. These are not refactors; each is a real behaviour change and the user has approved it as such.

Run the report in the caller's transcript — do not suppress findings to a fork.

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

## Proposed fixes

### Fix for [1] <finding title>
**File:** `path/to/file.ts:142`
**Shape:** `refactor` | `behaviour-change`
**Change:** <one-sentence description of the edit>
**Rationale:** <which principle from kanso-principles this upholds>

<minimal diff or before/after snippet>

### Fix for [2] ...
```

One entry per Tier 1 and Tier 2 finding. Tier 3 fixes are only proposed when the user ran `all` scope or explicitly asked. If a finding has no safe mechanical fix (e.g. an architecture disagreement), say so and omit it from the proposal list rather than inventing one.

### How to tag the shape

- **`refactor`** — the fix produces identical behaviour for every input the code already handles. Examples: delete dead code, inline a filler variable, remove a tautological comment, collapse a one-implementation factory, rename a local variable. These are the targets listed in `kanso-refactor`.
- **`behaviour-change`** — the fix alters what the code does under some input. Examples: fix an off-by-one, remove defensive theatre that was swallowing errors, patch an auth bypass, change an architectural boundary. When in doubt, tag as `behaviour-change` — that forces an explicit ask rather than a silent refactor.

## The approval gate

After the report, emit exactly this block and stop:

```
Proposed fixes: <n> across <m> files.

  Refactor (behaviour-preserving):
    1. <file>:<line> — <one-line change>
    2. <file>:<line> — <one-line change>

  Behaviour changes (need explicit approval):
    3. <file>:<line> — <one-line change>
    4. <file>:<line> — <one-line change>

Apply fixes? (y/n/edit/pick)
  y       — hand refactors to /kanso-refactor audit-report, then apply behaviour changes
  refactor-only  — hand refactors off; skip behaviour changes
  behaviour-only — apply behaviour changes; leave refactors for later
  pick 1,3 — apply only the numbered subset (routed by shape)
  edit    — amend the proposal
  n       — stop, leave the report in the transcript
```

Routing rules on apply:

- Any selected `refactor`-shaped fix → invoke `/kanso-refactor audit-report` with those findings. Do not edit refactor targets directly from this skill.
- Any selected `behaviour-change` fix → apply in the main session, one at a time. Re-read the file immediately before editing.
- Do not mix the two in a single commit. `kanso-refactor`'s hard rule — refactors and behaviour changes travel in separate commits — applies here too.

If there are zero Tier 1 or Tier 2 findings, skip the proposal and approval blocks entirely — the report stands alone.

## Framing

- Critique the code, not the author. Use "this code" not "you".
- Every finding includes a recommendation. A finding without a path forward is an alarm, not a report.
- Offer alternatives on architecture disagreements rather than declaring a verdict.
- If a finding is for learning rather than blocking, mark it FYI.
- Don't flag preference-only issues as findings. A style preference that isn't in a style guide goes in "what's good" or nowhere.
- Don't repeat findings. One entry per pattern per location.

## What this skill never does

- Edit files before the user has approved the proposal block.
- Run tests, build the project, or execute the code being reviewed.
- Check out branches or pull remote changes.
- Flag things a linter already catches, without also noting the missing linter rule.
- Generate findings for padding. If there are three Tier 1 findings and nothing else, the report has three findings.
- Prompt for approval when there are no Tier 1 or Tier 2 fixes to propose.

## Failure modes to avoid

- Reviewing without reading the PR description, commit messages, or repo conventions. Context first, then findings.
- Flagging AI-sounding code that is fine. Not every verbose function is slop.
- Treating this as a style-guide compliance scan. The anti-dilution taxonomy is about harm, not taste.
- Producing a review so long the author skims it. Density matters more than coverage.
- Proposing fixes that introduce the same anti-dilution patterns the audit just flagged.
- Bundling unrelated fixes into one approval block so the user can't accept a subset. Group by shape, then by finding; let `pick` select.
- Editing a file without re-reading it first — state may have changed between the audit and approval.
- Applying a refactor-shaped fix directly instead of handing it to `/kanso-refactor`. That's duplication, and it bypasses the behaviour-preserving discipline that skill enforces.
- Mislabelling a behaviour change as a refactor to slip it past the gate. When in doubt, tag `behaviour-change`.
- Mixing refactors and behaviour changes in a single commit once fixes are applied.
