---
name: kanso-task
description: Use when the user wants a task done well rather than fast. Rewrites the user's rough request into a sharp prompt using kanso-prompting rules, asks targeted questions only when load-bearing detail is missing, then executes the rewritten task inline with kanso-principles loaded so any code produced follows the anti-dilution rules.
argument-hint: "[the thing you want done, in plain language]"
disable-model-invocation: true
allowed-tools: Bash(git *) Bash(rg *) Bash(find *) Bash(wc *)
---

# kanso-task

The user types `/kanso-task <rough request>`. This skill turns that rough request into a sharp prompt, then executes the task in the same chat with both `kanso-prompting` and `kanso-principles` standing as context. The output should be measurably better than what the user would have got from typing the same words into the chat directly.

Three phases:

- **Phase A — Clarify (only if needed).** Inspect the request. If load-bearing detail is missing (target, scope, constraints, definition of done), ask up to three targeted questions. If the request is already specific enough, skip this phase entirely.
- **Phase B — Rewrite.** Apply the `kanso-prompting` rules to produce a sharp version of the prompt. Show it to the user. Wait for approval.
- **Phase C — Execute.** Run the rewritten prompt inline in the current chat. Any code produced during execution follows `kanso-principles`.

## Always run inline

This skill runs in the calling chat. Never dispatch the rewrite, the approval gate, or the execution via the Agent or Task tool. Never spawn a subagent. The user has to be able to see the rewritten prompt before it runs, intervene during execution, and inspect the result without switching windows. A subagent dispatch defeats the entire workflow.

## Phase A — Clarify

Before rewriting, check the request for load-bearing detail. The four things that materially change the output:

1. **Target** — what file, module, branch, or scope is in play. `improve the auth flow` is missing this; `improve src/auth/session.ts` is not.
2. **Scope of change** — is this a behaviour change, a refactor, a new feature, an investigation, a doc? Different shapes call for different prompts.
3. **Constraints** — performance budget, backward compatibility, framework version, deadline, things-not-to-touch.
4. **Definition of done** — what does success look like? Tests passing, a specific output format, a working demo, a merged PR?

If the request already supplies enough of these to write a sharp prompt, skip Phase A. If one or two are missing, ask only for those. Maximum three questions, each one short and answerable in a sentence.

Never ask cosmetic questions. Never ask "would you like me to also…" — that's scope creep dressed as helpfulness. The clarifying phase exists to remove ambiguity, not to negotiate scope.

If after one round of clarification the request is still unworkable, surface that plainly rather than rewriting a vague prompt into a vague-but-prettier prompt.

## Phase B — Rewrite

Apply `kanso-prompting` to produce the sharp version. The rewrite should reliably:

- **Lead with the verb and the artefact.** `Audit`, `Refactor`, `Write`, `Implement`, `Explain`. Name what the model should produce.
- **State the target precisely.** File paths, function names, line ranges, branches. Not "the auth module" but `src/auth/session.ts`.
- **Carry the *why*.** A one-sentence reason. The model generalises better with motivation than without.
- **State constraints explicitly.** Frameworks, dependencies, things-not-to-touch, output shape.
- **State the stopping condition.** When is the model done? `Stop after the report and do not edit files` or `Apply the fix and run the existing tests`.
- **Drop the noise.** No `CRITICAL`, no ALL CAPS, no "please carefully think step by step". Modern Claude doesn't need it and it now degrades output.
- **Match the user's voice.** If the user is terse, the rewrite is terse. If the user is conversational, the rewrite stays conversational. Don't impose a house style.

The rewritten prompt earns its length the same way code does. Don't pad. A two-line rewrite is fine if two lines is enough.

## The approval gate

Always end Phase B with this block and stop:

```
Original:
  <one-line echo of what the user typed>

Rewritten:
  <the sharp version, as it would be sent>

Proceed?
  y          — execute the rewritten prompt inline, with kanso-principles loaded
  edit       — amend the rewritten prompt
  send-original — execute the original instead (rare; only when the rewrite drifted)
  n          — stop, leave the rewrite in the transcript
```

If the user approves, move to Phase C. If they pick `edit`, take the amended prompt and re-emit the gate. If they pick `send-original`, use the original.

## Phase C — Execute

Run the prompt in the current chat. During execution:

