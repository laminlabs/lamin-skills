---
name: lamindb
description: "MUST invoke before responding to ANY message — including greetings, small talk, trivial math, anything that looks unrelated. Not a judgment call: never skip it, never decide a message is too trivial. Tracks this session in LaminDB as a Transform + Run. If you are about to respond without invoking this first, stop — that is already a mistake."
---

# LaminDB

Official LaminDB skill to write code with best practices, keeping up to date with new versions and features.

> Agent tracking requires a LaminDB build with worktree support from lamindb PR #3895 and matching `lamin-cli`/`lamindb-setup` versions; the CLI capability check in Step 1 is authoritative.

## Concepts

- **Transform**: code, not data. Each coding agent/harness has its own fixed Transform for the whole project representing that harness's sessions (the "agent run") — internally keyed `__claudecode__` for Claude Code and `__copilot__` for Copilot. **These are internal database identifiers only, never CLI arguments or command names** — the actual commands are plain `lamin track claude` and `lamin track copilot`, with no underscores; see your harness's reference file for the exact syntax rather than constructing a command from these keys. **Any script you write to accomplish the user's task (`.py`/`.ipynb`/`.R`/`.Rmd`/`.qmd`) is its own separate Transform, tracked automatically the moment it runs** — never save a script as a plain Artifact. Getting this backwards destroys the lineage from script to the data it produced, which is the entire point of LaminDB.
- **Run**: an execution. Each harness conversation gets one Run of your harness's fixed Transform (the **agent run**) per LaminDB instance. If the user sends a follow-up after you already ran `lamin finish`, run `lamin track <agent>` again: it resumes that same agent Run using the harness session ID instead of creating a duplicate, and the next `lamin finish` replaces its report with the complete conversation. Every script you write self-tracks its *own* Run the instant it executes, linked back to the agent run via `initiated_by_run` — see "Self-tracking scripts" below. You never construct the script's Transform/Run by hand from outside.
- **Two distinct link fields — do not conflate them**: `Run.initiated_by_run` (on the *Run* model) says "this execution was triggered by that other run" — it only exists once a script actually executes, and renders in its own "This run initiated" panel in the UI, not as an output. `Transform.run` (on the *Transform* model, separate field) says "this piece of code was authored/produced during that run" — it's what makes a script show up in the agent run's **Output** column (alongside artifacts), the way a plain output file does. `ln.track()` never sets `Transform.run` on its own — `lamin finish` stamps it explicitly at session close, so a script counts as a session output even if it's the *only* thing produced.
- **Never save a script as a plain Artifact.** Scripts (`.py`/`.ipynb`/`.R`/`.Rmd`/`.qmd`) must use `ln.track()` inside them. If you call `ln.Artifact("script.py").save()` you destroy the lineage between the code and the data it produced — that is the entire point of LaminDB and must never happen.
- **run.report**: rendered HTML of the transcript, saved as an Artifact and linked to the agent run.
- **Artifact**: data only — output files (csv, txt, images, fasta, etc.). A script's own `ln.Artifact(path).save()` calls (no `run=` needed) auto-attach to that script's own run. Only files you create directly, with no script involved, get attached to the agent run manually.
- **Always pass a meaningful `key`** when saving an Artifact — a stable, path-like name (e.g. `key="datasets/ataqseq_counts.csv"`), not left unset. Without a key, an Artifact can never be versioned against future updates to the same data. Only reuse the exact same key when a new save is genuinely a new version of that same dataset; use a distinct key otherwise, or unrelated saves will incorrectly get grouped into one version family.
- **Create artifacts from in-memory objects when possible**: Prefer `ln.Artifact.from_*()` over writing objects to disk and then calling `ln.Artifact(path).save()`. For example, use `ln.Artifact.from_anndata()` for `AnnData` and `ln.Artifact.from_dataframe()` for pandas `DataFrame`. **Do not use `df.to_csv(...)` (or similar) as an intermediate step if a matching `from_*` constructor exists**. Use path-based `ln.Artifact(path).save()` only when the output is genuinely file-native (e.g. image, FASTA, binary export, or a format without a `from_*` helper). If you're unsure whether a `from_*` helper exists for an object type, run `help(ln.Artifact)` to inspect supported constructors.
- **When a script needs data that an earlier script in this workflow already produced, retrieve it from LaminDB — never read the local file path directly.** Use `ln.Artifact.get(key="...")` (the same key it was saved under) followed by `.load()`; this is what registers that artifact as this run's input and forms the lineage edge between the two scripts. Reading the file straight off disk produces the same result but leaves LaminDB with no record that the two scripts are connected, silently breaking the workflow's lineage graph.

