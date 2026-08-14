---
name: lamindb
description: "MUST invoke before responding to ANY message — including greetings, small talk, trivial math, anything that looks unrelated. Not a judgment call: never skip it, never decide a message is too trivial. Tracks this session in LaminDB as a Transform + Run. If you are about to respond without invoking this first, stop — that is already a mistake."
---

# LaminDB

Official LaminDB skill to write code with best practices, keeping up to date with new versions and features.

> Requires lamindb >= 2.8.1

## Concepts

- **Transform**: code, not data. Each coding agent/harness has its own fixed Transform for the whole project representing that harness's sessions (the "agent run") — internally keyed `__claudecode__` for Claude Code and `__copilot__` for Copilot. **These are internal database identifiers only, never CLI arguments or command names** — the actual commands are plain `lamin track claude` and `lamin track copilot`, with no underscores; see your harness's reference file for the exact syntax rather than constructing a command from these keys. **Any script you write to accomplish the user's task (`.py`/`.ipynb`/`.R`/`.Rmd`/`.qmd`) is its own separate Transform, tracked automatically the moment it runs** — never save a script as a plain Artifact. Getting this backwards destroys the lineage from script to the data it produced, which is the entire point of LaminDB.
- **Run**: an execution. The session gets one Run of your harness's fixed Transform (the **agent run**). Every script you write self-tracks its *own* Run the instant it executes, linked back to the agent run via `initiated_by_run` — see "Self-tracking scripts" below. You never construct the script's Transform/Run by hand from outside.
- **Two distinct link fields — do not conflate them**: `Run.initiated_by_run` (on the *Run* model) says "this execution was triggered by that other run" — it only exists once a script actually executes, and renders in its own "This run initiated" panel in the UI, not as an output. `Transform.run` (on the *Transform* model, separate field) says "this piece of code was authored/produced during that run" — it's what makes a script show up in the agent run's **Output** column (alongside artifacts), the way a plain output file does. `ln.track()` never sets `Transform.run` on its own — `lamin finish` stamps it explicitly at session close, so a script counts as a session output even if it's the *only* thing produced.
- **Never save a script as a plain Artifact.** Scripts (`.py`/`.ipynb`/`.R`/`.Rmd`/`.qmd`) must use `ln.track()` inside them. If you call `ln.Artifact("script.py").save()` you destroy the lineage between the code and the data it produced — that is the entire point of LaminDB and must never happen.
- **run.report**: rendered HTML of the transcript, saved as an Artifact and linked to the agent run.
- **Artifact**: data only — output files (csv, txt, images, fasta, etc.). A script's own `ln.Artifact(path).save()` calls (no `run=` needed) auto-attach to that script's own run. Only files you create directly, with no script involved, get attached to the agent run manually.
- **Always pass a meaningful `key`** when saving an Artifact — a stable, path-like name (e.g. `key="datasets/ataqseq_counts.csv"`), not left unset. Without a key, an Artifact can never be versioned against future updates to the same data. Only reuse the exact same key when a new save is genuinely a new version of that same dataset; use a distinct key otherwise, or unrelated saves will incorrectly get grouped into one version family.
- **When a script needs data that an earlier script in this workflow already produced, retrieve it from LaminDB — never read the local file path directly.** Use `ln.Artifact.get(key="...")` (the same key it was saved under) followed by `.load()`; this is what registers that artifact as this run's input and forms the lineage edge between the two scripts. Reading the file straight off disk produces the same result but leaves LaminDB with no record that the two scripts are connected, silently breaking the workflow's lineage graph.

## Self-tracking scripts and notebooks

Every script or notebook you write to do the user's actual task must instrument itself — this is what gives each output file a real lineage back to the exact code that produced it:

```python
import lamindb as ln
ln.track()
# if this step consumes an earlier step's output, retrieve it — never read the local file path directly:
# input_artifact = ln.Artifact.get(key="<key used when it was saved>")
# df = input_artifact.load()  # registers it as this run's input, forming the lineage edge
# ... the actual task ...
ln.Artifact("output.csv", key="<meaningful/folder/path>/output.csv", description="...").save()  # no run= needed, auto-attaches
ln.finish()
```

