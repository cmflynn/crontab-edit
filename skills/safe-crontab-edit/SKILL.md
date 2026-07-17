---
name: safe-crontab-edit
description: >-
  Safely install, replace, restore, or edit a user crontab from the command
  line without silently losing jobs. Use this whenever the task involves
  `crontab`, cron jobs, scheduled jobs, "restore my crontab", "replace the
  crontab", "add a cron entry", a crontab backup file, or any editing of
  scheduled tasks via the `crontab` command — especially on macOS, where the
  plain `crontab <file>` install can silently no-op. Trigger even when the user
  just says "install this crontab" or hands over a `crontab.*.txt` file.
---

# Safe crontab editing

Editing a crontab is deceptively risky: `crontab` overwrites the *entire* table
at once, some install forms fail **silently** (exit 0, nothing persisted), and
crontabs routinely contain secrets. This skill is a reliable
back-up → install-via-stdin → verify loop that never trusts an exit code alone.

## The core loop

Always follow these four steps in order. Skipping the verify step is how jobs
get silently lost.

### 1. Back up the current crontab first

`crontab` replaces the whole table, so capture what's there before you touch it —
even if you expect it to be empty. An empty read is itself useful information.

```bash
crontab -l > "/tmp/crontab.backup.$(date +%Y%m%d-%H%M%S).txt" 2>/dev/null \
  && echo "backed up" || echo "no existing crontab (or unreadable)"
```

`crontab -l` exits non-zero and prints `no crontab for <user>` when the table is
empty — that's normal, not an error.

### 2. Read the content before installing it

Whether restoring a backup or writing new entries, **read the file first** with
your file-reading tool. You are looking for three things:

- **Secrets.** Crontabs commonly carry `TOKEN=`, `API_KEY=`, `PASSWORD=`, OAuth
  tokens, or `$(cat ~/.config/.../token)` reads. These are about to become live
  in the user's crontab. Flag them to the user, and **never** copy a real secret
  into example text, a committed file, a screenshot, or anywhere it leaves the
  user's machine. Use a placeholder like `sk-...REDACTED` when you need to show a
  line.
- **A master kill switch.** Many well-built crontabs guard every job with a
  single variable, e.g. `CRON_ENABLED=0` plus each line prefixed with
  `[ "$CRON_ENABLED" = "1" ] && ...`. Check its value — it decides whether the
  jobs you install actually *run*. Report it explicitly.
- **What will fire and when.** Summarize the active (uncommented) schedule lines
  so the user can confirm intent before it goes live.

### 3. Install via stdin, not the file-path form

This is the most important lesson and the easiest to get wrong.

**Prefer piping the content in on stdin:**

```bash
crontab - < /path/to/new_crontab.txt      # or: cat file | crontab -
```

The `crontab <file>` positional form *can silently no-op* in sandboxed or
restricted execution environments (notably macOS under a command sandbox, or
without Full Disk Access): the command returns exit 0, prints nothing, yet the
table is never written. The stdin form reliably persists. If you only have the
positional form and it "succeeds" but step 4 reads back empty, switch to stdin.

Do **not** conclude success from the exit code. Only step 4 proves it worked.

### 4. Verify by reading back — count lines and inspect

An exit code of 0 is not proof. Read the table back and compare it to the source:

```bash
# line-count parity with the source file
echo "installed: $(crontab -l | wc -l | tr -d ' ')  source: $(wc -l < /path/to/new_crontab.txt | tr -d ' ')"

# show the active (uncommented, non-env) job lines, secrets redacted
crontab -l | grep -vE '^#|^$|^[A-Z_]+=' | sed -E 's/(sk-[a-zA-Z0-9]{6})[^ ]*/\1…REDACTED/g'

# confirm the kill-switch value, if the crontab uses one
crontab -l | grep -E '^CRON_ENABLED='
```

If the line count doesn't match, or the active jobs aren't what you expected,
the install failed or was partial — go back to step 3 (stdin form) and retry.
Report the concrete verification result, not just "done".

## Common operations

**Restore from a backup file** — steps 1–4 with the backup as the source.

**Replace with a new file** — identical flow; call out the *diff* from what was
installed (which jobs were added, removed, commented out, and whether the kill
switch changed).

**Add or edit a single entry without clobbering the rest** — never hand-write the
whole table from memory. Round-trip it: dump, edit the dump, reinstall.

```bash
crontab -l > /tmp/ct.txt 2>/dev/null            # 1. dump (may be empty)
# ...edit /tmp/ct.txt with a file tool: append/modify exactly the target line...
crontab - < /tmp/ct.txt                          # 3. reinstall via stdin
crontab -l                                        # 4. verify
```

**Disable everything without deleting** — if the crontab has a kill switch, set
`CRON_ENABLED=0` and reinstall; the jobs stay in the file but short-circuit.
Otherwise comment the lines (prefix `# `) rather than deleting them, so intent
and schedule are preserved.

**Disable a single job** — comment its line; keep the descriptive comment block
above it. A `# DISABLED (per user) <original line>` prefix keeps the original
recoverable.

## macOS specifics

- Spool dirs (`/usr/lib/cron/tabs`, `/var/at/tabs`) are **not world-readable**;
  a `Permission denied` when listing them is expected and not the failure. Use
  `crontab -l`, never a directory listing, to inspect the table.
- If installs succeed-but-don't-persist even via stdin, the terminal/process
  likely lacks **Full Disk Access**; tell the user to grant it to their terminal
  in System Settings → Privacy & Security → Full Disk Access.
- Times in cron are the machine's local timezone. Don't silently assume UTC vs.
  ET; if it matters, state which you're reasoning in.

## Guardrails

- Installing or enabling a crontab creates scheduled processes that run
  unattended. Do this only on the user's explicit instruction, and surface what
  will run and when before it goes live — a flipped kill switch (`0 → 1`) is a
  behavior change worth calling out loudly.
- Preserve the descriptive comment blocks in a crontab. They encode timing
  rationale and coordination between jobs (staggered minutes to avoid resource
  contention); dropping them loses hard-won operational knowledge.
