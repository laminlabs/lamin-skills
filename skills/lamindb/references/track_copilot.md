# Track Copilot session in LaminDB

See [SKILL.md](../SKILL.md) for concepts and the shared steps — this covers only what's specific to Copilot.

**Do not write your own tracking logic.** Run every command below exactly as shown, as its own tool call, in order — Step 1 now, the script/notebook command each time you run one, and Step 3 at the end. Don't skip a step because the task seems simple, and don't consider it finished until Step 3's `lamin track finish` has actually run.

## Step 1 — Start of session

Run this now, before anything else:
```bash
LAMIN_BIN=$(find . -maxdepth 6 -type f -name lamin 2>/dev/null | head -1)
[ -z "$LAMIN_BIN" ] && LAMIN_BIN=$(command -v lamin 2>/dev/null)
if [ -z "$LAMIN_BIN" ]; then
  echo "NOT_FOUND: lamin"
else
  "$LAMIN_BIN" track copilot --name "<one sentence describing this session's task>" || true
fi
```

This resolves your current session on its own (no session-id environment variable exists for Copilot, unlike Claude Code) and writes `.claude/.lamindb_run_uid_copilot_<session-id>` — safe for parallel sessions in the same directory, since each gets its own uniquely suffixed file.

## Running self-tracking scripts and notebooks

Copilot has no environment variable identifying the current session, so a plain `cat` isn't enough — find your own run by embedding a fresh literal token in the command and matching it against your own session's transcript. Run this exact pattern every time you execute a script or notebook — never plain `python3 script.py`, and never a hand-rolled `ln.track()` call without this wrapper:

```bash
MARKER="lamin-<write a fresh random-looking token here each time>"
SESSION_ID=$(grep -l "$MARKER" ~/.copilot/session-state/*/events.jsonl 2>/dev/null | head -1 | xargs dirname | xargs basename)
LAMIN_INITIATED_BY_RUN_UID=$(cat ".claude/.lamindb_run_uid_copilot_${SESSION_ID}") uv run --with lamindb python script.py
```

`MARKER` must be a literal string written directly into the command — not computed at runtime (e.g. not `$(date +%s)`) — since it's matched against the exact command text Copilot already logged before this script started running.

For notebooks, same idea:
```bash
MARKER="lamin-<fresh token>"
SESSION_ID=$(grep -l "$MARKER" ~/.copilot/session-state/*/events.jsonl 2>/dev/null | head -1 | xargs dirname | xargs basename)
LAMIN_INITIATED_BY_RUN_UID=$(cat ".claude/.lamindb_run_uid_copilot_${SESSION_ID}") uv run --with lamindb --with jupyter jupyter nbconvert --to notebook --execute notebook.ipynb --inplace
```

## Step 3 — Attaching direct output files

If you created output files directly (no script involved), resolve your run the same way, then attach:
```bash
MARKER="lamin-<fresh token>"
SESSION_ID=$(grep -l "$MARKER" ~/.copilot/session-state/*/events.jsonl 2>/dev/null | head -1 | xargs dirname | xargs basename)
uv run --with lamindb python -c "
import lamindb as ln
from pathlib import Path
run = ln.Run.get(uid=Path('.claude/.lamindb_run_uid_copilot_${SESSION_ID}').read_text().strip())
ln.Artifact('output.csv', description='<what it is>', run=run).save()
# repeat for each direct file
"
```

Then run [SKILL.md](../SKILL.md)'s Step 3 closing command (`lamin track finish`) as its own tool call — it resolves which Copilot session is finishing the same way Step 1 does. Don't stop after just writing/running the user's script.
