---
name: safe-crontab-edit
description: >-
  Edit, install, replace, or restore a user crontab from the command line
  without silently wiping it. Use whenever the task touches `crontab` — "restore
  my crontab", "install this crontab", "add a cron job", or editing a crontab
  backup file.
---

# Safe crontab editing

`crontab` overwrites the *entire* table at once, and the `crontab <file>` form
can silently no-op (exit 0, nothing written) in sandboxed/macOS shells. Two rules
prevent lost jobs.

## 1. Back up first

```bash
crontab -l > /tmp/crontab.backup.txt 2>/dev/null || echo "(no existing crontab)"
```

An empty/nonzero result just means the table is empty — that's fine.

## 2. Install via stdin, then verify by reading back

Pipe the content in on **stdin**. Do not use `crontab <file>`, and do not trust
the exit code — only the read-back proves it worked.

```bash
crontab - < /path/to/new_crontab.txt        # install (NOT: crontab file)
crontab -l | wc -l                           # verify: line count matches source
crontab -l                                   # verify: jobs are what you expect
```

If the read-back is empty or the count is wrong, the install failed — retry the
stdin form.

To edit in place without clobbering the rest, round-trip it: `crontab -l` to a
file, edit that file, then reinstall via stdin.