For Jupyter notebooks, `ln.track()` cannot auto-detect the notebook path when run via `nbconvert`, and without `new_run=True` it loads the latest run instead of creating a new one. Fix both by passing `path` and `new_run=True` explicitly in the first cell:
```python
import lamindb as ln
ln.track(path="notebook.ipynb", new_run=True)
```

When you actually **run** such a script or notebook, always set `LAMIN_INITIATED_BY_RUN_UID` — run it the same way you'd normally run code in this project, falling back to `uv run --with lamindb` if that fails for any reason (see your harness's reference file for the exact fallback command and the mechanical trigger condition to use). This links the script's/notebook's self-created Run back to the agent run automatically. On its own this only sets `Run.initiated_by_run`; `lamin finish` separately stamps `Transform.run` so the file also shows up as a session **output**, not just an "initiated" run. How you find your own `LAMIN_INITIATED_BY_RUN_UID` value differs by coding agent — see your harness's reference file (linked below) for the exact command, and run it exactly as shown. Writing your own simplified tracking (e.g. calling `ln.track()` without `LAMIN_INITIATED_BY_RUN_UID`, or skipping this entirely) breaks the lineage back to the agent run and defeats the point of tracking at all.

## Step 1 — Start of session (before the user's actual task)

First, resolve whether this instance has a configured development directory. This matters for one specific reason: when you later run a self-tracking script, lamindb derives that script's own Transform key from the directory it actually executes in — so the script needs to run from the dev-dir for that key to be stable, and lamindb has no other way to know where that is.
```bash
lamin settings dev-dir get
```
Escalate to the fallback below only if this command errors (non-zero exit status). Under no other circumstance should you run any additional command before or instead of accepting this result. A command error here usually means `lamin` is only installed in a project-local virtualenv rather than on `PATH`:
```bash
LAMIN_BIN=$(find . -maxdepth 6 -type f -name lamin 2>/dev/null | head -1)
[ -z "$LAMIN_BIN" ] && LAMIN_BIN=$(command -v lamin 2>/dev/null)
if [ -z "$LAMIN_BIN" ]; then
  echo "NOT_FOUND: lamin"
else
  "$LAMIN_BIN" settings dev-dir get
fi
```
If the output is the literal string `None`, this instance has no dev-dir configured — nothing further to do here. Otherwise, remember the printed path exactly as shown for later — you'll need it when running self-tracking scripts (see your harness's reference file for the exact command). `lamin track <agent>` and `lamin finish` already resolve the dev-dir internally, so neither needs a `cd` prefix; script execution is the one place that does, since that's where the actual working directory affects lamindb's own behavior. Don't rely on a shell variable to carry the path forward — each tool call may run in a fresh subprocess, so type the literal path again when you need it.

Determine which coding agent you are running as and follow the matching file under Quick reference below.

Before running the tracking command, ask the user a single yes/no question — "Should this session be tracked in LaminDB?". **If your harness has a dedicated clarifying-question or ask-user tool, you must use it — do not fall back to plain response text when a real interactive mechanism is available.** Only ask directly in your response text if no such tool exists at all. **This is a blocking question: stop and wait for the user's actual reply before doing anything else.** Do not assume an answer, do not phrase it as "I'll proceed unless you say no," and do not continue in the same turn — treat it exactly like any other question you'd wait for a real answer to. Ask this once, at the very start of the session, regardless of how obviously code-related the task seems — every session, no exceptions in either direction. If the user declines, do not run `lamin track <agent>` or attempt Step 2/3 for the rest of the session — there's no run to attach anything to. Only once the user has explicitly replied yes (or an equivalent affirmative), proceed with the command below.

