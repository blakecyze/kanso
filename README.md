# kanso

> 簡素 — elimination of clutter.

Six Claude Code skills for people who think AI-generated code is too long. kanso audits, refactors, commits, and writes PRs with a curatorial bias: delete before you add, match the repo's voice, earn every line. Install in one line:

```
/plugin marketplace add blakecyze/kanso && /plugin install kanso
```

A suite of Claude Code skills for keeping AI-generated code tight, intentional, and free of dilution.

## What this is

Six skills that encode anti-dilution principles and taste into reusable Claude Code invocations. Built around the research finding that LLM-generated code is systematically 2.2× more verbose than human baselines, with predictable anti-patterns (tautological comments, defensive programming theatre, filler variables, premature abstraction, fake test coverage) appearing in 80-100% of AI-generated repositories.

The suite is curatorial, not generative. The bias is toward deletion.

## The skills

| Skill | Purpose | Invocation |
|---|---|---|
| `kanso-principles` | Standing anti-dilution rules. Auto-loads into every code-related task. | Not manually invoked |
| `kanso-audit` | Read-only code review. Forks context to an Explore subagent. Produces a structured findings report. | `/kanso-audit [scope]` |
| `kanso-refactor` | Behaviour-preserving cleanup. Attacks the anti-pattern taxonomy. Refactor and behaviour change never share a commit. | `/kanso-refactor [scope]` |
| `kanso-commit` | Atomic commits with messages that answer *why*. Detects and matches repo convention. | `/kanso-commit` |
| `kanso-pr` | Self-contained PR descriptions. Pulls from commit history. Matches voice of recently merged PRs. | `/kanso-pr` |
| `kanso-context` | Curates AGENTS.md and CLAUDE.md. Prunes more than it adds. Empirically grounded in Vercel and ETH Zurich benchmarks. | `/kanso-context [mode]` |

## Install

### As a plugin (recommended)

```
/plugin marketplace add blakecyze/kanso
/plugin install kanso
```

### Manual, personal scope

```bash
git clone https://github.com/blakecyze/kanso ~/kanso
mkdir -p ~/.claude/skills
cp -r ~/kanso/skills/* ~/.claude/skills/
```

### Manual, project scope

```bash
mkdir -p .claude/skills
cp -r path/to/kanso/skills/* .claude/skills/
```

Skills auto-load on next Claude Code session (or immediately, via the file watcher).

## Design principles

**Skills over legacy slash commands.** Anthropic merged the two formats in Claude Code v2.x; skills win on naming conflicts, support live reload, and expose frontmatter controls (`disable-model-invocation`, `context: fork`) that commands cannot use.

**Descriptions describe triggers, not workflows.** Summarising a skill's workflow in its `description` causes Claude to follow the synopsis and skip the body. Every description states triggering conditions only.

**Side-effecting skills are manual-only.** Everything that writes files, commits, or opens PRs carries `disable-model-invocation: true`. Only `kanso-audit` auto-invokes because it's read-only.

**Audit forks context.** `/kanso-audit` runs in an Explore subagent so findings don't pollute the working session.

**Context target is AGENTS.md.** Cross-tool standard (Linux Foundation, December 2025). Works for Claude Code, Codex, Cursor, Copilot, and others. CLAUDE.md is only updated when guidance is Claude-specific.

**Anti-dilution rules live in one standing skill.** `kanso-principles` auto-loads as background context rather than being copy-pasted into every skill.

## Cross-cutting rules

- Prefer deletion over addition
- Preserve the author's voice, don't impose a generic one
- Match existing repo conventions before introducing new ones
- Surface questionable changes as questions rather than silently making them
- No filler, no hedging, no corporate prose

## Research foundations

The design is pinned to empirical work from 2025-2026:

- [CodeRabbit State of AI vs Human Code Generation](https://www.businesswire.com/news/home/20251217666881/en/)
- [OX Security 10 AI code anti-patterns](https://www.ox.security/blog/)
- [SlopCodeBench](https://arxiv.org/abs/2603.24755)
- [Vercel AGENTS.md agent evals](https://vercel.com/blog/agents-md-outperforms-skills-in-our-agent-evals)
- [ETH Zurich AGENTbench study](https://arxiv.org/html/2602.11988v1)
- [Google Engineering Practices](https://google.github.io/eng-practices/)
- [Palantir Code Review Best Practices](https://blog.palantir.com/code-review-best-practices-19e02780015f)

## Contributing

Skills should be tight, specific, and pull their weight. A skill that can't describe its trigger in one sentence probably isn't a skill; it's a note.

## License

MIT.
