---
name: kanso-audit
description: Use when the user asks for a code review, audit, pre-PR check, quality sweep, or pattern analysis of a diff, branch, module, or codebase. Also use when the user asks to "check" or "look over" their code for issues.
argument-hint: "[scope: diff|branch|module-path|all]"
allowed-tools: Bash(git *) Bash(gh *) Bash(rg *) Bash(find *) Bash(wc *) SlashCommand
---

# kanso-audit

End-to-end code review: investigate against the kanso principles, hand back a hyper-summarised verdict in British English, gate once on the user, then auto-route refactor-shaped fixes to `/kanso-refactor` and report what changed. Behaviour-change-shaped fixes are flagged but **not** auto-applied — they sit in the verdict so the user can decide what to do with them.

Four phases:

- **Phase A — Investigation.** Read-only. Gather context and grade findings against the framework below. The skill's `allowed-tools` exclude Edit/Write so the investigation cannot modify files.
- **Phase B — Verdict.** A hyper-summarised digest in British English. Tier 1 + Tier 2 findings only, one line each, tagged `refactor` or `behaviour-change`. Skip the verdict block entirely on a clean diff.
- **Phase C — Gate.** A single short prompt: proceed, modify, or stop. Free-text feedback is allowed (e.g. "skip #2", "also remove the dead import in foo.ts"). Wait for the user.
- **Phase D — Apply + recap.** Hand every approved `refactor`-shaped fix to `/kanso-refactor audit-report`. Leave `behaviour-change` items for the user. End with a hyper-summarised recap in British English of what `kanso-refactor` actually shipped.

The first thing the skill emits must be a visible status line so the user knows it is running. Example: `Auditing working tree vs HEAD…`. Never start with a silent tool call.

## Tone

British English throughout — every summary, recap, and prompt the user reads. Examples: *behaviour*, *colour*, *catalogue*, *organise*, *recognise*, *whilst*, *full stop*. Use this in narrative prose only; never alter code, identifiers, or quoted strings.

## Resolve the scope

`$ARGUMENTS` is one of:

- `diff` → unstaged + staged working tree (default if empty)
- `branch` → commits on the current branch vs its upstream or `main`
- a path like `src/billing/` → everything under that path
- `all` → the whole repo (warn the user; this is noisy on anything bigger than a small service)

If `$ARGUMENTS` is empty, default to `diff`. If the diff is empty, fall back to the last commit and say so in the status line.

Gather context before reviewing:

- Diff: `!git diff HEAD` or `!git diff main...HEAD` depending on scope
- Changed files: `!git diff --name-only`
- Recent commits for voice calibration: `!git log -n 20 --oneline`
- Existing conventions: check for `AGENTS.md`, `CLAUDE.md`, `.cursor/rules/`, `CONTRIBUTING.md`, linter configs

Read enough of the surrounding codebase to calibrate repo conventions. A finding that contradicts existing repo style is noise, not signal.

## The review framework

