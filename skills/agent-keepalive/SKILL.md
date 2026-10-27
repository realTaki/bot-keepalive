---
name: agent-keepalive
description: Use this when a long-running job (experiment, training run, batch script) has to keep running on an agent's cloud VM that the platform suspends whenever no agent turn is active. One watcher subagent keeps the whole machine awake until every registered job finishes.
---

# Keep an agent VM awake during long jobs

## Why
Some agent platforms pause the whole VM when no agent is working. Everything freezes, including
`nohup`, `tmux`, and `systemd` processes. `uptime` does not reset, but a process's elapsed time
falls behind the wall clock. A scheduled wake-up (cron or routine) every few minutes doesn't fix
this, because the machine still freezes between runs.

What does work: the platform treats the machine as busy while an agent turn is blocked waiting
on a shell command. So one background subagent sits in foreground shell waits until the work
is done.

**Cost:** the watcher burns agent quota for as long as it runs. Only use it for jobs that really
have to run continuously. Jobs that can resume from a checkpoint may be fine just running slower.

## 1. Start each job detached and register it
Start the job so it survives the agent's turn ending, then register it in one shared folder:

```bash
mkdir -p ~/.keepalive/jobs ~/.keepalive/ended
cd /path/to/job
setsid nohup bash -c '<your command>; touch DONE' > run.log 2>&1 < /dev/null &
echo "$! $(pwd)" > ~/.keepalive/jobs/<job-name>
```

Each registry file holds `<PID> <job dir>`. A job counts as finished when `DONE` exists in its
folder or its PID is gone.

## 2. Make the job suspension-safe
- Measure timeouts in run time, not wall-clock time. A monotonic clock or the process's own CPU
  time works. Otherwise a long freeze looks like a hang and kills the job.
- Checkpoint regularly so a crash or freeze costs little.

## 3. Run one global watcher
Run one watcher for the whole machine, not one per job. Before starting a watcher, check whether
one is already running (for example a lock file `~/.keepalive/watcher.lock` holding its start
time). If one is, just register the new job and stop there. The running watcher picks it up.

Start the watcher as a background subagent at the lowest effort level. It only reads and never
modifies anything. Give it a hard deadline (an absolute time) and this loop:

1. Run one foreground shell command with the longest blocking wait the platform allows (for
   example about 12–13 minutes). The command polls every 5 seconds and returns early once
   nothing is running:
   ```bash
   alive() { for f in ~/.keepalive/jobs/*; do [ -e "$f" ] || continue; read pid dir < "$f"
     if [ -f "$dir/DONE" ] || ! kill -0 "$pid" 2>/dev/null; then mv "$f" ~/.keepalive/ended/; else echo 1; fi
   done; }
   for i in $(seq 1 150); do [ -z "$(alive)" ] && break; sleep 5; done
   date; ls ~/.keepalive/jobs ~/.keepalive/ended
   ```
   Set the loop count so that 5 s × count stays under the platform's blocking limit.
2. If jobs are still running and the deadline hasn't passed, start the next wait right away.
   Don't pause between waits or write commentary. Tokens are the cost.
3. When no jobs are left or the deadline passes, remove the lock and report, for each job:
   whether it finished (`DONE`), disappeared, or was still running at the deadline, plus its end
   time and the last few lines of `run.log`.

## 4. Progress reports are separate
If someone wants progress updates, use a separate low-frequency schedule (for example hourly).
The watcher stays silent until it exits. Once a watcher is running, drop any frequent
"wake-up" schedules, since they only add cost.

## Notes
- If the watcher stalls or repeats itself, stop it and start a fresh one. The jobs keep running
  either way.
- This is a workaround. Remove it once the platform stops suspending busy machines.
