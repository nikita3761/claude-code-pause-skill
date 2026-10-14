# /pause — a safe pause for Claude Code

**English** · [Русский](README.ru.md)

A [Claude Code](https://claude.com/claude-code) skill. When you type `/pause`, it stops a session that is
running many tasks, subagents and background processes at the nearest safe point. `/pause resume` picks the
work up where it stopped, even after `/compact` or a restart.

Use it when you need to:

- compact the context with `/compact` in the middle of a large task;
- close the laptop lid or put the computer to sleep;
- step away for lunch or until the evening.

## Why

Closing the laptop in the middle of work can go wrong in two ways:

- **Sleep** freezes processes and drops network connections. A database migration can break off halfway,
  a subagent can leave a half-written file, and an API request fails on wake.
- **Compaction** leaves processes running but erases the model's memory of what was started, by whom and
  why. Afterwards the session doesn't know which files belong to which task or what comes next.

The skill handles both the same way: **nothing may be running, and everything needed to continue must be
on disk**.

## Installation

The skill is a single `pause` folder containing `SKILL.md`. Put it in your personal skills directory.

```bash
git clone https://github.com/nikita3761/claude-code-pause-skill.git /tmp/claude-code-pause-skill
```

```bash
mkdir -p ~/.claude/skills && cp -R /tmp/claude-code-pause-skill/pause ~/.claude/skills/
```

To use it in one project only, put the folder in that project's `.claude/skills/`. New Claude Code sessions
pick the skill up automatically.

## Usage

| Command | What happens |
|---|---|
| `/pause` | Pause at the nearest safe point |
| `/pause closing the laptop in 10 minutes` | The same, within your deadline |
| `/pause resume` or "continue" | Continue where the work stopped |

It is safe to close the laptop or run `/compact` once the reply contains this line:

```
PAUSED - safe to sleep or /compact.
```

## How it works

### Pause

1. **Freeze.** No new tasks or subagents are started.
2. **The first reply** lists what is running, what must not be interrupted and roughly how long to wait.
3. **Inventory** of everything in flight: subagents, background commands, timers, the session's own
   unfinished edit, external operations.
4. **A draft checkpoint** is written to `.claude/PAUSE.md` right away with status `DRAINING`. If the lid
   closes early, the next session still knows where things stopped.
5. **Drain to a safe point**, with a rule for each kind of work:

   | In flight | What happens to it |
   |---|---|
   | Migration, deploy, push, publish | Never interrupted. No pause is declared until it finishes |
   | The session's own half-done edit | Brought to a consistent state, and no further |
   | Writing subagent | Finishes its current edit, reports, and stops |
   | Read-only subagent | Hands over its findings so far and stops |
   | Tests running on half-edited code | Stopped and marked "rerun" |
   | Dev server, watcher | Stopped last; the restart command is recorded |
   | Timers, crons, loops | Cancelled; their prompt and schedule are recorded |

6. **Quiescence check.** Agents and processes are listed again, and only then does the file's status
   change to `PAUSED`.
7. **The pause is declared** with a summary: what finished, what was stopped, what is uncommitted and where.

### Resume

1. `.claude/PAUSE.md` is read.
2. The record is checked against reality: branch, HEAD, changed files, worktrees, live processes, the state
   of external operations.
3. Any mismatch (a commit made by someone else, an unlisted file, a partly applied migration) is brought to
   you for a decision. External operations are never re-run without your consent.
4. Interrupted tasks are re-dispatched from the saved briefs, and work continues from the recorded next step.

## The checkpoint file `.claude/PAUSE.md`

The file is written so that a reader with no context can follow it. It is never committed. It contains:

- the goal and a link to the plan;
- finished tasks with commit hashes;
- git state: branch, HEAD, changed files grouped by task, worktrees;
- interrupted work, with a brief to re-dispatch each task;
- tasks not started yet;
- external operations and how their result was verified;
- stopped processes and timers, with commands to restore them;
- untrusted results that must be produced again;
- the next step and open questions.

## Limitations

- If the session has no tool to cancel a scheduled wakeup, the skill cannot cancel it. The wakeup is
  recorded in the file and may fire once, but it will not resume the work.
- The skill waits as long as needed for an external operation with no known end time. If you have to leave
  sooner, it says "not safe yet" and names the concrete risk.
- The skill assumes a git project. Without git, the repository section of the checkpoint stays empty.

## How it was tested

The skill was written with the TDD-for-skills method from
[superpowers](https://github.com/obra/superpowers): runs without the skill first, then with it.

- **Without the skill** (3 runs): agents left the migration alone, but each put the checkpoint file in a
  different place, and none cancelled timers or saved subagent briefs for re-dispatch.
- **With the skill** (3 pause runs and 1 resume run): the file path, its structure and the order of steps
  matched in every run. On resume, the agent stopped at the planted mismatches and did not re-run the
  migration without asking.

The runs were written scenarios, not live sessions with a real computer sleep. If you find a case where the
skill gets it wrong, please open an issue.

## License

[MIT](LICENSE)
