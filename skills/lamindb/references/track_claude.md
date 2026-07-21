# Track Claude Code session in LaminDB

See [SKILL.md](../SKILL.md) for concepts and the shared steps — this covers only what's specific to Claude Code.

## Step 1 — Start of session

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

`$CLAUDE_CODE_SESSION_ID` is already set in every subprocess Claude Code spawns, so finding your own run is a plain `cat`:

```bash
LAMIN_INITIATED_BY_RUN_UID=$(cat .claude/.lamindb_run_uid_${CLAUDE_CODE_SESSION_ID}) uv run --with lamindb python script.py
```

For notebooks:
```bash
LAMIN_INITIATED_BY_RUN_UID=$(cat .claude/.lamindb_run_uid_${CLAUDE_CODE_SESSION_ID}) uv run --with lamindb --with jupyter jupyter nbconvert --to notebook --execute notebook.ipynb --inplace
```

## Step 3 — Attaching direct output files

If you created output files directly (no script involved):
```bash
uv run --with lamindb python -c "
import lamindb as ln
from pathlib import Path
import os
run = ln.Run.get(uid=Path(f'.claude/.lamindb_run_uid_{os.environ[\"CLAUDE_CODE_SESSION_ID\"]}').read_text().strip())
ln.Artifact('output.csv', description='<what it is>', run=run).save()
# repeat for each direct file
"
```

Then close the session as described in [SKILL.md](../SKILL.md) (`lamin track finish`, same command regardless of harness).
