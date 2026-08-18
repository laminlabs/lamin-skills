# Track Claude Code session in LaminDB

See [SKILL.md](../SKILL.md) for concepts and the shared steps — this covers only what's specific to Claude Code.

## Step 1 — Start of session

Run this now, before anything else (no `cd` needed — it resolves the dev-dir internally, see [SKILL.md](../SKILL.md)'s Step 1). **`--name` is mandatory — never omit it, never run this command without it:**
```bash
lamin track claude --name "<one sentence describing this session's task>"
```
Escalate to the fallback below only if this command errors (non-zero exit status) — under no other circumstance (not a lamindb warning, not wanting to double-check, not comparing against a local virtualenv version) should you run any additional command before or instead of accepting this result. A command error here usually means `lamin` is likely only installed in a project-local virtualenv rather than on `PATH`:
```bash
LAMIN_BIN=$(find . -maxdepth 6 -type f -name lamin 2>/dev/null | head -1)
[ -z "$LAMIN_BIN" ] && LAMIN_BIN=$(command -v lamin 2>/dev/null)
if [ -z "$LAMIN_BIN" ]; then
  echo "NOT_FOUND: lamin"
else
  "$LAMIN_BIN" track claude --name "<one sentence describing this session's task>"
fi
```

This writes `.claude/.lamindb_run_uid_${CLAUDE_CODE_SESSION_ID}` and `.claude/.lamindb_transcript_path_${CLAUDE_CODE_SESSION_ID}`, keyed by Claude Code's own session id — safe for parallel sessions in the same directory since each gets its own uniquely suffixed file. If a dev-dir is configured, these live there instead of cwd, so `lamin finish` finds them consistently regardless of which directory it's invoked from.

## Running self-tracking scripts and notebooks

`$CLAUDE_CODE_SESSION_ID` is already set in every subprocess Claude Code spawns, so finding your own run is a plain `cat`. Run the script or notebook exactly like you'd run any other one in this project — same tool, same environment — just with `LAMIN_INITIATED_BY_RUN_UID` set first (and prefixed with `cd "<dev-dir>" &&` using the same remembered path, if any). **Do not add flags, error-suppression (`2>/dev/null`, `|| true`), or any other modification to the `cat` command — run it exactly as shown.** If the file doesn't exist, let `cat` fail visibly rather than silently substituting an empty value:

```bash
printf 'y\n' | LAMIN_INITIATED_BY_RUN_UID=$(cat .claude/.lamindb_run_uid_${CLAUDE_CODE_SESSION_ID}) <however you'd normally run this file>
```
The leading `printf 'y\n' |` auto-answers the "overwrite existing source code?" prompt `ln.track()` shows when a previously-tracked script's content has changed — normal when iterating — otherwise it hangs/crashes waiting for input that will never come.

Escalate to the fallback below only if this command errors (non-zero exit status) — regardless of the specific reason (wrong interpreter name, missing lamindb, anything else). Under no other circumstance should you run any additional command before or instead of accepting this result:
```bash
printf 'y\n' | LAMIN_INITIATED_BY_RUN_UID=$(cat .claude/.lamindb_run_uid_${CLAUDE_CODE_SESSION_ID}) uv run --with lamindb python script.py
```

## Step 3 — Attaching direct output files

If you created output files directly (no script involved), no `cd` needed for this one — just build the path directly using the dev-dir resolved in [SKILL.md](../SKILL.md)'s Step 1, if any (otherwise use the plain relative path shown). Run this the same way you'd normally run Python in this project — same tool, same environment:
```bash
python3 -c "
import lamindb as ln
from pathlib import Path
run = ln.Run.get(uid=Path('<dev-dir, if any>/.claude/.lamindb_run_uid_${CLAUDE_CODE_SESSION_ID}').read_text().strip())
ln.Artifact('output.csv', key='<meaningful/folder/path>/output.csv', description='<what it is>', run=run).save()
# repeat for each direct file
"
```
Escalate to the fallback below only if this command errors (non-zero exit status) — regardless of the specific reason (wrong interpreter name, missing lamindb, anything else). Under no other circumstance should you run any additional command before or instead of accepting this result:
```bash
uv run --with lamindb python -c "
import lamindb as ln
from pathlib import Path
run = ln.Run.get(uid=Path('<dev-dir, if any>/.claude/.lamindb_run_uid_${CLAUDE_CODE_SESSION_ID}').read_text().strip())
ln.Artifact('output.csv', key='<meaningful/folder/path>/output.csv', description='<what it is>', run=run).save()
# repeat for each direct file
"
```

Then close the session per [SKILL.md](../SKILL.md) Step 3 (`lamin finish`).
