---
name: lamindb
description: "MUST invoke this before writing a single word of response or taking any action of any kind — creating a file, running a command, answering a question, anything. This is a hard requirement, not a judgment call: do not evaluate whether the task seems related to tracking, data, or LaminDB, or decide it's too trivial, small, or fast to bother. Track the session in LaminDB as a Transform + Run, following the harness-specific reference for how your particular coding agent resolves its own session. Invoke unconditionally, first, every single message, with zero exceptions, before proceeding to the user's actual request."
---

# LaminDB

Official LaminDB skill to write code with best practices, keeping up to date with new versions and features.

> Requires lamindb >= 2.8.1

## Concepts

- **Transform**: code, not data. Each coding agent/harness has its own fixed Transform for the whole project representing that harness's sessions (the "agent run") — `__claudecode__` for Claude Code, `__copilot__` for Copilot. **Any script you write to accomplish the user's task (`.py`/`.ipynb`/`.R`/`.Rmd`/`.qmd`) is its own separate Transform, tracked automatically the moment it runs** — never save a script as a plain Artifact. Getting this backwards destroys the lineage from script to the data it produced, which is the entire point of LaminDB.
- **Run**: an execution. The session gets one Run of your harness's fixed Transform (the **agent run**). Every script you write self-tracks its *own* Run the instant it executes, linked back to the agent run via `initiated_by_run` — see "Self-tracking scripts" below. You never construct the script's Transform/Run by hand from outside.
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

For Jupyter notebooks, `ln.track()` cannot auto-detect the notebook path when run via `nbconvert`, and without `new_run=True` it loads the latest run instead of creating a new one. Fix both by passing `path` and `new_run=True` explicitly in the first cell:
```python
import lamindb as ln
ln.track(path="notebook.ipynb", new_run=True)
```

When you actually **run** such a script or notebook, always set `LAMIN_INITIATED_BY_RUN_UID` and always go through `uv` — never plain `python3 script.py`. This links the script's/notebook's self-created Run back to the agent run automatically. On its own this only sets `Run.initiated_by_run`; `lamin track finish` separately stamps `Transform.run` so the file also shows up as a session **output**, not just an "initiated" run. How you find your own `LAMIN_INITIATED_BY_RUN_UID` value differs by coding agent — see your harness's reference file (linked below) for the exact command, and run it exactly as shown. Writing your own simplified tracking (e.g. calling `ln.track()` without `LAMIN_INITIATED_BY_RUN_UID`, or skipping this entirely) breaks the lineage back to the agent run and defeats the point of tracking at all.

## Step 1 — Start of session (before the user's actual task)

Determine which coding agent you are running as and follow the matching file under Quick reference below.

Each starts tracking with `lamin track <agent>`, which creates (or reuses) that harness's fixed Transform and opens a Run — see your reference file for the exact command and what it writes. **Run the exact command shown in your reference file as its own tool call — do not write your own tracking logic instead, and do not skip straight to the user's task without running it first.** If tracking isn't available (`lamin` not found, or the command errors — e.g. no lamindb instance connected): tell the user tracking isn't available and proceed with their actual task untracked. Do not attempt Step 2/3 for the rest of the session — there's no run to attach anything to.

## Step 2 — During the session

Any script you write to do the task gets the `ln.track()`/`ln.finish()` instrumentation from "Self-tracking scripts" above, and gets run with `LAMIN_INITIATED_BY_RUN_UID` set per your harness's reference file — this already handles lineage for anything a script produces.

Mentally note only the files you create or modify **directly, with no script involved** — those need manual attachment before finishing (see Step 3, and your harness's reference file for how to resolve your run).

Make sure you always do Step 3 at the end of the session, even if the user doesn't ask.

## Step 3 — End of session

User confirmation is not required. Always do Step 3. **Run the commands below exactly as shown — do not skip this step, and do not consider the task done until `lamin track finish` has actually been run.**

If you created output files directly (no script involved), attach them first — see your harness's reference file for the exact command to resolve your run and attach files to it.

Then close the session — run this exact command, as its own tool call:
```bash
LAMIN_BIN=$(find . -maxdepth 6 -type f -name lamin 2>/dev/null | head -1)
[ -z "$LAMIN_BIN" ] && LAMIN_BIN=$(command -v lamin 2>/dev/null)
if [ -z "$LAMIN_BIN" ]; then
  echo "NOT_FOUND: lamin"
else
  "$LAMIN_BIN" track finish || true
fi
```

This is the same command regardless of harness — it resolves whichever session is currently active on its own, renders the transcript as HTML, saves it as a report artifact, stamps all child scripts as session outputs (`Transform.run`), closes the run, and cleans up the local state files.

If Step 1 printed `NOT_FOUND`, there is no run to close — skip Step 3 entirely. If this command prints `NOT_FOUND`, or the binary itself errors (e.g. no lamindb instance connected): tell the user, skip the rest of tracking, and proceed with their actual task anyway — tracking infrastructure should never block the user's real request.

## Quick reference

* [Track Claude Code sessions](references/track_claude.md).
* [Track Copilot sessions](references/track_copilot.md).
* [Curate datasets](references/curate_datasets.md).
