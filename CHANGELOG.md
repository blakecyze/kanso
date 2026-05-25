# Changelog

All notable changes to kanso are recorded here. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/). Versioning follows [SemVer](https://semver.org/).

## [0.2.0] — unreleased

### Added

- `kanso-task` — rewrites a rough request into a sharp prompt and executes it inline with `kanso-principles` loaded. Maintains a running `implementation-notes.md` during execution so judgment calls stay visible.
- `kanso-prompting` — standing rules for prompting current frontier models (Claude 4.x specifically). Loaded by `/kanso-task`.

### Changed

- `kanso-audit` rewritten for a single-command loop: two-line summary + one-line findings, mandatory approval gate when findings exist, plain-language explanation of behaviour changes. Always runs inline; never dispatched as a subagent.
- `kanso-refactor` adds matching "always run inline" rule so the audit handoff stays in the calling chat.

## [0.1.0] — unreleased

### Added

- `kanso-principles` — standing anti-dilution rules, auto-loaded for any code-related task.
- `kanso-audit` — code review via an Explore subagent that produces a structured findings report, proposes concrete fixes tagged by shape (`refactor` or `behaviour-change`), and on approval hands refactors to `/kanso-refactor audit-report` while applying behaviour changes in place.
- `kanso-refactor` — behaviour-preserving cleanup against the anti-pattern taxonomy.
- `kanso-commit` — atomic commits with messages that answer *why*.
- `kanso-pr` — self-contained PR descriptions pulled from commit history.
- `kanso-context` — curation of `AGENTS.md` and `CLAUDE.md`, biased toward pruning.
