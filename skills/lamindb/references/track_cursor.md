# Track Cursor IDE session in LaminDB

See [SKILL.md](../SKILL.md) for the shared steps. This reference covers only Cursor IDE Agent chats; it does not apply to Cursor's standalone CLI or cloud agents.

Cursor does not expose its conversation ID to commands run by the IDE Agent. `lamin finish` uses the Run UID printed by `lamin track cursor` to find the conversation in Cursor's local chat database, then reads the matching JSONL transcript. Do not generate, print, pass, or search for a session ID yourself.

If shared Step 1 starts at the base dev-dir, choose a task-specific branch name with a unique agent-chosen suffix of at least eight hexadecimal characters, for example `favorite-protein-fasta-a1b2c3d4`. No extra shell command is needed for the suffix. Keep this branch dedicated to the current Cursor conversation; Cursor's tracking state is branch-local.

If the user chose **Do not track**, stop here. Otherwise complete [SKILL.md](../SKILL.md)'s Step 1, including its worktree prerequisite and session-working-directory resolution, before running the commands below. Do not write your own tracking logic.

## Step 1 — Start of session

Run from the resolved session working directory as its own tool call. `--name` is mandatory:

```bash
lamin track cursor --name "<one sentence describing this session's task>"
```

Only if this command errors, use the same `LAMIN_BIN` fallback described in [SKILL.md](../SKILL.md), substituting `track cursor --name "<one sentence describing this session's task>"` for the command arguments.

This writes the active Run UID to `.cursor/.lamindb_run_uid_cursor` inside the worktree. A follow-up in the same Cursor conversation should repeat `lamin track cursor --name "<one sentence describing the follow-up>"` from the same worktree, which resumes the existing Run.

## Running self-tracking scripts and notebooks

Use the matching Python executable from the environment that provided `lamin`, as required by [SKILL.md](../SKILL.md). Run each script or notebook with the agent Run UID linked:

```bash
printf 'y\n' | LAMIN_INITIATED_BY_RUN_UID=$(cat .cursor/.lamindb_run_uid_cursor) <however you'd normally run this file>
```

Do not suppress errors from `cat`, and do not run the file without the wrapper. Follow the shared lineage-verification requirement before execution.

## Step 3 — Attaching direct output files

For files created directly, with no script involved, attach them from the same worktree using the matching Python executable:

```bash
<matching-python-executable> -c "
import lamindb as ln
from pathlib import Path
run = ln.Run.get(uid=Path('.cursor/.lamindb_run_uid_cursor').read_text().strip())
ln.Artifact('output.csv', key='<meaningful/folder/path>/output.csv', description='<what it is>', run=run).save()
"
```

Then run the shared closing command `lamin finish` as its own tool call. It waits for Cursor's closing tool call to reach the transcript before rendering the report. A later finish after a follow-up updates the same Run's report.
