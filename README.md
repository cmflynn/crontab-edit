# safe-crontab-edit

A Claude Code / agent skill for safely installing, replacing, restoring, and
editing user crontabs from the command line — without silently losing jobs.

Editing a crontab is riskier than it looks: `crontab` overwrites the whole table
at once, the `crontab <file>` install form can **silently no-op** in sandboxed
environments (exit 0, nothing written — common on macOS), and crontabs routinely
contain secrets. This skill encodes a reliable
**back up → install via stdin → verify by reading back** loop that never trusts
an exit code alone.

## Install

```bash
npx skills add cmflynn/safe-crontab-edit-skill
```

Or copy `skills/safe-crontab-edit/SKILL.md` into your agent's skills directory
(e.g. `~/.claude/skills/safe-crontab-edit/`).

## What it covers

- Backing up the current crontab before any change
- Installing via stdin (`crontab -`) instead of the silently-failing `crontab <file>`
- Verifying the result by line-count parity and inspecting active job lines
- Detecting and handling a `CRON_ENABLED` master kill switch
- Spotting secrets (tokens/keys) before they leak
- macOS specifics (spool permissions, Full Disk Access, timezones)
- Restore / replace / add-single-entry / disable-without-deleting recipes

## License

MIT