Findings sort into four categories (Google's pillars) and three tiers (signal priority).

### Categories

**Correctness** — Will this cause incorrect behaviour? Logic errors, off-by-ones, race conditions, missing error handling, security holes, auth bypasses, injection vectors, concurrency problems. Also: *defensive programming theatre*, which looks safe but hides failures.

**Clarity** — Can another developer read this quickly? Over-long functions, dense nesting, single-character or zero-entropy names, comments explaining *what* instead of *why*, over-engineering for imagined future needs.

**Consistency** — Does it match the surrounding codebase? Naming (mixed `get`/`fetch`/`load`), error handling style, decomposition patterns, API conventions, data-type choices. Anything a linter could catch is *not* a human review finding — note the missing linter rule instead.

**Architecture** — Does this code belong here? Does it introduce new coupling? Does it break backward compatibility? Does this add a dependency that deserves scrutiny? Does it anticipate a problem that doesn't exist yet?

### Tiers

**Tier 1 — Blocker.** Would cause production failures or material harm if merged. Logic bugs, breaking API changes, security holes, race conditions, data loss risks.

**Tier 2 — Important.** Would cause maintainability or scale issues. Architectural violations, missing tests for critical paths, performance regressions, increased coupling, anti-dilution patterns that will metastasise.

**Tier 3 — Polish.** Subjective or stylistic. Surfaced internally for completeness but never shown in the verdict.

### The signal rule

Only Tier 1 and Tier 2 findings reach the verdict. Tier 3 findings are dropped before the user ever sees them. If everything is Tier 3, the verdict is "clean" and the gate is skipped.

## The adversarial lens

Think like an adversary. Construct inputs that break the code:

- Empty input, null, zero, negative, huge?
- Concurrent access?
- Malicious user? Upstream service down?
- Retry — is the operation idempotent?
- Called a million times?

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

## How to tag the shape

- **`refactor`** — the fix produces identical behaviour for every input the code already handles. Examples: delete dead code, inline a filler variable, remove a tautological comment, collapse a one-implementation factory, rename a local variable. These are the targets `/kanso-refactor` knows how to apply.
- **`behaviour-change`** — the fix alters what the code does under some input. Examples: fix an off-by-one, remove defensive theatre that was swallowing errors, patch an auth bypass, change an architectural boundary. When in doubt, tag as `behaviour-change`.

## Phase B — the verdict

After investigation, emit a single short block. No long-form findings, no per-finding markdown sections, no "what's good" filler. Hyper-summarised, British English, one line per item.

Format:

```
Audit — <scope>: <n> blocker, <m> important. <one-sentence headline>.

Refactor (will be auto-applied):
  1. <path>:<line> — <one-line summary>
  2. <path>:<line> — <one-line summary>

Behaviour change (left for you):
  3. <path>:<line> — <one-line summary>
  4. <path>:<line> — <one-line summary>
```

Rules:

- One sentence headline. Lead with the highest-severity finding, or "Clean diff" if there are no Tier 1 or 2 issues.
- One line per finding. Path, line, and a verb-led summary in British English. No rationale, no diff snippets, no category tags — that detail lives in the model's context for Phase D, not on the user's screen.
- Omit the `Refactor` group if there are none. Same for `Behaviour change`.
- If there are zero Tier 1 + Tier 2 findings: emit only the headline line ("Audit — diff: clean."), skip groups, skip the gate, stop.

## Phase C — the gate

Immediately after the verdict, emit exactly:

```
Proceed? (y / feedback / n)
  y         — apply the refactors via /kanso-refactor; flag behaviour changes for you
  <text>    — free-text modifications (e.g. "skip 2", "also remove the dead import in foo.ts", "split refactor 1 into two commits")
  n         — stop, leave the verdict in the transcript
```

Stop and wait. Do not proceed to Phase D until the user replies. If they reply with free-text, treat it as instructions: amend, drop, or extend the refactor list, then re-emit the verdict + gate so they can confirm. Loop until `y` or `n`.

## Phase D — apply + recap

On `y`:

1. Bundle every approved `refactor`-shaped fix into one invocation of `/kanso-refactor audit-report`. Pass the list as structured context so kanso-refactor can split it into commits per its own rules.
2. Wait for `/kanso-refactor` to finish.
3. Emit a recap. Hyper-summarised, British English, one line per refactor that actually shipped, plus a one-line behaviour-change reminder if any were flagged.

Recap format:

```
Done — <n> refactor(s) applied across <m> file(s). Build/typecheck status: <pass | not run by kanso-refactor>.

Applied:
  1. <path>:<line> — <one-line summary> (commit <sha>)
  2. <path>:<line> — <one-line summary> (commit <sha>)

Skipped or failed:
  3. <path>:<line> — <reason>

Behaviour changes still on you:
  4. <path>:<line> — <one-line summary>
```

Rules:

- Only list refactors `/kanso-refactor` actually shipped — do not pre-list approvals as if they succeeded.
- If `/kanso-refactor` skipped or rejected anything, surface it with the reason it gave.
- Behaviour changes are never silently applied. They appear in this recap only as a reminder of what the user still needs to handle.
- One blank line, then stop. No "let me know if…", no offers for further work.

## Framing

- Critique the code, not the author. Use "this code" not "you".
- Every Tier 1 or Tier 2 finding maps to one verdict line. No padding.
- Offer alternatives on architecture disagreements rather than declaring a verdict.
- Don't flag preference-only issues. A style preference that isn't in a style guide goes nowhere.
- Don't repeat findings. One entry per pattern per location.

## What this skill never does

- Auto-apply `behaviour-change` fixes. They surface in the verdict and the recap; the user owns them.
- Apply refactors directly. They go through `/kanso-refactor` so the behaviour-preserving discipline holds.
- Edit files in Phase A, B, or C.
- Run tests or build the project.
- Check out branches or pull remote changes.
- Flag things a linter already catches without also noting the missing linter rule.
- Surface Tier 3 findings to the user. They are dropped after grading.
- Start silently. Always emit the status line first.

## Failure modes to avoid

- Reviewing without reading the PR description, commit messages, or repo conventions. Context first, then findings.
- Verbose verdicts. The user sees one line per finding, no exceptions.
- Mislabelling a behaviour change as a refactor to slip it past the gate. When in doubt, tag `behaviour-change`.
- Bundling refactors and behaviour changes in a single commit once fixes are applied. `kanso-refactor`'s split-commit rule applies.
- Reporting refactors as "applied" before `/kanso-refactor` has actually shipped them. The recap reflects reality, not intent.
- Padding the verdict with "what's good" or summary prose. The headline carries that signal.
