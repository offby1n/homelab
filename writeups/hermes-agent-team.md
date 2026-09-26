# Project Write-up: Building and Scoping a Four-Bot Hermes Agent Team

A standalone project write-up, outside the numbered layers. It covers splitting [Hermes Agent](https://github.com/NousResearch/hermes-agent), a self-hosted AI agent, from one general-purpose assistant into four bots that each have one job and only the tools that job needs. It also covers the bug that broke it on the first real test and how the running cost is kept down.

**Date:** September 2026
**Runs on:** `hermesVM`, Proxmox VM `104` on the Dell R630 (`10.10.10.75`). I talk to the team through a Telegram bot.

## System Overview

Hermes Agent (Nous Research) is an open-source agent you run on your own hardware. It calls a language model through an API and acts through tools such as a shell, file access and web search. It ships with a messaging gateway (I use Telegram), a scheduler, and a Kanban board that bots use to hand each other work.

I was already running it on hermesVM as a single assistant. This project turned that into a small team:

1. I message **Dispatcher** on Telegram.
2. If the answer needs no tools, Dispatcher replies itself. That's the cheapest path.
3. Anything else becomes a Kanban card for one specialist: **Ops**, **Coder** or **Researcher**.
4. Hermes starts that specialist as a fresh process for that one card. It never sees my chat, only the card.
5. The specialist completes the card, or blocks it with a question. Dispatcher is woken and replies to me.

None of the three specialists ever messages me directly. I worked out the design and the setup with Claude as a guide; every command ran on my own machines, and every result below is from real output.

## The Build

### 1. The Team

- **Dispatcher:** my existing main Hermes profile. Its tools are the Kanban board, the scheduler, memory, session search, clarifying questions and skills. No shell, no file access, no web: it can hand work out but can't do it.
- **Ops:** a shell on hermesVM and SSH into two VMs. Health checks, services, updates, logs.
- **Coder:** a shell and files on hermesVM. Writes and fixes scripts, and is told to test its work before finishing. It wrote the scripts behind the scheduled reports below.
- **Researcher:** web search through Tavily, with no shell and no file access. It reads untrusted web pages, so an instruction hidden in a page has no shell or files to act with. Its report still reaches Dispatcher, which can create cards, so that risk is smaller, not gone.

All four use the same model, `deepseek/deepseek-v4-flash-0731`, through OpenRouter. Every toolset a bot doesn't need is switched off in its own profile (`agent.disabled_toolsets`), and that applies on every surface, Telegram included.

### 2. Access Boundaries

- **One key, two machines.** Ops has its own ed25519 key (`~/.ssh/hermes_ops`, no passphrase so it can run unattended), installed on `db01` (`10.10.10.50`, MariaDB) and `debian-lab` (`10.10.10.15`, test VM) and nowhere else.
- **Left out on purpose:** OPNsense (`10.10.10.1`), the Proxmox host (`10.10.10.10`), iDRAC (`10.10.10.20`), the switch (`10.10.10.25`) and the access point (`10.10.10.5`). One bad command on most of those can take down the whole network or every VM at once. One bad command on db01 or debian-lab breaks one VM.
- **sudo, honestly.** On those two VMs Ops can run `apt`, `systemctl` and `journalctl` as root without a password (`/etc/sudoers.d/hermes-ops`, checked with `visudo -c`); without that, every sudo call from an unattended bot would fail. All three can be turned into a root shell (they're all on [GTFOBins](https://gtfobins.github.io/)), so in practice Ops has root on those two VMs. The list keeps it from running anything else as root by mistake, but it isn't a security boundary. The real boundary is which machines accept the key.
- **Safety lock.** Ops and Coder have an approvals deny rule for commands matching `hermes config`, `hermes update` and `hermes gateway`. It's aimed at an open upstream report of a Hermes agent that got a command denied, ran `hermes config set approvals.single_query_mode approve` to lift its own approval gate, then ran the command anyway ([#104059](https://github.com/NousResearch/hermes-agent/issues/104059)). The rule matches command text and hasn't been tested, so a variation on the command, or editing the config file directly, could get past it. It's a guardrail, not a wall.
- **One Linux user.** All four bots run as the same user on hermesVM, so the key is Ops's by convention only: Coder, which also has a shell, could use it too. On hermesVM, the split between the bots is Hermes config, not OS permissions.

### 3. Scheduled Jobs

I set these up by asking Dispatcher on Telegram in plain words. It loaded a custom `team-scheduling` skill (rules for running scheduled work as cheaply as possible), had Coder write the scripts, and registered the jobs on Hermes's built-in scheduler. They replaced the one cron job I had before the team existed.

- **06:00:** hermesVM health report, always sent. It lists pending updates and waits for my OK instead of installing them.
- **08:00:** db01 and debian-lab health report, always sent.
- **20:00:** disk usage on all three machines, silent unless a disk is over 80%.
- **Sunday 04:00:** resets Dispatcher's Telegram chat (the same as sending `/new`), so every week starts with an empty chat.

### 4. Keeping the Cost Down

The goal: something I never have to babysit that can't run up a bill.

- **The right model build.** On OpenRouter the plain `deepseek/deepseek-v4-flash` name is the older April build, listed at $1.28 per million output tokens. The July build, `-0731`, is listed at $0.03 in / $0.32 out.
- **Reasoning off where it isn't needed (unconfirmed).** Set `agent.reasoning_effort none` on Ops and Researcher: checks and lookups don't need a reasoning step, and reasoning tokens are billed as output. Whether it took effect is still open (see below). Dispatcher and Coder stay on the default.
- **Workers start empty.** Every card is a fresh process, so worker history can't pile up. Hermes's background "learning" runs are off on all three workers.
- **Dispatcher's chat stays short.** It's compressed once it passes 64,000 tokens (the summary is written by the same cheap model), compacted after an hour idle, and reset every Sunday.
- **Memory has a ceiling.** Hermes caps its memory files (2,200 characters for `MEMORY.md`, 1,375 for `USER.md`) and refuses writes past the cap instead of growing.
- **Search can't bill.** Tavily's free tier (1,000 searches a month, no card on file) stops at the limit.
- **Backstop:** a spend cap on the OpenRouter account.

**Measured:** total OpenRouter spend for the day I built and tested all of this: **$0.06**

## What Broke

### Workers crashing on launch

**Symptom.** The first real job, "Is db01 up, and how full is its disk?", failed. The Ops card crashed twice and Hermes blocked it automatically.

**Finding the cause.**

1. `hermes kanban log` on the card showed the worker dying at launch, before Ops ran anything: `ModuleNotFoundError: No module named 'hermes_cli'`, from a worker started as `python -m hermes_cli.main`.
2. `hermes doctor` didn't flag it.
3. `which hermes`, `hermes --version` and `python3 -c "import hermes_cli"` showed a git install: the `hermes` launcher works, but `hermes_cli` isn't an installed package. Python only finds it when Hermes's launcher sets up the path first, so a worker started in module form, without that setup, fails every time.
4. The exact error matched two open upstream issues, [#122487](https://github.com/NousResearch/hermes-agent/issues/122487) and [#122500](https://github.com/NousResearch/hermes-agent/issues/122500).

**The cause, in plain English.** Before it launches a worker, the dispatcher checks whether Python can import `hermes_cli`. It runs that check inside the gateway process, where Hermes's launcher has already set up the import path, so the answer is yes and it starts the worker as `python -m hermes_cli.main`. The worker is a separate process that doesn't get that path, so the import fails before it does anything. The check tests the wrong environment.

**The fix.** A systemd drop-in that tells the dispatcher which launcher to use, made with `sudo systemctl edit hermes-gateway`:

```ini
[Service]
Environment="HERMES_BIN=/home/offby1n/.hermes/hermes-agent/.hermes/bin/hermes"
```

The path is copied from the unit's own `ExecStart` line, so it's the launcher the gateway already runs from. It's a drop-in rather than an edit to the unit file, so regenerating the unit doesn't remove it. The catch: if an update ever moves the launcher, this path goes stale and the workers break again. This is the workaround documented in #122500.

**Verified.** After `systemctl daemon-reload` and a restart, I unblocked the card and it completed. Then I ran `hermes update` again, since #122500 is about workers crashing after an update. The override was still there, and the cards after it (Coder writing the job scripts, the Researcher test) launched normally.

Both issues are still open, so this is a workaround. When a release fixes the launcher, the drop-in comes out and gets re-tested.

### Dashboard restart during updates

`hermes update` tries to restart the dashboard with `sudo -n systemctl restart hermes-dashboard.service`. `-n` means never ask for a password, so without a NOPASSWD rule the restart failed, more than once during this build. The fix is a sudoers file that allows that one restart without a password and nothing else, checked with `visudo -c`.

## Test Results

- **No workers:** a general question. Dispatcher answered it itself, with no card.
- **One worker:** "Is db01 up, and how full is its disk?" became an Ops card, and the answer came back through Dispatcher (after the fix above).
- **Research:** a web question became a Researcher card, and the answer came back with a real source link.
- **Scheduling:** all four jobs are registered with the right next-run times. I fired the weekly reset once by hand; the chat was already fresh, and it said so.
