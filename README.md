# safe-crontab-edit

A Claude Code / agent skill for editing a user crontab from the command line
without silently wiping it.

Two rules: **back up first**, then **install via stdin** (`crontab - < file`) and
verify by reading back — never the `crontab <file>` form, which can silently
no-op in sandboxed/macOS shells.

## Install

```bash
npx skills add cmflynn/safe-crontab-edit-skill
```

Or copy `skills/safe-crontab-edit/SKILL.md` into `~/.claude/skills/safe-crontab-edit/`.

## License

MIT
