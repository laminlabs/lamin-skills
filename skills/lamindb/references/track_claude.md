# Track Claude Code session in LaminDB

See [SKILL.md](../SKILL.md) for concepts and the shared steps — this covers only what's specific to Claude Code.

## Step 1 — Start of session

Run this now, before anything else:
```bash
lamin track claude --name "<one sentence describing this session's task>" || echo "LAMIN_ESCALATE"
```
Escalate to the fallback below only if the output literally contains `LAMIN_ESCALATE` — under no other circumstance (not a lamindb warning, not wanting to double-check, not comparing against a local virtualenv version) should you run any additional command before or instead of accepting this result. `LAMIN_ESCALATE` means `lamin` is likely only installed in a project-local virtualenv rather than on `PATH`:
```bash
LAMIN_BIN=$(find . -maxdepth 6 -type f -name lamin 2>/dev/null | head -1)
[ -z "$LAMIN_BIN" ] && LAMIN_BIN=$(command -v lamin 2>/dev/null)
if [ -z "$LAMIN_BIN" ]; then
  echo "NOT_FOUND: lamin"
else
  "$LAMIN_BIN" track claude --name "<one sentence describing this session's task>" || true
fi
```

This writes `.claude/.lamindb_run_uid_${CLAUDE_CODE_SESSION_ID}` and `.claude/.lamindb_transcript_path_${CLAUDE_CODE_SESSION_ID}`, keyed by Claude Code's own session id — safe for parallel sessions in the same directory since each gets its own uniquely suffixed file.

## Running self-tracking scripts and notebooks

`$CLAUDE_CODE_SESSION_ID` is already set in every subprocess Claude Code spawns, so finding your own run is a plain `cat`. Run the script or notebook exactly like you'd run any other one in this project — same tool, same environment — just with `LAMIN_INITIATED_BY_RUN_UID` set first:

```bash
LAMIN_INITIATED_BY_RUN_UID=$(cat .claude/.lamindb_run_uid_${CLAUDE_CODE_SESSION_ID}) <however you'd normally run this file> || echo "LAMIN_ESCALATE"
```
Escalate to the fallback below only if the output literally contains `LAMIN_ESCALATE` — regardless of the specific reason (wrong interpreter name, missing lamindb, anything else). Under no other circumstance should you run any additional command before or instead of accepting this result:
```bash
LAMIN_INITIATED_BY_RUN_UID=$(cat .claude/.lamindb_run_uid_${CLAUDE_CODE_SESSION_ID}) uv run --with lamindb python script.py
```

## Step 3 — Attaching direct output files

If you created output files directly (no script involved):
```bash
uv run --with lamindb python -c "
import lamindb as ln
from pathlib import Path
run = ln.Run.get(uid=Path('.claude/.lamindb_run_uid_${CLAUDE_CODE_SESSION_ID}').read_text().strip())
ln.Artifact('output.csv', description='<what it is>', run=run).save()
# repeat for each direct file
"
```

Then close the session per [SKILL.md](../SKILL.md) Step 3 (`lamin track finish`).
