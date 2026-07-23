# Track Copilot session in LaminDB

See [SKILL.md](../SKILL.md) for concepts and the shared steps — this covers only what's specific to Copilot.

**Do not write your own tracking logic.** Run every command below exactly as shown, as its own tool call, in order — Step 1 now, then resolve your session id once, then the script/notebook command each time you run one, and Step 3 at the end. Don't skip a step because the task seems simple, and don't consider it finished until Step 3's `lamin track finish` has actually run.

## Step 1 — Start of session

Run this now, before anything else:
```bash
lamin track copilot --name "<one sentence describing this session's task>" || echo "LAMIN_ESCALATE"
```
Escalate to the fallback below only if the output literally contains `LAMIN_ESCALATE` — under no other circumstance (not a lamindb warning, not wanting to double-check, not comparing against a local virtualenv version) should you run any additional command before or instead of accepting this result. `LAMIN_ESCALATE` means `lamin` is likely only installed in a project-local virtualenv rather than on `PATH`:
```bash
LAMIN_BIN=$(find . -maxdepth 6 -type f -name lamin 2>/dev/null | head -1)
[ -z "$LAMIN_BIN" ] && LAMIN_BIN=$(command -v lamin 2>/dev/null)
if [ -z "$LAMIN_BIN" ]; then
  echo "NOT_FOUND: lamin"
else
  "$LAMIN_BIN" track copilot --name "<one sentence describing this session's task>" || true
fi
```

This resolves your current session on its own (no session-id environment variable exists for Copilot, unlike Claude Code) and writes `.copilot/.lamindb_run_uid_copilot_<session-id>` — safe for parallel sessions in the same directory, since each gets its own uniquely suffixed file.

## Resolve your session id once

Copilot has no environment variable identifying the current session, so immediately after Step 1, resolve your `SESSION_ID` once by embedding a fresh literal token in a command and matching it against your own session's transcript:

```bash
MARKER="lamin-<16+ random alphanumeric characters, mixing case and digits, e.g. 7kP2x9Qm4wZ1a6Tb — never a short or guessable string like abc123 or test1>"
SESSION_ID=$(grep -l "$MARKER" ~/.copilot/session-state/*/events.jsonl 2>/dev/null | head -1 | xargs dirname | xargs basename)
echo "$SESSION_ID"
```

`MARKER` must be a literal string written directly into the command — not computed at runtime (e.g. not `$(date +%s)`) — since it's matched against the exact command text Copilot already logged before this command ran. It must also have real entropy: a short or guessable marker (like `abc123`) risks colliding with another session's marker, or a stale entry from earlier in this same session, which would resolve the wrong `SESSION_ID`/`RUN_UID` and silently link things to the wrong session.

Do this **once**, then remember the `SESSION_ID` value printed here and reuse it literally — not re-derive it — in every command below for the rest of this session. Each separate tool call runs in a fresh subprocess, so the shell variable itself won't persist; only what you remember from this output carries forward.

## Running self-tracking scripts and notebooks

Run this exact pattern every time you execute a script or notebook, using the `SESSION_ID` you already resolved above — never without this wrapper, and never a hand-rolled `ln.track()` call without it either:

```bash
LAMIN_INITIATED_BY_RUN_UID=$(cat ".copilot/.lamindb_run_uid_copilot_<SESSION_ID resolved above>") <however you'd normally run this file> || echo "LAMIN_ESCALATE"
```

Run the file itself exactly like you'd run any other script or notebook in this project — same tool, same environment — the only requirement is that `LAMIN_INITIATED_BY_RUN_UID` is set first.

Escalate to the fallback below only if the output literally contains `LAMIN_ESCALATE` — regardless of the specific reason (wrong interpreter name, missing lamindb, anything else). Under no other circumstance should you run any additional command before or instead of accepting this result. Retry as a separate command, still using the same `SESSION_ID`, this time via `uv run`:
```bash
LAMIN_INITIATED_BY_RUN_UID=$(cat ".copilot/.lamindb_run_uid_copilot_<SESSION_ID resolved above>") uv run --with lamindb python script.py
```

## Step 3 — Attaching direct output files

If you created output files directly (no script involved), attach them using the same `SESSION_ID`:
```bash
uv run --with lamindb python -c "
import lamindb as ln
from pathlib import Path
run = ln.Run.get(uid=Path('.copilot/.lamindb_run_uid_copilot_<SESSION_ID resolved above>').read_text().strip())
ln.Artifact('output.csv', description='<what it is>', run=run).save()
# repeat for each direct file
"
```

Then run [SKILL.md](../SKILL.md)'s Step 3 closing command (`lamin track finish`) as its own tool call — it resolves which Copilot session is finishing the same way Step 1 does. Don't stop after just writing/running the user's script.
