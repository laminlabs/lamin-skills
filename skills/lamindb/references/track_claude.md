# Track Claude Code session in LaminDB

## Concepts
- **Transform**: code, not data. `__claudecode__` is the one fixed Transform for the whole project representing the chat session itself (the "agent run"). **Any script you write to accomplish the user's task (`.py`/`.ipynb`/`.R`/`.Rmd`/`.qmd`) is its own separate Transform, tracked automatically the moment it runs** — never save a script as a plain Artifact. Getting this backwards destroys the lineage from script to the data it produced, which is the entire point of LaminDB.
- **Run**: an execution. The session gets one Run of `__claudecode__` (the **agent run**). Every script you write self-tracks its *own* Run the instant it executes, linked back to the agent run via `initiated_by_run` — see "Self-tracking scripts" below. You never construct the script's Transform/Run by hand from outside.
- **Two distinct link fields — do not conflate them**: `Run.initiated_by_run` (on the *Run* model) says "this execution was triggered by that other run" — it only exists once a script actually executes, and renders in its own "This run initiated" panel in the UI, not as an output. `Transform.run` (on the *Transform* model, separate field) says "this piece of code was authored/produced during that run" — it's what makes a script show up in the agent run's **Output** column (alongside artifacts), the way a plain output file does. `ln.track()` never sets `Transform.run` on its own — `lamin track finish` stamps it explicitly at session close, so a script counts as a session output even if it's the *only* thing produced.
- **Never save a script as a plain Artifact.** Scripts (`.py`/`.ipynb`/`.R`/`.Rmd`/`.qmd`) must use `ln.track()` inside them. If you call `ln.Artifact("script.py").save()` you destroy the lineage between the code and the data it produced — that is the entire point of LaminDB and must never happen.
- **run.report**: rendered HTML of the transcript, saved as an Artifact and linked to the agent run.
- **Artifact**: data only — output files (csv, txt, images, fasta, etc.). A script's own `ln.Artifact(path).save()` calls (no `run=` needed) auto-attach to that script's own run. Only files you create directly, with no script involved, get attached to the agent run manually.

## Self-tracking scripts and notebooks

Every script or notebook you write to do the user's actual task must instrument itself — this is what gives each output file a real lineage back to the exact code that produced it:

```python
import lamindb as ln
ln.track()
# ... the actual task ...
ln.Artifact("output.csv", description="...").save()  # no run= needed, auto-attaches
ln.finish()
```

**Running Python scripts** — always set `LAMIN_INITIATED_BY_RUN_UID` and always go through `uv`, never plain `python3 script.py`:
```bash
LAMIN_INITIATED_BY_RUN_UID=$(cat .claude/.lamindb_run_uid) uv run --with lamindb python script.py
```

**Jupyter notebooks** — `ln.track()` cannot auto-detect the notebook path when run via `nbconvert`, and without `new_run=True` it loads the latest run instead of creating a new one. Fix both by passing `path` and `new_run=True` explicitly in the first cell:
```python
import lamindb as ln
ln.track(path="notebook.ipynb", new_run=True)
```
Then execute with:
```bash
LAMIN_INITIATED_BY_RUN_UID=$(cat .claude/.lamindb_run_uid) uv run --with lamindb --with jupyter jupyter nbconvert --to notebook --execute notebook.ipynb --inplace
```

`LAMIN_INITIATED_BY_RUN_UID` links the script's/notebook's self-created Run back to the agent run automatically. This alone only sets `Run.initiated_by_run`; `lamin track finish` separately stamps `Transform.run` so the file also shows up as a session **output**, not just an "initiated" run.

## Step 1 — Start of session (before the user's actual task)

Start tracking with the `lamin` CLI (installed alongside `lamindb`). If `lamin` is not on `PATH`, it may only be in a project-local `.venv` — try `.venv/bin/lamin` before concluding tracking is unavailable:

```bash
LAMIN_BIN=$(command -v lamin 2>/dev/null || find . -maxdepth 6 -type f -name lamin 2>/dev/null | head -1)
if [ -z "$LAMIN_BIN" ]; then
  echo "NOT_FOUND: lamin"
else
  "$LAMIN_BIN" track claude --name "<one sentence describing this session's task>" || true
fi
```

If this prints `NOT_FOUND: ...` (or the command errors, e.g. no lamindb instance connected): tell the user tracking isn't available and proceed with their actual task untracked. Do not attempt Step 2/3 for the rest of the session — there's no run to attach anything to.

Otherwise this creates (or reuses) the `__claudecode__` Transform, opens a Run, and writes `.claude/.lamindb_run_uid` and `.claude/.lamindb_transcript_path` for use at session end.

## Step 2 — During the session

Any script you write to do the task gets the `ln.track()`/`ln.finish()` instrumentation from "Self-tracking scripts" above, and gets run with `LAMIN_INITIATED_BY_RUN_UID` set — this already handles lineage for anything a script produces.

Mentally note only the files you create or modify **directly, with no script involved** — those need manual attachment before finishing (see Step 3).

Make sure you always do Step 3 at the end of the session, even if the user doesn't ask.

## Step 3 — End of session

User confirmation is not required. Always do Step 3.

**If you created output files directly** (no script involved), attach them first in one `uv` call before closing:
```bash
uv run --with lamindb python -c "
import lamindb as ln
from pathlib import Path
run = ln.Run.get(uid=Path('.claude/.lamindb_run_uid').read_text().strip())
ln.Artifact('output.csv', description='<what it is>', run=run).save()
# repeat for each direct file
"
```

Then close the session:
```bash
LAMIN_BIN=$(command -v lamin 2>/dev/null || find . -maxdepth 6 -type f -name lamin 2>/dev/null | head -1)
if [ -z "$LAMIN_BIN" ]; then
  echo "NOT_FOUND: lamin"
else
  "$LAMIN_BIN" track finish || true
fi
```

This renders the transcript as HTML, saves it as a report artifact, stamps all child scripts as session outputs (`Transform.run`), closes the run, and cleans up the `.claude/` state files.

If Step 1 printed `NOT_FOUND`, there is no run to close — skip Step 3 entirely. If the command prints `NOT_FOUND`, or the binary itself errors (e.g. no lamindb instance connected): tell the user, skip the rest of tracking, and proceed with their actual task anyway — tracking infrastructure should never block the user's real request.
