# pi-pilot

Launch and observe [PI](https://github.com/earendil-works/pi) runs from an orchestrating agent, and
a skill that tells the orchestrator how to brief PI and how to judge what comes back.

Two files:

| | |
|---|---|
| `pilot.py` | starts PI runs, records them, and answers *what is running* — no dependencies beyond the Python standard library |
| `SKILL.md` | an [Agent Skills](https://agentskills.io) skill for the orchestrating agent: how to brief PI, what its failure modes are, and where it is strong and weak |

## Why this exists

Driving another coding agent from inside a coding agent is mostly not a prompting problem. It is an
observation problem: you need to know whether the run you started is alive, and every ad-hoc way of
finding out is wrong in the same direction.

Each of these cost real time before it was written down:

**`pi -p` waits for EOF on stdin before it does anything.** From a shell whose stdin is a socket or
an open pipe — which is what an agent harness hands you — it never starts. No API call, no session
file, no output, no error. One run sat at **8h50m having spent 0.94 seconds of CPU**. Held constant
except stdin: `</dev/null` exits 0 in seconds, `< <(sleep 300)` times out with nothing.

**PI renames its own process after `exec`.** The launcher `exec`s node; node then sets its process
title to `pi`. Whether you read `/proc/<pid>/comm` before or after that rename is a race — so a
liveness check that compares the recorded name against the current one declares a healthy run dead
seconds after it starts. Compare **pid and kernel start time**, never the name.

**`ps | grep <install path>` matches nothing, running or not**, because PI appears as `pi` — one
word, no path, no `node`. A filter built from the install location once hid an orphan spending the
operator's credential for over an hour.

**`| tail -N` on a live run holds everything until the pipe closes.** The log looks empty while PI
works. That emptiness is `tail`.

**`pi --list-models` exits 0 with no models configured.** Read its output, not its exit code.

All of these fail towards *nothing is running* — the answer that invites you to start a second run
against the same target, spending the operator's credential twice.

## `pilot.py`

```
pilot.py start  --unit <id> --task <kind> --cwd <dir> --brief <file> [--model M] [--session S] [--force]
pilot.py status [--json]
pilot.py wait   <run-id> [--interval N] [--lines N]
pilot.py log    <run-id> [--lines N]
pilot.py stop   <run-id>
pilot.py selftest
```

- **`start`** launches PI detached with `stdin=DEVNULL` and stdout redirected to a file, never a pipe,
  and records pid, kernel start time, session id, cwd and brief path under
  `~/.local/state/pi-pilot/`. It **refuses** a second run for the same `--unit`/`--task` while one is
  live. That refusal is the feature.
- **`status`** decides liveness from `/proc/<pid>` plus the kernel's start time against the recorded
  pid. Start time alone defeats pid reuse; the process name is deliberately not consulted.
- **`wait`** blocks and prints progress. Run it as a job your harness tracks — a detached run is
  invisible to anyone who does not think to ask `status`, and that includes the human watching you.
- **`selftest`** spends no model credit and proves `status` can report **both** running and finished.
  Its probe **renames itself** via `prctl`, because the real subject does; it also asserts the rename
  actually happened, so the test cannot quietly become weaker than it looks. Run it before believing
  any negative answer.

Expects PI at `~/.local/bin/pi`. Override the state directory with `PI_PILOT_HOME`.

## `SKILL.md`

Drop it where your agent finds skills — `.claude/skills/pi-pilot/SKILL.md`,
`~/.pi/agent/skills/pi-pilot/SKILL.md`, `.agents/skills/pi-pilot/SKILL.md`, or any other
Agent-Skills location — and adjust the `pilot.py` paths in its examples to wherever you put the
script.

It carries the failure modes above, the invocation rules that matter (`--no-approve` is not a sandbox
control; registration of a skill is not application of it), how to write a brief PI will not waste
the operator's credential on, and an honest account of what PI is good and bad at.

The short version of that account, measured over several rounds on one codebase, as both reviewer and
author: **PI is a strong close reader and a weak self-doubter.** It found a TOCTOU race between a
guard query and the read it guarded, a newly added check that nothing proved could fail, and a
credential leaking into durable evidence through an error string — all by reading, all missed by a
stronger model on the same diff. It also accepts a green check as proof, misjudges severity on
defects it has itself reproduced, and once changed a test and added its probe in the same commit, so
the probe agreed with the fix by construction.

Use it where the answer can be measured and the job is to measure carefully. Do not use it where the
job is to distrust the evidence in front of it, and never as the only reviewer of something that
matters.

## Provenance

Extracted from a working orchestration setup where an Opus session plans and decides, Sonnet
subagents implement, and PI and Opus review in parallel. Everything stated as measured here was
measured on that host — Docker CE 29.8.0, Node 24, PI 0.85.1 — and the numbers are from the runs that
produced the lesson, not from an estimate.

MIT.
