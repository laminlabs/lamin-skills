---
name: lamindb-track
description: MUST invoke this first, before responding to anything else, at the start of every single session in this project — no matter what the user's first message is (writing code, creating a file, answering a question, debugging). This project requires every session to be tracked in LaminDB as a Transform + Run. Skipping this step is a mistake even if the task looks unrelated to tracking, logging, or data — invoke unconditionally, then proceed with the user's actual request.
---

# LaminDB Session Tracking (Claude Code only)

## Concepts
- **Transform**: code, not data. `__claudecode__` is the one fixed Transform for the whole project representing the chat session itself (the "agent run"). **Any script you write to accomplish the user's task (`.py`/`.ipynb`/`.R`/`.Rmd`/`.qmd`) is its own separate Transform, tracked automatically the moment it runs** — never save a script as a plain Artifact. Getting this backwards destroys the lineage from script to the data it produced, which is the entire point of LaminDB.
- **Run**: an execution. The session gets one Run of `__claudecode__` (the **agent run**). Every script you write self-tracks its *own* Run the instant it executes, linked back to the agent run via `initiated_by_run` — see "Self-tracking scripts" below. You never construct the script's Transform/Run by hand from outside.
- **run.report**: rendered HTML of the transcript, saved as an Artifact and linked to the agent run.
- **Artifact**: data only — output files (csv, txt, images, fasta, etc.). A script's own `ln.Artifact(path).save()` calls (no `run=` needed) auto-attach to that script's own run. Only files you create directly, with no script involved, get attached to the agent run manually.

## Self-tracking scripts

Every script you write to do the user's actual task must instrument itself — this is what gives each output file a real lineage back to the exact code that produced it:

```python
import lamindb as ln
ln.track()
# ... the actual task ...
ln.Artifact("output.csv", description="...").save()  # no run= needed, auto-attaches
ln.finish()
```

When you **run** such a script, set `LAMIN_INITIATED_BY_RUN_UID` from the agent run's uid file in the same command, and go through `uv` so `lamindb` is importable — never plain `python3 script.py`:
```bash
LAMIN_INITIATED_BY_RUN_UID=$(cat .claude/.lamindb_run_uid) uv run --with lamindb python script.py
```
This links the script's self-created Run back to the agent run automatically — no need to pass the uid into the script's own source, which would pollute its hash/content for no reason.

## Step 0 — Use the right Python

No local venv has `lamindb`. Always go through `uv`, which fetches it on the fly:
```
uv run --with lamindb python -c "..."
```
Bare `python`/`python3` hits the system interpreter → `ModuleNotFoundError`.

## Step 1 — Start of session (before the user's actual task)

```python
import os, lamindb as ln
from pathlib import Path
from datetime import datetime, timezone

transform = ln.Transform.filter(key="__claudecode__").first()
if transform is None:
    transform = ln.Transform(key="__claudecode__", kind="pipeline", description="All Claude Code sessions in this project")
    transform.save()
    print("created transform:", transform.uid)
else:
    print("using existing transform:", transform.uid)

run = ln.Run(transform)
run.started_at = datetime.now(timezone.utc)
run.description = "<one sentence describing this session's task>"
run.save()
Path(".claude").mkdir(exist_ok=True)
Path(".claude/.lamindb_run_uid").write_text(run.uid)
Path(".claude/.lamindb_transcript_path").write_text(
    str(Path.home() / ".claude" / "projects" / os.getcwd().replace("/", "-") / f'{os.environ.get("CLAUDE_CODE_SESSION_ID")}.jsonl')
)
print(run.uid)
```
The session id/cwd are known right now and never change — persist the transcript path too, so Step 3 never needs to re-derive or check anything. Read both files back in Step 3; never retype/recall a uid.

## Step 2 — During the session

Mentally track every file you **create or significantly modify** (not files you only read). Make sure you always save the lamindb session (do step 3) even if user doesn't ask.

Any script you write to do the task gets the `ln.track()`/`ln.finish()` instrumentation from "Self-tracking scripts" above, and gets run with `LAMIN_INITIATED_BY_RUN_UID` set — this already handles lineage for anything a script produces. Separately note only the files you create or modify **directly, with no script involved** — those still need manual attachment in Step 3.

## Step 3 — End of session: build the report, close the run

User confirmation is not required. Always do Step 3.

**Exactly 1 Bash call — one script, nothing else.** No `echo`/`ls`/session-id checks: the uid and transcript path are already sitting in the two files Step 1 wrote. Reading a file is not worth a separate Bash call.

In that one script, in order:

1. **Load the run** — uid from `.claude/.lamindb_run_uid`, then `ln.Run.get(uid=...)`.
2. **Load the transcript path** — from `.claude/.lamindb_transcript_path` (written in Step 1).
3. **Parse line by line** (`json.loads`, skip unparseable lines). Keep only `message.role` in `{"user", "assistant"}`. Drop three kinds of bookkeeping entries entirely — never shown in the report: the Step 1 setup call, this Step 3 call itself, and any entry whose text starts with "Base directory for this skill:" (the skill's own instructions, injected when this skill loads — not the user's task).

   **`message.content` is either a plain string or a list of blocks — check for "Base directory for this skill:" in both shapes**, not just one. If `content` is a string, check the string itself. If it's a list, check the `text` field of its first `text`-type block. Missing either shape is exactly how this has leaked into the report before.

   **The Step 1/Step 3 calls live inside `tool_use` blocks, not assistant prose** — they're Bash commands, so the actual script text is in `input["command"]`, not in any `text`/`thinking` block. Check `input["command"]` itself for `ln.Transform(` / `ln.Run(transform)` (Step 1) or `ln.Run.get(uid=` combined with report-building (Step 3), and if it matches, drop that whole `tool_use` block *and* its paired `tool_result` (the output of running it) — not just adjacent text. Checking only surrounding text blocks misses these entirely, since the command itself never appears as plain text.
4. **Per content block**, by `type`: `text` → render prose. `thinking` → if the `thinking` field is non-empty, render collapsed (`<details>`); if it's empty (Claude Code sometimes stores only an encrypted `signature` with no plaintext), skip the block entirely — don't render an empty collapsible section. `tool_use` → if `name=="Bash"` show `input["command"]` raw, else pretty-print `input` as JSON; label with tool name. `tool_result` → join `content` text and render as output. Anything else → skip silently.
5. **Render one HTML page**, one block per turn, transcript order. Rules: `html.escape` all transcript text (untrusted). Distinguish user/assistant visually, and tool calls/results from prose. Light theme only (white/light background). Truncate any block past a few thousand chars.
6. **Save the report** — write HTML to a temp file, `ln.Artifact(path, description="Claude Code session transcript (rendered)", run=False).save()`, set `run.report = <artifact>`, delete the temp file.
7. **Save output files you created directly** — anything a self-tracking script produced is already correctly attached to *that script's own run* (see "Self-tracking scripts") and needs nothing further here. For files you created or modified directly, with no script involved: `ln.Artifact(path, description="<what it is>", run=run).save()`.
8. **Close the run** — `run.finished_at = datetime.now(timezone.utc)`; `run.save()`; delete `.claude/.lamindb_run_uid` and `.claude/.lamindb_transcript_path`.

If `lamindb`/`uv` isn't available or no instance is connected: skip all of this, tell the user, proceed with their task anyway.
