# agent-keepalive

An agent skill that keeps a cloud VM awake while long jobs run, on platforms that suspend the
whole machine when no agent turn is active.

The idea: start jobs detached, register them in one folder, and run **one** low-effort watcher
subagent that sits in back-to-back blocking shell waits, polling every 5 seconds, until every job
is done or a hard deadline passes. A blocked agent turn counts as activity, so the machine isn't
suspended. Periodic wake-up schedules can't do this, because the machine still freezes between runs.

See [`skills/agent-keepalive/SKILL.md`](skills/agent-keepalive/SKILL.md).

## Install
Copy `skills/agent-keepalive/` into your agent's skills folder (any tool that reads `SKILL.md`
files with `name` / `description` frontmatter).

## 中文简介
有些 agent 平台在没有 agent 干活的时候会把整台云电脑挂起，后台长任务也会一起冻住。这个技能的做法是：任务脱离会话启动并登记，再开**一个**全局守护 subagent，连续在前台阻塞等待、每 5 秒检查一次，所有任务结束或到截止时间就退出。守护期间会持续消耗额度。

## License
MIT