Each starts tracking with `lamin track <agent>`, which creates (or reuses) that harness's fixed Transform and opens a Run — see your reference file for the exact command and what it writes. **Run the exact command shown in your reference file as its own tool call — do not write your own tracking logic instead, do not add any other command alongside it, and do not skip straight to the user's task without running it first.** If tracking isn't available (`lamin` not found, or the command errors — e.g. no lamindb instance connected): tell the user tracking isn't available and proceed with their actual task untracked. Do not attempt Step 2/3 for the rest of the session — there's no run to attach anything to.

## Step 2 — During the session

Every script you write to do the task — the first one and every later one, on any message in this session — gets the `ln.track()`/`ln.finish()` instrumentation from "Self-tracking scripts" above, and gets run with `LAMIN_INITIATED_BY_RUN_UID` set per your harness's reference file — this already handles lineage for anything a script produces. Having done Step 1 once already does not exempt a later script from this wrapper. **This applies even to a script that just generates, fetches, or prepares input data before the "real" pipeline runs — there is no "just setup," "just a quick test," or "just dummy data" exemption.** If a script exists and gets executed, it gets the wrapper — full stop, regardless of how small or preliminary it feels. A local file that never went through `ln.track()`/`ln.Artifact(...).save()` cannot be turned into a real input later; the lineage gap it leaves is permanent.

For any script that's written, its inputs should not be local; they must be lamindb artifacts. If an input is available locally, it must be uploaded to lamindb as an artifact before running the script and adding the run to lamin. If any script was run or code was generated within the session to create that input, this script or code should also be added to lamin.

