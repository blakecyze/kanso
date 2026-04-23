# kanso

> 簡素 — elimination of clutter.

Six Claude Code skills for people who think AI-generated code is too long.

kanso audits, refactors, commits, and writes PRs with a curatorial bias: delete before you add, match the repo's voice, earn every line.

```
/plugin marketplace add blakecyze/kanso
/plugin install kanso
```

&nbsp;

## The skills

| Skill | What it does | Invocation |
|---|---|---|
| `kanso-principles` | Standing anti-dilution rules. Loaded automatically. | auto |
| `kanso-audit` | Read-only review. Runs in a forked subagent, returns a findings report. | `/kanso-audit [scope]` |
| `kanso-refactor` | Behaviour-preserving cleanup. Never mixes refactor with behaviour change. | `/kanso-refactor [scope]` |
| `kanso-commit` | Atomic commits with messages that answer *why*. | `/kanso-commit` |
| `kanso-pr` | Self-contained PR descriptions drawn from commit history. | `/kanso-pr` |
| `kanso-context` | Prunes and curates `AGENTS.md` / `CLAUDE.md`. | `/kanso-context [mode]` |

&nbsp;

## What it looks like

**Before `/kanso-refactor`:**

```python
def get_user(user_id):
    # Fetch user by ID
    try:
        result = db.query(User).filter(User.id == user_id).first()
        if result is not None:
            return result
        else:
            return None
    except Exception as e:
        return None
```

**After:**

```python
def get_user(user_id):
    return db.query(User).filter(User.id == user_id).first()
```

Five anti-patterns removed: tautological comment, filler variable, redundant `is not None` check, dead `else`, silent exception swallow.

&nbsp;

**`/kanso-commit` output:**

```
fix(auth): reject tokens issued before password change

Previously, changing the password did not invalidate existing sessions,
allowing a compromised token to remain valid indefinitely. Reject any
token with `iat` earlier than the user's password_updated_at.
```

&nbsp;

## Install

Plugin (recommended):

```
/plugin marketplace add blakecyze/kanso
/plugin install kanso
```

Manual, personal scope:

```bash
git clone https://github.com/blakecyze/kanso ~/kanso
mkdir -p ~/.claude/skills
cp -r ~/kanso/skills/* ~/.claude/skills/
```

Manual, project scope:

```bash
mkdir -p .claude/skills
cp -r path/to/kanso/skills/* .claude/skills/
```

Skills load on next session, or immediately via the file watcher.

&nbsp;

## How it behaves

- **Audit is read-only and forks its context.** Findings don't pollute the working session.
- **Everything that writes is manual-only.** Commit, PR, refactor, and context edits never auto-invoke.
- **Context target is `AGENTS.md`.** `CLAUDE.md` is only touched when the guidance is Claude-specific.
- **Voice preservation over house style.** If the repo is terse, kanso stays terse.

&nbsp;

## Contributing

A skill has to earn its place. If you can't describe its trigger in one sentence, it's a note, not a skill. See [CONTRIBUTING.md](CONTRIBUTING.md).

&nbsp;

## License

MIT.
