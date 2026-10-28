# bot-keepalive

**Keep your [Grok Bot](https://x.ai/bot) cloud computer awake while long jobs run, for the fewest tokens possible.**

One low-effort subagent, one tool call every ~13 minutes, about **1.8M tokens a day** in our own use. No more jobs dying every time the computer dozes off.

[中文说明](#中文说明)

---

## The problem

Grok Bot puts its cloud computer to sleep after about 5 minutes of "idle". But idle is defined loosely: if no agent is in a conversation and you don't have the remote desktop open, the computer counts as idle, **even if it's running a pile of background jobs**. The whole machine freezes, `nohup` and `tmux` included.

That breaks anything long-running: trading bots, monitors, server-fleet management, ML training and scheduling.

**Why routines don't fix it**

- The shortest routine interval is 5 minutes, the same as the sleep timer.
- Routines don't fire on time. Daily ones can drift 15 to 30 minutes at busy hours.
- So even with a 5-minute routine, the computer keeps falling asleep and freezing for minutes at a time. Jobs die, and restarting them costs more tokens.
- Each routine run is a fresh bot that re-reads the task context and takes several turns before it finishes.

## The fix

A bot turn that is **waiting on a tool call counts as active**, and a single Shell call can block for up to 13 minutes. So:

1. Start your jobs detached (`setsid nohup …`) and register each one in `~/.keepalive/jobs/`.
2. Run **one** global watcher subagent at **low effort**, outside your main chat so it never blocks you.
3. Each turn, the watcher makes one Shell call: a bash loop that checks every 5 seconds whether the jobs are done, for about 12 minutes, then returns.
4. Repeat until every job is done or a hard deadline passes. A lock file makes sure only one watcher runs per account, no matter how many bots start jobs.

The 5-second polling happens inside bash, not in the model. The agent only wakes once per ~13 minutes, for one tool call.

## Cost

| | Routine every 5 min | bot-keepalive |
|---|---|---|
| Keeps the computer awake | No, still freezes between runs | Yes |
| Timing | Drifts, up to 15 to 30 min | Steady, ~13 min per turn |
| Work per wake-up | New bot, re-reads context, several turns | One turn, one tool call |
| Model effort | Whatever the routine uses | Low |
| Measured cost | More | ~1.8M tokens/day |

For a paid plan, that's small change.

## Install

Send your Grok Bot this link and ask it to save it as a skill:

```
https://github.com/realTaki/bot-keepalive/blob/main/skills/bot-keepalive/SKILL.md
```

Every bot on your account can then use it. Next time you start a long job, tell the bot to "keep the computer awake with bot-keepalive until it finishes".

The full method, loop and rules are in [`SKILL.md`](skills/bot-keepalive/SKILL.md).

---

## 中文说明

**Grok Bot 云电脑防休眠，最省 Token 的做法。** 一个 low effort 的 subagent，大约每 13 分钟一次 tool call，我们自己实测一天额外开销约 180 万 Token 额度。

**问题。** Grok Bot 闲置约 5 分钟就休眠，但“闲置”定义得太宽：没有 agent 在对话、你也没开远程桌面，就算闲置，哪怕后台开着一堆任务在跑。整台机器冻住，`nohup`、`tmux` 也一起停。打狗、监控、管服务器集群、ML 任务监控和调度都很难用。

**Routine 为什么不行。** Routine 最短 5 分钟一次，跟休眠时间一样；而且不准时，高峰期误差能到 15 到 30 分钟。所以就算 5 分钟一次，电脑还是反复休眠，任务挂掉，修复重启还要额外花 Token。每次 Routine 都是一个新 bot，要从头读上下文、跑好几个 turn 才结束。

**解法。** 等待 tool call 返回的 turn 算活跃，单次 Shell 调用最长能阻塞 13 分钟。所以长任务脱离会话启动并登记到 `~/.keepalive/jobs/`，再开**一个**全局 low effort 守护 subagent（不阻塞你正常对话）：每个 turn 只执行一条 bash 命令，在命令里每 5 秒检查一次任务，约 12 分钟返回一次，直到全部完成或到截止时间。锁文件保证整个账号只有一个守护。

**安装。** 把上面那个 `SKILL.md` 链接发给你的 Grok Bot，让它存成技能就行。

## License

MIT
