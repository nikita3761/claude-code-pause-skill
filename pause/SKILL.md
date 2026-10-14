---
name: pause
description: Use when the user types /pause or asks to pause, suspend, hold or freeze a running session with multiple tasks, subagents or background processes - before /compact, closing the laptop, putting the computer to sleep, or a break - and when the user types /pause resume or asks to continue, resume or unpause work that was paused this way.
---

# Pause

## Overview

A pause is safe when **nothing is running and everything needed to continue is on disk**. Sleep suspends
processes and drops network connections, so anything mid-request fails on wake. Compaction keeps processes
alive but erases your memory of them. Both are survived the same way: drain, write the checkpoint, verify,
then declare.

`/pause resume`, or "continue" / "продолжай" while `.claude/PAUSE.md` says `PAUSED` or `DRAINING` → go to **Resume**.

## Pause

1. **Freeze.** Start no new task, subagent or work unit from this moment.
2. **First reply, before any drain action:** what is still running, what cannot be interrupted, your ETA,
   and "do not close the laptop until I say PAUSED".
3. **Inventory** everything in flight: subagents, background shells, timers (wakeups, crons, loops,
   monitors), your own unfinished edit, external operations. List them with tools (agent listing, `ps`,
   `lsof`, shell output files); memory only for what no tool can list.
4. **Write the draft checkpoint** to `.claude/PAUSE.md` in the project root (always this path) with
   `STATUS: DRAINING`, so an early lid-close still leaves a usable file. Untracked; never commit it.
5. **Drain** each item by its row in the table. Wait on completion notifications. Subagent deadline: the
   user's window minus 2 minutes, else 3 minutes.
6. **Verify quiescence:** re-list agents and processes; everything is finished, stopped, or named in the
   checkpoint as deliberately left. Write the final checkpoint with `git status`, HEAD and `STATUS: PAUSED`.
7. **Declare.** The final message opens with `PAUSED - safe to sleep or /compact.`, then: what
   finished, what was stopped, what is uncommitted and where, the checkpoint path, and `/pause resume`.
   Then end the turn. While the status is `DRAINING` or `PAUSED`, anything other than a resume request -
   a user question, a timer firing - gets a one-line answer and no work.

| In flight | Drain |
|---|---|
| Irreversible or external operation (migration, deploy, push, publish, payment) | Never stop it. Wait for exit and verify the result. No PAUSED line until it exits; if the user must leave sooner, say "not safe yet" with the concrete risk. |
| Your own half-done edit | Bring it to the nearest consistent point (finish that unit or revert it). Nothing beyond it. Files a writing subagent shares: edit only after that subagent has stopped. |
| Writing subagent | Message it: finish the current edit so every file parses, start nothing new, report files / done / left. Stop it at the deadline and record "files may be half-written". |
| Read-only subagent | Ask for findings so far; stop it; mark "redo" if partial. |
| Test or build run | Tree was mid-edit during the run → stop it now, record "rerun". Clean tree → let it finish if it fits the window, record the result. |
| Dev server, watcher | Stop it last, after everything that uses it. Record the restart command. |
| Timer (wakeup, cron, loop, monitor) | Cancel it and record its prompt and schedule. No tool to cancel it → record it and tell the user it may fire once. Create no timer that fires after the PAUSED line. |

Commit policy does not change during a pause: commit only what the session's convention already commits.
Uncommitted work on disk survives; record which files belong to which task.

## Checkpoint: `.claude/PAUSE.md`

Self-sufficient for a reader with zero context. Every section is REQUIRED; write "none" rather than omit.

```markdown
STATUS: PAUSED            <!-- DRAINING | PAUSED | RESUMED -->
Paused: <date time>  Reason: <sleep | compact | break>
## Goal and plan         <!-- one line + path to the plan/spec -->
## Done                  <!-- tasks finished, with commit hashes -->
## Git                   <!-- branch, HEAD sha, dirty files grouped by owner/task, worktrees: path + branch -->
## Interrupted work      <!-- per item: what, state on disk, done / left, agent tier, and a re-dispatch brief:
                              plan reference + files that hold partial work + what is left + write scope -->
## Not started           <!-- remaining tasks, so every task of the plan has a status in this file -->
## External operations   <!-- result verified how; if unfinished: exact state and "ask before re-running" -->
## Stopped processes and timers   <!-- restart command or prompt + schedule for each -->
## Untrusted results     <!-- test/CI results taken on a mid-edit tree: rerun -->
## Next action           <!-- the single first step after resume, then the order after it -->
## Open decisions        <!-- questions waiting for the user -->
```

## Resume

1. Read `.claude/PAUSE.md`. Missing → say so, rebuild from git and the plan, confirm with the user.
   `DRAINING` → the pause was cut short: treat every listed item as possibly still running or half-done.
2. Verify reality against it: branch, HEAD, `git status`, worktrees, live agents and processes, the state
   of each external operation. Old agent IDs are dead unless the listing shows them alive.
3. Anything the checkpoint does not predict (moved HEAD, unlisted dirty file, missing worktree, stray
   process) or an unfinished external operation → report all of it in one message and ask. Until answered,
   run only work that cannot be affected by the open question; never re-run an external operation unasked.
4. Set `STATUS: RESUMED`. Restart the recorded processes and timers that the next step needs.
5. One-line status to the user, then continue from **Next action**, re-dispatching interrupted work from
   the recorded briefs.

## Common mistakes

- Saying "paused" while a subagent or shell is still running, or before re-listing them.
- Checkpoint in an ad-hoc path, so the post-compaction session cannot find it.
- Recording that an agent was stopped but not the brief needed to restart it.
- Killing a migration or deploy to meet the user's deadline.
- Dispatching "one more quick task" or a review during the drain.
- Trusting the checkpoint on resume without checking git and the external system.
