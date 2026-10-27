# bot-keepalive

A [Grok Bot](https://x.ai/bot) skill that keeps a bot's cloud computer awake while long jobs
run. Grok Bot suspends the whole computer whenever no bot turn is active, so background jobs freeze
too, `nohup` and `tmux` included. Routines can't fix this because the computer still freezes
between runs.

The idea: start jobs detached, register them in one folder, and run **one** low-effort watcher
subagent that sits in back-to-back blocking Shell waits, polling every 5 seconds, until every job
is done or a hard deadline passes. A bot turn waiting on a command counts as activity, so the
computer stays awake.

See [`skills/bot-keepalive/SKILL.md`](skills/bot-keepalive/SKILL.md).

## Install
Send your Grok Bot the link to `SKILL.md` and ask it to save that as a skill. Every bot on your
account can then use it.

## 中文简介
Grok Bot 在没有 bot 干活的时候会把整台云电脑挂起，后台长任务也会一起冻住。这个技能的做法是：任务脱离会话启动并登记，再开**一个**全局守护 subagent（low effort），连续在前台阻塞等待、每 5 秒检查一次，所有任务结束或到截止时间就退出。守护期间会持续消耗额度。

## License
MIT