## Self-tracking scripts and notebooks

Every script or notebook you write to do the user's actual task must instrument itself — this is what gives each output file a real lineage back to the exact code that produced it:

```python
import lamindb as ln
ln.track()
# if this step consumes an earlier step's output, retrieve it — never read the local file path directly:
input_artifact = ln.Artifact.get(key="<key used when it was saved>")
df = input_artifact.load()  # registers it as this run's input, forming the lineage edge
# ... the actual task ... e.g. process_data(df)
# for in-memory objects, prefer from_* constructors (no to_csv/to_parquet/any other intermediate write)
artifact = ln.Artifact.from_dataframe(df, key="<meaningful/folder/path>/output.csv", description="...")
artifact.save()
# use path-based save only for genuinely file-native outputs:
ln.Artifact("figure.png", key="<meaningful/folder/path>/figure.png", description="...").save()
ln.finish()
```

For Jupyter notebooks, `ln.track()` cannot auto-detect the notebook path when run via `nbconvert`, and without `new_run=True` it loads the latest run instead of creating a new one. Fix both by passing `path` and `new_run=True` explicitly in the first cell:
```python
import lamindb as ln
ln.track(path="notebook.ipynb", new_run=True)
```

When you actually **run** such a script or notebook, always set `LAMIN_INITIATED_BY_RUN_UID` and use the same project environment and LaminDB installation that created the agent Run and selected its branch. If execution fails because a task dependency such as `pandas` is missing, add only that dependency to the same environment and retry with the same interpreter. **Never switch to a temporary `uv run --with lamindb` environment**: another LaminDB installation may resolve different branch state and save the child Run and artifacts to the wrong branch. If the matching interpreter cannot import LaminDB, stop and ask the user to repair that environment. This links the script's/notebook's self-created Run back to the agent run automatically. On its own this only sets `Run.initiated_by_run`; `lamin finish` separately stamps `Transform.run` so the file also shows up as a session **output**, not just an "initiated" run. How you find your own `LAMIN_INITIATED_BY_RUN_UID` value differs by coding agent — see your harness's reference file (linked below) for the exact command, and run it exactly as shown. Writing your own simplified tracking (e.g. calling `ln.track()` without `LAMIN_INITIATED_BY_RUN_UID`, or skipping this entirely) breaks the lineage back to the agent run and defeats the point of tracking at all.

## Step 1 — Start of session (before the user's actual task)

First, resolve whether this instance has a configured development directory. In worktree mode this is the parent of all branch directories.
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
If the output is the literal string `None`, stop before doing the user's task and say: **"Configure a dev-dir for agent tracking."** Show `lamin settings dev-dir set <path>` and wait for the user. On their next message, rerun the check. Otherwise, remember the printed path exactly as the base dev-dir. Don't rely on a shell variable to carry it forward — each tool call may run in a fresh subprocess, so type the literal path again when needed.

Next, read the current worktree setting:
```bash
lamin settings worktree get
```
Use the same `LAMIN_BIN` fallback pattern above only if this command errors. If both attempts fail because the installed CLI does not support worktree settings, stop and tell the user that agent tracking requires a newer LaminDB installation.

Determine which coding agent you are running as and follow the matching file under Quick reference below.

If the result is `false`, stop before doing the user's task and respond with exactly this sentence and nothing else: **"Switch on worktree mode for agent tracking."** Do not show a command, explanation, interactive question, or options, and do not change the setting yourself. The CLI handles migration of an existing manual dev-dir into its branch directory. On the user's next message, rerun both the dev-dir and worktree checks before continuing.

When worktree mode is enabled, ask one blocking interactive question using this exact sentence: **"Track this session in LaminDB?"** Use exactly these two labels, in this order, without descriptions or recommendation text:

1. **Track**
2. **Do not track**

**If your harness has a dedicated clarifying-question or ask-user tool, you must use it.** Only ask directly in response text if no such tool exists. Stop and wait for the user's actual selection; do not assume one or continue in the same turn. Show this dialogue once at the start and never repeat it on a normal follow-up.

If the user selects **Do not track**, do not create or switch a branch, run `lamin track`, or attempt Step 2/3 for the rest of the conversation. Continue the actual task normally and leave all LaminDB settings unchanged.

### Resolve the session working directory after Track

