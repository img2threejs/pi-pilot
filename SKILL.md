---
name: pi-pilot
description: Hand a task to the PI coding agent on this host and judge what comes back. Use whenever work should be done by PI rather than in-session, and whenever you need to know whether a PI run is still alive.
---

# Drive PI

PI does the work. You write the brief, verify the result, and decide. **Never trust PI's transcript
as evidence** — it reports what it believes it did, and that is the same failure mode every author
has.

## Never drive PI by hand

`pilot.py` starts PI runs and answers what is running. Use it. Every wrong call recorded below came
from an instrument improvised at the prompt, and the improvised instrument always failed towards
*nothing is running* — the answer that invites a second run spending the operator's credential on
work already in flight.

```sh
python3 pilot.py selftest                       # before believing any negative answer
python3 pilot.py status                         # what is live, measured
python3 pilot.py start --unit W3 --task review \
        --cwd /path/to/worktree --brief /path/to/brief.txt
python3 pilot.py wait <run-id> --lines 80       # run this as a tracked job
python3 pilot.py log  <run-id> --lines 80       # read the file, never a live pipe
python3 pilot.py stop <run-id>
```

`start` refuses a second run for the same `--unit`/`--task` while one is live. That refusal is the
feature; `--force` exists for when you mean it.

`status` decides liveness from `/proc/<pid>` and the kernel's start time, against a pid the launcher
recorded — never from a pattern you typed. `selftest` spends no model credit and proves `status` can
report **both** running and finished, using a probe that renames itself the way PI does.

## The failure modes, measured

These are the ones that have actually cost time. Each is a fact about PI or about the instrument, not
a style preference.

**`pi -p` waits for EOF on stdin before it does anything.** Run it from a shell whose stdin is a
socket or an open pipe — which is what an agent harness hands you — and it never starts: no API call,
no session file, no output. One run sat at 8h50m having spent 0.94 seconds of CPU. Held constant
except stdin: `</dev/null` exits 0 in seconds; `< <(sleep 300)` times out with nothing. `pilot.py`
passes `stdin=DEVNULL`. Typing the command by hand does not.

**PI renames its own process after `exec`.** The launcher is a shell wrapper that `exec`s node, and
node then sets its process title to `pi`. Whether your liveness check reads `comm` before or after
that rename is a race, so a run recorded as `node` and later reporting `pi` gets declared dead
seconds after it started while it goes on working. Compare pid and kernel start time; never `comm`.
A `sleep`-based selftest will not catch this — the probe has to rename itself too.

**`ps | grep <the path you launched>` matches nothing, running or not.** PI appears in `ps` as `pi` —
one word, no path, no `node`. A narrow filter built from the install location once hid an orphan
spending the operator's credential for over an hour.

**`| tail -N` on a live run holds everything until the pipe closes.** The output file is empty and
stays empty while PI works. That emptiness is `tail`, not PI. Redirect to a file; take a summary
after the run, never through it.

**`pi --list-models` exits 0 with no models configured.** Read its output, never its exit code.

**Before concluding from a negative check, prove the check can produce a positive.** "No output", "no
process", "no session" are claims about your instrument as much as about PI. A session log is
evidence that a *finished* run happened; it is not evidence about a run in flight.

## Invoking

- **`-p`** is non-interactive: it processes the brief, edits files, and exits. It writes without asking.
- **PI reads `AGENTS.md` and `CLAUDE.md` from the working directory by default.** That is how a
  project's contract reaches it, so run from inside the checkout and never pass `--no-context-files`.
- **Never pass `--no-approve`.** Its own help says *ignore project-local files for this run* — it is
  not a sandbox control, and it would make PI ignore the skills the work depends on.
- `--skill <dir>` forces a skill into the prompt. **Registration is not application** — PI's own
  documentation says models do not always read what is registered. Name the skill in the brief as well.
- Skills are discovered from `~/.pi/agent/skills/`, `~/.agents/skills/`, `.pi/skills/` and
  `.agents/skills/`, searched from `cwd` upward.
- **PI is slow.** Budget wall-clock generously; a trivial prompt can take minutes. There is no turn
  cap, only your timeout.
- Give each task its own session (`--session-id`) or none (`--no-session`), so one task's context
  never bleeds into the next. A reviewer that shares the author's session is not independent.

**PI runs with full permissions and no isolation.** On a single-user host that is the same access you
have, so it is not a new exposure — but it is why PI is never the thing that runs an untrusted
prompt.

## The brief

PI is not in your conversation. Everything it needs is in the brief or in files it can read.

- Point at the files, do not summarise them — say where they are.
- State the standard explicitly: **demonstrate, do not assert**, and **a probe is derived from the
  defect, never from the fix** — write it from the issue, run it against the unfixed code, watch it
  fail there first.
- **A passing check is never evidence that a rule holds.** Require PI to say what each check could
  fail for. This matters more with PI than with other models: its failure mode is not silence, it is
  confidence. Asked whether a rule held beyond the one case its test covered, it answered yes and
  cited the passing test — a test that manufactured the very condition it then reported as a refusal.
- Name what it must not touch.
- Ask it to report what it could not do. An honest gap is worth more than a quiet one.

## After PI returns

**Verify before you believe.** Run the build, run the tests, and reproduce at least one claim by
hand.

Then dispatch an independent review — a **fresh** PI session, never the session that wrote the code.

You judge the findings. Blocking ones go back with the finding quoted, not paraphrased.

## What PI is good at, and what it is not

Measured over several rounds on one codebase, as reviewer and as author:

**Good at close reading.** It found a TOCTOU race between a guard query and the read it guarded, a
newly added check that nothing proved could fail, and a credential leaking into durable evidence
through an error message — all by reading, and all missed by a stronger model reviewing the same
diff. It is fast and it runs on the operator's own credential, so a second opinion from it costs
almost nothing.

**Weak at doubting its own evidence.** It accepts a green check as proof, reproduces a defect and
then misjudges its severity, and will report an execution point that does not exist. As an author it
once changed a test and added the probe for it in the same commit, so the probe agreed with the fix
by construction.

So: use it where the answer can be measured and the task is to measure carefully. Do not use it where
the task is to distrust the evidence in front of it — and never as the only reviewer of something
that matters.

## When not to use PI

- Anything needing your conversation's context — an orchestration decision, a judgement between
  reviewers.
- A task whose whole content is a question. PI edits files; asking it a question spends minutes to
  get a paragraph.
- Work on a shared checkout. PI gets its own worktree, like every other agent.

## Cost

PI runs on the operator's own model credential. That makes it cheap to run and expensive to run
*wrongly* — a vague brief spends the operator's budget on work you will reject. Spend the effort in
the brief.