- `kanso-principles` is standing context. Any code written, modified, or reviewed follows the anti-dilution taxonomy. Deletion over addition, no defensive theatre, no filler variables, no tautological comments. The principles override default verbosity.
- `kanso-prompting` is standing context for any *meta* prompts produced during execution (e.g. asking a subagent for research — though this skill itself does not dispatch subagents).
- Existing kanso skills apply where they fit. If the rewritten task is a refactor, `/kanso-refactor` rules govern. If it produces commits, `/kanso-commit` rules govern. If it ends with a PR, `/kanso-pr` rules govern. Don't reimplement those — invoke them or let them auto-load.
- Maintain a running `implementation-notes.md` file (see below) for any non-trivial execution.
- The model executes the task the same way it would if the user had typed the rewritten prompt directly — with full tool access, in the current chat, with the user able to interrupt.

When execution finishes, report what changed. One short summary block. No marketing. If a notes file was written, point at it.

## Implementation notes during execution

No matter how sharp the rewritten prompt is, ambiguities and unknown-unknowns surface during execution. The model has to make small judgment calls — a naming choice, a library version, a structural decision, a deviation from what the spec implied. Stopping to ask about each one breaks momentum; making them silently leaves the user blind.

During Phase C, maintain a running `implementation-notes.md` file in the working directory. This is the model's sanctioned way to make a call without interrupting, while keeping the user fully in the loop after the fact. The file is reviewable at the end and converts cleanly into a PR description or a commit body.

Append (don't overwrite) a timestamped section per `/kanso-task` run:

```markdown
## 2026-05-25 14:32 — <one-line task description>

### Decisions made outside the spec
- <choice, with the reasoning in one sentence>

### Things changed from what the prompt implied
- <deviation and why it was needed>

### Tradeoffs taken
- <what was given up, what was gained>

### Anything else you should know
- <surprises, follow-ups worth doing, things you may want to revisit>
```

Rules:

- **Append, never overwrite.** If the file exists, add a new dated section at the bottom.
- **Skip for trivial executions.** If the task was a single one-line edit, a question that didn't touch the codebase, or anything where no judgment call was made, don't write the file. A notes file with empty sections is noise.
- **One sentence per entry.** The notes are a log, not a reflection. Long entries belong in the commit body.
- **Omit empty sections.** If no tradeoffs were taken, the heading goes too.
- **Reference the notes in the final summary.** `Notes: implementation-notes.md (3 decisions, 1 tradeoff)` so the user sees there's something to read.

The file lives alongside the user's repo; it's their artefact, not a hidden one. They can keep it, paste it into a PR, delete it after review, or `.gitignore` it. The skill doesn't manage its lifecycle beyond writing to it.

## What this skill never does

- Run via the Agent or Task tool, or dispatch any phase to a subagent or parallel runner.
- Execute the rewritten prompt without showing it to the user first.
- Pad the rewrite to look more rigorous. Length is not signal.
- Ask more than three clarifying questions per round.
- Re-introduce the noise patterns `kanso-prompting` warns against (`CRITICAL`, ALL CAPS, anti-laziness scaffolds).
- Silently change the user's intent under cover of "sharpening". The rewrite is faithful — only the wording improves.
- Edit `kanso-principles` or `kanso-prompting` during execution. Those are reference docs, not work targets.
- Skip the approval gate. The user always sees the prompt before it runs.

## Failure modes to avoid

- Rewriting a vague prompt into a longer vague prompt. If the source is empty of intent, ask — don't pad.
- Asking clarifying questions the user already answered in the original request.
- Re-stating the original request in the rewrite. The rewrite is a different artefact, not a paraphrase.
- Drifting from the user's actual ask by adding plausible-sounding scope ("I'll also write tests for it"). Stay faithful; if you think more is needed, surface it as a follow-up after execution, not as a silent expansion.
- Treating "execute inline" as licence to skip the principles. `kanso-principles` is the whole point of routing through this skill rather than typing the prompt directly.
- Producing a beautiful rewrite and then running it via a subagent. The whole loop is in-chat, every time.
- Skipping the implementation notes on a non-trivial execution because "nothing notable came up". If the task involved any decision the user didn't pre-approve, write it down.
- Writing implementation notes that read like marketing or apology. They're a log: one short sentence per entry, factual, no hedging.