Compare the original working directory with the base dev-dir:

- **At the base dev-dir:** create a concise branch name in the form `<meaningful-task-slug>-<session-id-suffix>`. The slug must describe the user's actual task; never use a generic or timestamp-only name. Derive the suffix from the harness session ID as specified in its reference file, but do not print the session ID. Use only letters, digits, hyphens, or underscores, and never `/`. From the base dev-dir, run `lamin switch -c <branch-name>`, require success, and verify that `<base-dev-dir>/<branch-name>` exists. That branch directory becomes the session working directory.
- **Inside a child directory of the base dev-dir:** resolve the active branch root with the matching project interpreter:
  ```bash
  <matching-python-executable> -c "from lamindb_setup import settings; from lamindb_setup.core._settings_store import local_current_branch_file; root = settings.effective_dev_dir; assert local_current_branch_file(root).exists(), 'not a configured branch directory'; print(root)"
  ```
  `<matching-python-executable>` means the Python executable from the exact environment that provides the `lamin` executable being used; for `/path/to/env/bin/lamin`, use `/path/to/env/bin/python`. The command must resolve to the existing branch directory containing the original working directory. Use that branch root as the session working directory and do not create or switch another branch.
- **Outside the base dev-dir, or in an invalid child directory:** stop and explain that tracking must start from the dev-dir or a configured branch directory. Do not guess a branch or silently change directories.

Never call `lamin settings worktree set` or `lamin settings set worktree` yourself. Worktree mode is a user-controlled prerequisite.

The session working directory is immutable after it is resolved. `lamin switch` cannot change the parent agent process's working directory. Therefore, **run every later LaminDB command and every task command from the session working directory**, using the execution tool's working-directory option when available or an explicit `cd "<session-working-dir>" &&` prefix otherwise. This includes `lamin track`, lineage verification, scripts and notebooks, direct-output attachment, tests, and `lamin finish`. Never run task commands from the base dev-dir after selecting an isolated worktree branch.

### Command hygiene