Mentally note only the files you create or modify **directly, with no script involved** — those need manual attachment before finishing (see Step 3, and your harness's reference file for how to resolve your run).

Make sure you always do Step 3 at the end of the session, even if the user doesn't ask.

## Step 3 — End of session

User confirmation is not required. Always do Step 3. It has three parts that must happen in this order: attach output files you created directly (3a), clean up the superseded transform versions you created (3b), then close the session (3c). **Run the commands below exactly as shown — do not skip a part, and do not consider the task done until `lamin finish` has actually been run.** 3b is the one part that asks the user anything, because it is the one part that removes records; a declined or unanswered question there skips the cleanup and continues to 3c rather than stopping the session.

### Step 3a — Attach output files you created directly

If you created output files directly (no script involved), attach them first — see your harness's reference file for the exact command to resolve your run and attach files to it.

### Step 3b — Clean up the superseded transform versions you created

Iterating on a script leaves a trail behind: each time a tracked script's source code changes and it runs again, lamindb makes a **new** Transform version (same `stem_uid`, incremented `uid` suffix) and demotes the previous one to `is_latest=False` — notebooks bump a version on every re-run. Only the newest version is what you actually delivered, so a script you rewrote five times leaves four dead versions, each with its own Run, and the user's registry turns into a junk drawer. Clear out the ones **you** superseded in this session.

Cleaning up is two moves and the order matters: first repair the lineage of any artifact still attributed to a run you're about to remove, then trash the superseded versions along with the outputs that only they produced.

The first move is needed because lamindb deduplicates artifacts by hash. When a later version of your script writes the same content again, `Artifact()` returns the **existing** record rather than creating a new one, leaves `artifact.run` pointing at the run that created it *first*, and appends the later run to `artifact.recreating_runs`. So the artifact you just "re-created" is still attributed to your earliest, throwaway iteration. Move `artifact.run` forward to the newest surviving run that re-created it before that early run disappears — otherwise you trash the run a live artifact is attributed to and leave the lineage worse than you found it.

**Nothing here happens without the user's approval.** This is the one part of Step 3 that removes things, so it runs as plan, then ask, then apply. Run the command below **before** `lamin finish`, which deletes the run-uid state file it needs. The only per-harness part is that state file's path — take it from your harness's reference file and substitute it below; run the rest exactly as shown, no `cd` needed:

```bash
uv run --with lamindb python -c "
import os
import lamindb as ln
from pathlib import Path

apply = os.environ.get('LAMIN_CLEANUP_APPLY') == '1'
run = ln.Run.get(uid=Path('<your harness run-uid state file>').read_text().strip())
child_transform_ids = list(ln.Run.filter(initiated_by_run=run).values_list('transform_id', flat=True))
superseded = [
    transform
    for transform in ln.Transform.filter(id__in=child_transform_ids, is_latest=False)
    if transform.created_at >= run.started_at  # older versions were not written by you
]
superseded_run_ids = list(ln.Run.filter(transform__in=superseded).values_list('id', flat=True))
for transform in superseded:
    repoints, blockers = [], []
    for artifact in ln.Artifact.filter(run__transform_id=transform.id, is_latest=True):
        surviving_run = artifact.recreating_runs.exclude(id__in=superseded_run_ids).order_by('-started_at').first()
        if surviving_run is None:
            blockers.append(artifact)
        else:
            repoints.append((artifact, surviving_run))
    if blockers:
        print(f'keep {transform.uid} ({transform.key}): {len(blockers)} current artifact(s) still attributed to its runs')
        continue
    for artifact, surviving_run in repoints:
        print(f're-point {artifact.uid} ({artifact.key}).run -> surviving run {surviving_run.uid}')
        if apply:
            ln.Artifact.objects.filter(id=artifact.id).update(run_id=surviving_run.id)
            artifact.recreating_runs.remove(surviving_run)
    for artifact in ln.Artifact.filter(run__transform_id=transform.id, is_latest=False):
        if artifact.input_of_runs.exists() or artifact.collections.exists():
            print(f'keep artifact {artifact.uid}: still referenced elsewhere')
            continue
        print(f'trash superseded artifact {artifact.uid} ({artifact.key})')
        if apply:
            artifact.delete(storage=False)
    print(f'trash superseded {transform.uid} ({transform.key}) and its {ln.Run.filter(transform=transform).count()} run(s)')
    if apply:
        ln.Run.filter(transform=transform).delete()
        transform.delete()
print('--- applied' if apply else '--- plan only, nothing changed yet')
```

As written, that command **changes nothing** — every mutation sits behind `if apply`, and `apply` is only true when `LAMIN_CLEANUP_APPLY=1` is set. So run it plain first to get the plan.

If it lists nothing to re-point or trash, you're done: say so briefly and move to Step 3c without asking anything. There is no point putting a confirmation prompt in front of a no-op.

Otherwise, show the user what it printed and ask a single yes/no question — "Clean up these superseded transform versions?". **If your harness has a dedicated clarifying-question or ask-user tool, you must use it**, exactly as in Step 1's tracking question; fall back to plain response text only if no such tool exists. Wait for a real answer. Then:

- **Approved** → re-run the *identical* command with the flag prepended, and nothing else changed: `LAMIN_CLEANUP_APPLY=1 uv run --with lamindb python -c "..."`.
- **Declined, or no clear answer** → skip the cleanup entirely and go straight to Step 3c. **A confirmation that never arrives must never strand the session.** Leaving some clutter behind costs nothing; leaving the run open with no report because you sat waiting on an answer is the one genuinely bad outcome of this step.

Run it twice rather than writing a separate read-only variant. Both passes execute the same selection over the same data, so the plan the user approves is the work that actually happens — a hand-written "preview" version drifts from the real one and starts describing deletions that differ from what lands.

Details that are easy to get wrong, so change nothing here:

- The re-point is a direct field update rather than `artifact.run = surviving_run; artifact.save()` on purpose. `Artifact.save()` carries the full storage path, upload, and validation machinery, none of which should fire when all you're correcting is one foreign key.
- `recreating_runs.remove(surviving_run)` preserves lamindb's own invariant that a run is either the creator (`artifact.run`) or a re-creator, never both — the same invariant `populate_subsequent_run` maintains.
- `surviving_run` must exclude the runs about to be trashed, or the artifact gets re-pointed at another doomed run and nothing is gained.
- Re-pointing happens only for transforms that are actually being trashed, which is why it sits after the `blockers` check rather than before it. `artifact.run` means "the run that first created this", and overriding that convention is justified only because the run in question is about to disappear. A transform that survives keeps its attribution untouched.
- Only artifacts that are the latest version of their family get re-pointed. An artifact that was itself superseded belongs with the run that made it; both are historical and go quiet together, which is what the second inner loop does — leave those artifacts live and you strand them in the UI pointing at a run in the trash, the same defect the re-point exists to prevent.
- `artifact.delete(storage=False)` trashes the record and deliberately leaves the stored file alone. The `storage=False` is not decoration: without it, `delete()` fast-fails with an `IntegrityError` on any artifact whose storage location is managed by a different instance. Never pass `storage=True` or `permanent=True` — a superseded *file* artifact owns its own object in storage, but for a folder artifact (`overwrite_versions=True`) every version shares one store, so deleting data here can take out versions you never looked at.
- The `input_of_runs` / `collections` check is what stops this from creating the very problem it set out to fix. An old version that something else still consumes stays live; trash it and you leave a live run or collection referencing a record in the trash.

Every deletion here moves records to the **trash** (`branch_id = -1`), which is what a bare `.delete()` does — they drop out of queries and the UI, and the user can `.restore()` any of them. **Never add `permanent=True` here.** Neither version history nor data is yours to destroy irreversibly on an automated step, and permanent deletion of a run would additionally fail outright while any Artifact still points at it, since `Artifact.run` is a protected foreign key.

Every filter is load-bearing and must stay exactly as written:

- Restricting to transforms that have a Run initiated by *your* run, and to those created after your run started, is what keeps this to versions **you** wrote in this session. A version a human authored, or one left behind by an earlier or someone else's session, is not yours to clean up — even though it is tempting to sweep those too, you cannot tell from the database whether it is abandoned or the state someone is deliberately working from.
- Splitting the current artifacts into `repoints` and `blockers` *before* touching anything is what lets one command serve as both plan and apply. A `blockers` entry is a current artifact that no surviving run re-created, so its attribution cannot be moved and the transform has to stay. Re-deriving that decision by re-querying after the re-point would make the plan pass and the apply pass disagree: on the plan pass nothing has moved yet, so it would report keeping transforms that the apply pass goes on to trash.
- Every `.filter()` call excludes already-trashed records, which keeps this non-interactive: `.delete()` on a record that is *already* in the trash prompts for confirmation and would hang.

**Never widen this into `ln.Transform.filter(is_latest=False).delete()`**, and never drop a filter to make the command "actually do something" — that trashes every version family on the instance, including work no agent ever touched. Printing nothing at all is a normal, successful result: it means you didn't supersede a script this session and there is nothing to clean up.

### Step 3c — Close the session

Run this exact command, as its own tool call (no `cd` needed — it resolves the dev-dir internally, the same as Step 1's `lamin track <agent>`):
```bash
lamin finish
```
Escalate to the fallback below only if this command errors (non-zero exit status) — under no other circumstance (not a lamindb warning, not wanting to double-check, not comparing against a local virtualenv version) should you run any additional command before or instead of accepting this result. A command error here usually means `lamin` is likely only installed in a project-local virtualenv rather than on `PATH`:
```bash
LAMIN_BIN=$(find . -maxdepth 6 -type f -name lamin 2>/dev/null | head -1)
[ -z "$LAMIN_BIN" ] && LAMIN_BIN=$(command -v lamin 2>/dev/null)
if [ -z "$LAMIN_BIN" ]; then
  echo "NOT_FOUND: lamin"
else
  "$LAMIN_BIN" finish
fi
```

This is the same command regardless of harness — it resolves whichever session is currently active on its own, renders the transcript as HTML, saves it as a report artifact, stamps all child scripts as session outputs (`Transform.run`), closes the run, and cleans up the local state files.

If Step 1 printed `NOT_FOUND`, there is no run to close — skip Step 3 entirely. If this command prints `NOT_FOUND`, or the binary itself errors (e.g. no lamindb instance connected): tell the user, skip the rest of tracking, and proceed with their actual task anyway — tracking infrastructure should never block the user's real request.

## Quick reference

* [Track Claude Code sessions](references/track_claude.md).
* [Track Copilot sessions](references/track_copilot.md).
* [Curate datasets](references/curate_datasets.md).
