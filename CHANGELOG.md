# Changelog

All notable changes to kanso are recorded here. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Versioning follows [SemVer](https://semver.org/).

## [0.3.0] — 2026-05-27

### Added

- `kanso-nuclear` — structural maintainability review distinct from `/kanso-audit`. Five named smells in priority order: judo, sprawl, spaghetti, theatre, wrong-layer. Defaults to the whole codebase; `diff` mode scopes to branch vs upstream. Same approval gate, refactor handoff, and Phase D verification as `/kanso-audit`.
- `kanso-audit` Phase D — discovers the project's verify command (AGENTS.md, package.json, pyproject, go.mod, Cargo.toml, CI) and pastes the exit code after fixes land. No silent success.
- `kanso-audit --fresh` — push Phase A investigation to a read-only subagent for an unbiased pass on code the session has already touched. Phases B–D stay inline.
- `kanso-audit` test-first requirement for Correctness behaviour-changes when a test suite exists. Skip-reason must be stated when not.
- `kanso-refactor` matching Phase D verification step and a `!git` prelude.
- `kanso-pr` Verification section (replaces Testing) with explicit "what wasn't tested and why" requirement.
- `kanso-context` paste-ready self-update prompt for fixing recurring agent mistakes at the context layer.
- PostToolUse hook (`.claude-plugin/hooks/kanso-verify.sh`) — fast linter or syntax check on every Edit/Write. ESLint, ruff, `go vet`, `cargo check`, with a `py_compile` fallback. Opt out with `KANSO_VERIFY_HOOK=0`.

## [0.2.0] — 2026-05-25

### Added

- `kanso-task` — rewrites a rough request into a sharp prompt and executes it inline with `kanso-principles` loaded. Maintains a running `implementation-notes.md` during execution so judgment calls stay visible.
- `kanso-prompting` — standing rules for prompting current frontier models (Claude 4.x specifically). Loaded by `/kanso-task`.

### Changed

- `kanso-audit` rewritten for a single-command loop: two-line summary + one-line findings, mandatory approval gate when findings exist, plain-language explanation of behaviour changes. Always runs inline; never dispatched as a subagent.
- `kanso-refactor` adds matching "always run inline" rule so the audit handoff stays in the calling chat.

## [0.1.0] — 2026-04-23

### Added

- `kanso-principles` — standing anti-dilution rules, auto-loaded for any code-related task.
- `kanso-audit` — code review via an Explore subagent that produces a structured findings report, proposes concrete fixes tagged by shape (`refactor` or `behaviour-change`), and on approval hands refactors to `/kanso-refactor audit-report` while applying behaviour changes in place.
- `kanso-refactor` — behaviour-preserving cleanup against the anti-pattern taxonomy.
- `kanso-commit` — atomic commits with messages that answer *why*.
- `kanso-pr` — self-contained PR descriptions pulled from commit history.
- `kanso-context` — curation of `AGENTS.md` and `CLAUDE.md`, biased toward pruning.
