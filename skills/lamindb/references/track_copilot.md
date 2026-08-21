# Track Copilot session in LaminDB

See [SKILL.md](../SKILL.md) for concepts and the shared steps — this covers only what's specific to Copilot.

**Do not write your own tracking logic.** Run every command below exactly as shown, as its own tool call, in order — [SKILL.md](../SKILL.md)'s Step 1 (`lamin settings dev-dir get`) first, then [SKILL.md](../SKILL.md)'s ask-the-user step, then Step 1 here, then the script/notebook command each time you run one, and Steps 3a, 3b, 3c in that order at the end. **You must actually run that dev-dir command — never assume it equals the current working directory, even if that seems obvious.** Don't skip a step because the task seems simple. If the user declined tracking at the ask-the-user step, stop here — there's nothing further to run, including Step 3. Otherwise, don't consider it finished until Step 3c's `lamin finish` has actually run.

## Step 1 — Start of session

Run this now (no `cd` needed — it resolves the dev-dir internally). **`--name` is mandatory — never omit it, never run this command without it:**
```bash
lamin track copilot --name "<one sentence describing this session's task>"
```
Escalate to the fallback below only if this command errors (non-zero exit status) — under no other circumstance (not a lamindb warning, not wanting to double-check, not comparing against a local virtualenv version) should you run any additional command before or instead of accepting this result. A command error here usually means `lamin` is likely only installed in a project-local virtualenv rather than on `PATH`:
```bash
LAMIN_BIN=$(find . -maxdepth 6 -type f -name lamin 2>/dev/null | head -1)
[ -z "$LAMIN_BIN" ] && LAMIN_BIN=$(command -v lamin 2>/dev/null)
if [ -z "$LAMIN_BIN" ]; then
  echo "NOT_FOUND: lamin"
else
  "$LAMIN_BIN" track copilot --name "<one sentence describing this session's task>"
fi
```

This reads Copilot's own `$COPILOT_AGENT_SESSION_ID` (set in every tool call's environment, the same idea as `$CLAUDE_CODE_SESSION_ID` for Claude Code) and writes `.copilot/.lamindb_run_uid_copilot_${COPILOT_AGENT_SESSION_ID}` — safe for parallel sessions in the same directory, since each gets its own uniquely suffixed file. If a dev-dir is configured, this lives there instead of cwd, so `lamin finish` finds it consistently regardless of which directory it's invoked from.

You don't need to remember anything from this command's output — every later command below reads `$COPILOT_AGENT_SESSION_ID` from its own environment directly, the same way this one did.

## Running self-tracking scripts and notebooks

Run this exact pattern every time you execute a script or notebook — never without this wrapper, and never a hand-rolled `ln.track()` call without it either. Prefix it with `cd "<dev-dir>" &&` using the path resolved in [SKILL.md](../SKILL.md)'s Step 1, if any. **Do not add flags, error-suppression (`2>/dev/null`, `|| true`), or any other modification to the `cat` command — run it exactly as shown.** If the file doesn't exist, let `cat` fail visibly rather than silently substituting an empty value.

```bash
printf 'y\n' | LAMIN_INITIATED_BY_RUN_UID=$(cat ".copilot/.lamindb_run_uid_copilot_${COPILOT_AGENT_SESSION_ID}") <however you'd normally run this file>
```
The leading `printf 'y\n' |` auto-answers the "overwrite existing source code?" prompt `ln.track()` shows when a previously-tracked script's content has changed — normal when iterating — otherwise it hangs/crashes waiting for input that will never come.

Run the file itself exactly like you'd run any other script or notebook in this project — same tool, same environment — the only requirement is that `LAMIN_INITIATED_BY_RUN_UID` is set first.

Escalate to the fallback below only if this command errors (non-zero exit status) — regardless of the specific reason (wrong interpreter name, missing lamindb, anything else). Under no other circumstance should you run any additional command before or instead of accepting this result. Retry as a separate command via `uv run`:
```bash
printf 'y\n' | LAMIN_INITIATED_BY_RUN_UID=$(cat ".copilot/.lamindb_run_uid_copilot_${COPILOT_AGENT_SESSION_ID}") uv run --with lamindb python script.py
```

## Step 3a — Attaching direct output files

If you created output files directly (no script involved), attach them. No `cd` needed for this one — just build the path directly using the dev-dir resolved in [SKILL.md](../SKILL.md)'s Step 1, if any (otherwise use the plain relative path shown). Run this the same way you'd normally run Python in this project — same tool, same environment:
```bash
python3 -c "
import lamindb as ln
from pathlib import Path
run = ln.Run.get(uid=Path('<dev-dir, if any>/.copilot/.lamindb_run_uid_copilot_${COPILOT_AGENT_SESSION_ID}').read_text().strip())
ln.Artifact('output.csv', key='<meaningful/folder/path>/output.csv', description='<what it is>', run=run).save()
# repeat for each direct file
"
```
Escalate to the fallback below only if this command errors (non-zero exit status) — regardless of the specific reason (wrong interpreter name, missing lamindb, anything else). Under no other circumstance should you run any additional command before or instead of accepting this result:
```bash
uv run --with lamindb python -c "
import lamindb as ln
from pathlib import Path
run = ln.Run.get(uid=Path('<dev-dir, if any>/.copilot/.lamindb_run_uid_copilot_${COPILOT_AGENT_SESSION_ID}').read_text().strip())
ln.Artifact('output.csv', key='<meaningful/folder/path>/output.csv', description='<what it is>', run=run).save()
# repeat for each direct file
"
```

## Step 3b — Cleaning up superseded transform versions

Run [SKILL.md](../SKILL.md)'s Step 3b command — first as the plan, then, once the user approves, again with `LAMIN_CLEANUP_APPLY=1` — with this path substituted for `<your harness run-uid state file>` in both passes. It's the same file Step 1 wrote, built from the dev-dir resolved in [SKILL.md](../SKILL.md)'s Step 1, if any (otherwise the plain relative path):

```
<dev-dir, if any>/.copilot/.lamindb_run_uid_copilot_${COPILOT_AGENT_SESSION_ID}
```

`$COPILOT_AGENT_SESSION_ID` expands inside the `python -c` string because the surrounding shell quoting is double quotes — leave it as the variable rather than pasting a literal session id.

## Step 3c — Closing the session

Run [SKILL.md](../SKILL.md)'s Step 3c closing command (`lamin finish`) as its own tool call — it reads `$COPILOT_AGENT_SESSION_ID` from its own environment the same way Step 1 did. Run it *after* Step 3b, never before, since it deletes the run-uid state file that Step 3b reads. Don't stop after just writing/running the user's script.
