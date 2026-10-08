---
name: herd-orchestrator
description: "Orchestrate and coordinate work across multiple Herdr coding agents from the current pane. Use when the user asks to orchestrate / coordinate / plan-and-delegate across several herdr agents, open N agents, split work between workers, or run a ping-pong handshake and report-back protocol. Requires HERDR_ENV=1."
---

# Herd Orchestrator

Turn the current session into a **coordinator** that plans with the user and
delegates execution to sibling panes running other Herdr agents. The
coordinator does not search the repo or implement anything itself — that
keeps its context small and free to talk to the user. Repo search, reading,
editing, and testing all go to workers.

## Roles

- **Coordinator** — this session. Talks to the user, writes task specs,
  dispatches, verifies reports, summarizes. Never implements or searches.
- **Workers** — sibling panes/agents. Do the actual explore, edit, and test
  work and report back themselves.

Reuse the same worker for similar tasks (its context carries over). A good
default split is **one explore/review worker + one implement/test worker**,
but confirm the split, the goal, and whether workers may commit with the user
before dispatching.

## Preflight

1. Confirm this is a Herdr-managed pane; otherwise stop:

   ```bash
   test "${HERDR_ENV:-}" = 1
   ```

   The coordinator's pane is `$HERDR_PANE_ID`, workspace `$HERDR_WORKSPACE_ID`.

2. Discover existing workers — filter `agent list` to this workspace and
   exclude the coordinator:

   ```bash
   herdr agent list
   ```

   Report to the user how many workers are already open.

3. If the user asked for **N agents**, open only the missing ones. For each:

   ```bash
   herdr pane layout --pane "$HERDR_PANE_ID"
   # wide pane -> right; tall/narrow pane -> down; avoid many same-direction splits
   herdr pane split --current --direction <right|down> --cwd "$PWD" --no-focus
   # read the new pane id from .result.pane.pane_id
   herdr agent start <name> --kind <kind> --pane <new-pane-id>
   ```

   Default `--kind` = same kind as the coordinator (e.g. `opencode`). Use
   unique short names like `worker-a`, `worker-b`. **Never close panes you did
   not create.**

## Messaging protocol (critical)

- **Coordinator → worker:** `herdr agent prompt <worker> "<task>"` with **no
  `--wait`** — fire-and-forget. After dispatching, **end the turn.**
- **Never poll or wait on workers.** No `--wait`, no `agent wait` loops, no
  `agent read` polling. The user may ask questions meanwhile, and another
  worker may be trying to message the coordinator while it is blocked.
- **Worker → coordinator:** each worker reports by itself running
  `herdr agent prompt <coordinator-pane> "[worker X / <pane>] <concise result>"`
  with no `--wait`. Every task message you send must restate this reply
  instruction **including the coordinator's pane id**.
- **Long results:** the worker writes a file under `/tmp/opencode/` (or a tmp
  dir) and sends only the path.
- **Long task specs:** write the spec to a tmp file and send the worker the
  path (avoids shell-quoting bugs).
- **Busy target:** if `agent prompt` returns `agent_blocked`/busy, wait ~10s
  and retry once. Do not blindly resend.
- **Concatenated reports:** two workers' reports can arrive in one message, so
  require every report to start with its `[worker X / pane]` tag on its own
  line.
- **Known pitfall:** building messages with `printf "$MSG" a b c` duplicated
  text when the arg count did not match the `%s` count. Prefer plain string
  interpolation or a spec file.

## Ping-pong handshake

Run this after preflight/opening panes and **before any real work**.

Send each worker: who the coordinator is (its pane id), the worker's own
label/pane, the reply protocol, and the instruction to touch no files and just
reply. Then end the turn; when all PONGs arrive, confirm to the user with a
table of workers and ask what to work on.

**Handshake template:**

```
[coordinator -> worker-a]
You are worker-a in pane <worker-a-pane>. I am the coordinator in pane $HERDR_PANE_ID.
This is a handshake only: touch no files. When done, report back by running
(no --wait):
  herdr agent prompt $HERDR_PANE_ID "[worker-a / <worker-a-pane>] PONG - ready"
Just send me: [worker-a / <worker-a-pane>] PONG - ready
```

## Task lifecycle

1. Discuss the task with the user (goal, role split, file scope, whether
   workers may commit).
2. Write the task: scope, files, acceptance criteria, "do not commit unless
   told", and the reply instruction (coordinator pane id + `[worker X / pane]`
   tag).
3. Dispatch with `herdr agent prompt <worker> "<task>"` (no `--wait`).
4. End the turn.
5. On report: verify/summarize to the user, optionally send to the review
   worker, then move to the next task.

**Task template:**

```
[coordinator -> worker-b] Task: <one-line goal>.
Scope: <files / modules>. Acceptance: <criteria>.
Do not commit unless told.
When done, report back by running (no --wait):
  herdr agent prompt $HERDR_PANE_ID "[worker-b / <worker-b-pane>] <concise result>"
Keep the report to a few lines; for long results write a file under
/tmp/opencode/ and send only its path.
```
