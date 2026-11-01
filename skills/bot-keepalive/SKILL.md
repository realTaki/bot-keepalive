---
name: bot-keepalive
description: Use this when a long-running job (experiment, training run, batch script) has to keep running on a Grok Bot's cloud computer, which is suspended whenever no bot turn is active. One low-effort watcher subagent keeps the whole computer awake until every registered job finishes.
---

# Keep a Grok Bot's cloud computer awake during long jobs

## Why
When no bot is working, Grok Bot suspends the whole cloud computer. Everything on it freezes,
including processes started with `nohup` or in `tmux`. `uptime` does not reset, but a process's
elapsed time falls behind the wall clock. A routine can't keep it awake, because routines run at
most every 5 minutes and the computer still freezes between runs.

What does work: the computer stays awake while a bot turn is blocked on a foreground Shell
command. So one background executor sits in back-to-back Shell waits until the work is done.

**50-minute limit:** the platform stops any background subagent after 50 minutes, with the
notice "A background task was stopped after 50 minutes so it would not hang." So the watcher runs
as a relay: each leg does at most 3 waits (about 39 minutes), exits cleanly, and the main session
starts the next leg.

**Cost:** the watcher uses quota for as long as it runs, and every relay leg wakes the main
session once. Only use it for jobs that really have to run continuously. A job that can resume
from a checkpoint may be fine just running slower. For jobs expected to run longer than 2 hours,
prefer scheduled routines or remote compute (section 5).

## 1. Start each job detached and register it
Start the job so it survives the bot's turn ending, then register it in one shared folder:

```bash
mkdir -p ~/.keepalive/jobs ~/.keepalive/ended
cd /path/to/job
setsid nohup bash -c '<your command>; touch DONE' > run.log 2>&1 < /dev/null &
echo "$! $(pwd)" > ~/.keepalive/jobs/<job-name>
```

Each registry file holds `<PID> <job dir>`. A job counts as finished once `DONE` exists in its
folder or its PID is gone.

## 2. Make the job suspension-safe
- Measure timeouts in run time, not wall-clock time. A monotonic clock or the process's own CPU
  time works. Otherwise a long freeze looks like a hang and kills the job.
- Checkpoint regularly so a crash or freeze costs little.

## 3. Run one watcher for the whole computer
All of a user's bots share the same cloud computer, so one watcher covers every job on it.
Before starting one, check `~/.keepalive/watcher.lock`. If it exists and was updated in the last
15 minutes, a watcher is already running: register the new job and stop there. The running
watcher picks it up.

Otherwise, start a background executor subagent at low effort. Tell it the hard deadline (an
absolute time), that it only reads and never modifies anything, and to repeat this loop **at
most 3 times**:

1. Run one foreground Shell command with `block_until_ms` set to `760000` (the per-call limit is
   13 minutes). The command refreshes the lock, polls every 5 seconds, and returns early once
   nothing is running:
   ```bash
   alive() { for f in ~/.keepalive/jobs/*; do [ -e "$f" ] || continue; read pid dir < "$f"
     if [ -f "$dir/DONE" ] || ! kill -0 "$pid" 2>/dev/null; then mv "$f" ~/.keepalive/ended/; else echo 1; fi
   done; }
   for i in $(seq 1 144); do touch ~/.keepalive/watcher.lock; [ -z "$(alive)" ] && break; sleep 5; done
   date; ls ~/.keepalive/jobs ~/.keepalive/ended
   ```
2. If jobs are still running, the deadline hasn't passed, and fewer than 3 waits are done,
   start the next wait right away. Don't pause between waits or write commentary, since every
   token costs quota.
3. After 3 waits, exit **without** deleting the lock and reply with one line: `RELAY` plus the
   registry files still running.
4. When no jobs are left or the deadline passes, delete `~/.keepalive/watcher.lock` and report on
   each job: whether it finished (`DONE`), disappeared, or was still running at the deadline,
   plus its end time and the last few lines of its `run.log`.

**Relay in the main session:** when a leg replies `RELAY`, start the next leg at once with the
same prompt and deadline. The work is finished only when a leg sends the final report.

## 4. Report progress separately
For progress updates, use a separate hourly routine. The watcher stays silent until it exits.
Once a watcher is running, drop any frequent "wake-up" routines, since they only add cost.

## 5. Remote kernels and quota resets: use routines, not a watcher
Jobs on remote compute (a Kaggle or Colab kernel) keep running while the cloud computer sleeps,
so they need no watcher.
- After submitting, create a **one-off routine** at the expected finish time that checks the
  status. If it's done, it fetches the results, reports, and deletes the fallback routine.
- Also create an **hourly fallback routine** that checks the status and deletes itself once the
  job completes, errors, or runs past an agreed maximum time (reporting each case).
- A small status helper keeps routines cheap. For example, a script that prints one line of JSON
  with a `bucket` of COMPLETE, RUNNING, QUEUED, ERROR or OTHER and exits 0 (complete),
  1 (running or queued), 2 (error) or 3 (other or CLI failure).
- If you do watch a remote kernel from the computer for a while, give the guard script the same
  relay rule: after `MAX_ROUNDS` rounds (default 3) it exits with code 7 (HANDOFF, the same as
  `RELAY`), keeping its round count in a file in the output folder and resetting it on handoff or
  a final state. A single-check mode (for example `ROUND_SECS=0`) that doesn't count rounds is
  what the routines call.
- Waiting for a quota reset (such as a weekly GPU quota) works the same way: a one-off routine at
  the reset time starts the next step.

## Notes
- If the watcher stalls or repeats itself, stop it and start a fresh one. The jobs keep running
  either way.
- This is a temporary workaround. Remove it once Grok Bot stops suspending computers with jobs
  still running.