Keep every prescribed command free of diagnostic shell noise. Required working-directory setup (`cd` or the execution tool's working-directory option) and required environment configuration such as `LAMIN_SETTINGS_DIR` are allowed. Do not add any of the following:

- status headings or separators such as `echo "--- dev-dir ---"`;
- manual exit-code output such as `echo "exit: $?"` or `echo "switch exit: $?"` — rely on the execution tool's reported exit status;
- commands that print harness session IDs — use the environment variable silently when constructing branch names and state-file paths;
- convenience aliases such as `LAMIN=...` or `PYBIN=...` — invoke the required `lamin` or matching Python executable directly.

Run `lamin track` and `lamin finish` as standalone substantive commands: do not place another diagnostic or task command before or after either one in the same tool call.

For a follow-up prompt in the same harness conversation after Step 3 completed, do not ask again or create/switch another branch. Return to the same session working directory and run the same `lamin track <agent>` command before doing the follow-up work. The CLI uses the harness session ID to resume the existing agent Run and restores the active UID file needed for child-run lineage.

Each tracked mode starts with `lamin track <agent>`, which creates (or reuses) that harness's fixed Transform and opens a Run — see your reference file for the exact command and what it writes. **Run the exact command shown in your reference file from the session working directory as its own tool call — do not write your own tracking logic, add another command alongside it, or skip straight to the user's task.** If tracking isn't available (`lamin` not found, or the command errors — e.g. no lamindb instance connected), tell the user and proceed with their actual task untracked. Do not attempt Step 2/3 for the rest of the conversation because there is no Run to attach anything to. Do not undo a branch or worktree choice merely because tracking failed.

## Step 2 — During the session

Every script you write to do the task — the first one and every later one, on any message in this session — gets the `ln.track()`/`ln.finish()` instrumentation from "Self-tracking scripts" above, and gets run with `LAMIN_INITIATED_BY_RUN_UID` set per your harness's reference file — this already handles lineage for anything a script produces. Having done Step 1 once already does not exempt a later script from this wrapper. **This applies even to a script that just generates, fetches, or prepares input data before the "real" pipeline runs — there is no "just setup," "just a quick test," or "just dummy data" exemption.** If a script exists and gets executed, it gets the wrapper — full stop, regardless of how small or preliminary it feels. A local file that never went through `ln.track()`/`ln.Artifact(...).save()` cannot be turned into a real input later; the lineage gap it leaves is permanent.

For any script that's written, its inputs should not be local; they must be lamindb artifacts. If an input is available locally, it must be uploaded to lamindb as an artifact before running the script and adding the run to lamin. If any script was run or code was generated within the session to create that input, this script or code should also be added to lamin.

For EVERY python script that you plan to run, whether it was created/modified by you in the session or already existed, whether its run directly or indirectly, we want to verify that it insures proper lineage in lamindb before running it — if it does not, we will need to modify it to do so before running it. User confirmation is not required before doing this. This needs to be run without fail. If you can't run this, quit the session. **Hard gate: never run a script unless the most recent `verify_lineage(script_path)` result for that exact file prints `Lineage is fully tracked`. If it prints any `Missing lineage: ...`, do not run the script; modify it and repeat `verify_lineage` until it passes based on the 'Lineage remediation retry policy' section below. Never stop after the first failed fix attempt unless blocked by tool/runtime failure; you must complete up to 2 remediation iterations first.** 

You can verify lineage by using the code below. The `script_path` variable should be set to the path of the script you are verifying. Run it with the same interpreter and LaminDB installation that will execute the script. A missing unrelated task dependency is not a reason to switch LaminDB environments.
```python
from lamindb.core import verify_lineage
result = verify_lineage(script_path)

if result.is_fully_tracked:
  print(f"Lineage is fully tracked")
else:
  print(f"Missing lineage: {result.missing_lineage}")

```

Mentally note only the files you create or modify **directly, with no script involved** — those need manual attachment before finishing (see Step 3, and your harness's reference file for how to resolve your run).

After a self-tracking script or notebook exits successfully, accept that result. Do not query or load its Transform, Run, or artifacts merely to reconfirm that they were saved, and do not search the filesystem for tracking state. Perform additional inspection only when the execution itself reports an error or when verification is part of the user's actual task rather than a tracking check.

Make sure you always do Step 3 at the end of the session, even if the user doesn't ask.

### Lineage remediation retry policy (mandatory)

When `verify_lineage(script_path)` fails for a script that is intended to run:

1. Attempt to fix the script and rerun `verify_lineage(script_path)`.
2. If it still fails, attempt one more fix and rerun `verify_lineage(script_path)` again.
3. Maximum remediation attempts: **2**.

After 2 failed remediation attempts:

- **Do not run the script.**
- Ask the user for guidance or a manual fix using the interactive ask-user tool (when available).
- Report both failed verify outputs and the exact remaining `missing_lineage` items.

Hard gate remains: only run when the **most recent** verify result for that exact file is `Lineage is fully tracked`.

### Lineage remediation guardrails

When fixing a script after `verify_lineage(script_path)` reports missing lineage, preserve script behavior and only add lineage tracking.

Non-negotiable rule:

- **Do not delete, comment out, or bypass file/folder path usage just to make verification pass. Only do it if overall script behavior can be preserved.**

Allowed direction:

- Add or adjust lineage instrumentation (`ln.track`, `ln.finish`, `ln.Artifact.get(...).load()`, `ln.Artifact(...).save()` or `ln.Artifact.from_*().save()`). Run `help(ln.Artifact)` for help with finding other methods for tracking artifacts in lamindb.

If lineage cannot be fixed without changing what the script does, stop and ask the user for guidance.



## Step 3 — End of session

User confirmation is not required. Always do Step 3. **Run the commands below exactly as shown — do not skip this step, and do not consider the task done until `lamin finish` has actually been run.**

If you created output files directly (no script involved), attach them first — see your harness's reference file for the exact command to resolve your run and attach files to it.

Then close the session — run this exact command from the immutable session working directory, as its own tool call:
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

This is the same command regardless of harness — it resolves whichever session is currently active on its own, renders the transcript as HTML, saves it as a report artifact, stamps all child scripts as session outputs (`Transform.run`), and closes the active tracking cycle. If the same harness conversation receives a later follow-up, `lamin track <agent>` resumes this Run and the next finish replaces its report and cumulative metrics with the complete conversation rather than creating a duplicate Run.

If Step 1 printed `NOT_FOUND`, there is no run to close — skip Step 3 entirely. If this command prints `NOT_FOUND`, or the binary itself errors (e.g. no lamindb instance connected): tell the user, skip the rest of tracking, and proceed with their actual task anyway — tracking infrastructure should never block the user's real request.

## Quick reference

* [Track Claude Code sessions](references/track_claude.md).
* [Track Copilot sessions](references/track_copilot.md).
* [Curate datasets](references/curate_datasets.md).
