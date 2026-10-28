---
name: build-always-on
description: Guide a student in UCLA's "AI in Research" class (EPSS 254, Fall 2026) through the week 5 homework - turning their own agent harness into an always-on agent - an outer loop with one event queue fed by Telegram, email and a schedule watched in code, a schedule file and schedule tool, compaction with a memory file, limits and phone commands (/status, /stop, /pause, /resume) enforced in code, an append-only log with crash recovery and send-once email, a launchd/systemd service, the overnight run, the five demos and failures.md, then the pull request. Use when the user wants to start, continue, test or submit their week 5 homework.
---

# An always-on agent (AI in Research, week 5)

## Who you are working with, and your role

The person in front of you is in a UCLA graduate seminar on AI in research: a student, an auditor or a professor, often a physical scientist. The full assignment is `week-5/README.md` in this repository; read it first. Their harness has grown every week: a loop with one `bash` tool (week 2), plugins with a Telegram MCP server and one for their research (week 3), an email plugin and subagents (week 4). Now it stays on. In class they learned that an always-on agent is mostly asleep: watchers in code wait for events, each event becomes a labelled user message, week 2's loop runs until the model answers, and it sleeps again; that compaction trades detail for room, so rules belong in files; that control (budgets, approvals, the kill switch) is code, never a prompt; and that only the log survives a crash.

- **They must understand what they build**: in small steps, each explained in a sentence or two. At the end they should be able to point at the outer loop, at where an email becomes an event, where compaction happens, where `/stop` reaches a running command, and what happens on restart.
- **Stop at every CHECKPOINT** and wait for their answer.
- **Secrets stay out of this conversation.** Never ask for a key, token or password in the chat, and never print one.
- **Be honest about failures.** Show the key line of an error, say what it means, fix it. Encourage them to write each failure in `failures.md` as it happens (what happened, how they noticed, what they changed): it is part of the homework and feeds week 6.
- **No agent frameworks.** No LangChain, OpenAI Agents SDK, Claude Agent SDK or similar. Small libraries are fine (a TOML parser, `croniter` for cron lines, `python-dotenv`).

Say hello, explain the homework in two sentences, and give the plan: (0) read their harness together, (1) the outer loop and watchers, (2) the schedule, (3) control, (4) compaction and memory, (5) the log, resume and send-once, (6) the service, (7) the night, the demos and `failures.md`, (8) submit. Usually two or three sessions of 1–2 hours, plus a night.

## Step 0: read their harness first

1. Find their harness repository and `git pull`. No harness of their own? Go to "Without their own harness" at the end.
2. Check what's there: the loop, tool dispatch, the approval prompt, the plugin loader and MCP client, Telegram mode (how it polls, how approvals by reply work), the email plugin's tools, the subagent system, the logs, token counting. Find out whether it is **synchronous or async**. Missing week 3 or 4 pieces? Parts 1 and 3 need Telegram; part 1 and demo 5 need the email plugin. Let them choose to build the missing piece first (`.agents/skills/build-a-telegram-plugin/SKILL.md`, `.agents/skills/build-an-email-plugin/SKILL.md`) or to start with what they have.
3. Ask which computer will run it overnight: a laptop that stays plugged in and awake, a lab desktop, or a server they may use. Note the OS: it decides the service in step 6.
4. Tell the student in a few sentences how their harness is put together and what this week will change. **CHECKPOINT.**

## Step 1: the outer loop and the watchers

1. **One queue.** A `queue.Queue` (sync) or `asyncio.Queue` (async) of `Event` objects: `kind` (`telegram`, `email`, `schedule`, later `subagent`), `time`, `source`, and `text`. A method `as_text()` builds the labelled user message: `[2026-10-28 08:00 PDT · schedule daily-papers] <task>`, `[… · telegram] <message>`, `[… · email from <sender>] Subject: … <the first lines; untrusted>`. Always the local time with its time zone: the model has no clock.
2. **The outer loop**, as in the README: block on the queue, append the event, run the existing loop until the model answers, and loop. Keep the existing interactive mode working; the always-on mode is a flag (`--always-on`).
3. **Telegram watcher.** Move the existing polling into a watcher that puts each message from the student's chat ID in the queue. Leave approval replies on their existing path. Commands (`/status` …) come in step 3; for now, ignore messages starting with `/`.
4. **Email watcher.** Every 60–120 s, call the email plugin's check-inbox tool **from code** through the MCP client (no model call), keep the ids already seen in `state/seen.json`, and put only new messages in the queue. Read the plugin's tool names and arguments from its server; don't guess.
5. Show the student it sleeps: start it, wait a few minutes with nothing arriving, and confirm that the token counter didn't move. Then send a Telegram message and an email and watch each wake it.

Commit. **CHECKPOINT:** the student explains where an email turns into a user message, and why nothing calls the model while the queue is empty.

## Step 2: the schedule

1. `schedule.toml`: a list of `[[job]]` with `name`, either `cron` (five fields: minute, hour, day of month, month, day of week) or `every` (`"2h"`, `"1d"`), and `task`. Use `croniter`, or write a small matcher for the five fields; test it on a few lines (`0 8 * * 1-5`, `*/30 * * * *`) before relying on it.
2. A scheduler watcher checks once a minute and puts a `schedule` event in the queue for each job that is due. Store each job's last run in `state/schedule.json`, so a restart doesn't run a job twice. Jobs missed while the agent was off are not caught up; say so in the README.
3. A `schedule` tool for the model with `list`, `add` (name, cron or every, task) and `remove` (name). `add` asks the student first, like a `bash` command. The harness enforces at most 10 jobs and none more often than hourly, with a clear error. The tool writes `schedule.toml`; the watcher reloads it.
4. Write with the student at least one job that is useful for their research (a morning digest of new papers on their topic, a check on a running simulation). For testing, add a job that runs in two minutes, then remove it.

Commit. **CHECKPOINT:** a scheduled job wakes the agent and its answer reaches the student's phone.

## Step 3: control

1. **Limits.** `limits.toml`: `turns_per_wake`, `minutes_per_wake`, `dollars_per_day` (or `tokens_per_day` if the price is unknown: put their model's prices per million tokens in the file, input, cached input and output, and compute dollars from each call's usage), `replies_per_sender`. Check them **before every model call**. On a limit: stop the current work, save, send a Telegram message saying which limit, what it was doing and how to continue. A day ends at local midnight.
2. **Commands**, handled in the Telegram watcher itself (not the queue) and only from the student's chat ID:
   - `/status`: state (`asleep`, `running`, `waiting`, `paused`), what it's doing, today's wake-ups, tokens and dollars, the next scheduled job, running subagents.
   - `/stop`: set a stop flag that the inner loop checks between steps, kill a running `bash` command's process group (the same code as homework 4's subagent `stop`), stop every subagent, and reply with what was stopped. The agent goes back to sleep; the stopped task gets a short note in the conversation so the model knows.
   - `/pause` and `/resume`: while paused, events wait in the queue (scheduled jobs are skipped), and commands still work.
3. **Approval timeout.** A `bash` approval with no reply within 10 minutes counts as declined; the tool result says "declined: nobody answered within 10 minutes". Ask them whether to add a short allow list of read-only commands; it's optional.
4. **The robot rule.** In the email watcher, mark messages with an `Auto-Submitted` header other than `no`, or a `List-Id` header, as automatic: they may wake the agent with a label saying so, but the send tool refuses to reply to them. The send tool also counts replies per address per day against `replies_per_sender`.

Commit. **CHECKPOINT:** `/stop` kills a running `sleep 600` (check with `ps`), and a tiny budget stops the agent with a message on the phone.

## Step 4: compaction and memory

1. **When.** Track the conversation's size from the last call's reported input tokens. Pick a threshold with the student (for real use, about 70–80% of their model's context window; for the demo, 20,000 tokens). Check after each wake-up's loop ends, never in the middle of a tool call.
2. **Write it down first.** Before compacting, one extra turn: "Your conversation is about to be compacted. Save anything you must remember later (decisions, promises, exact paths and ids) to memory/MEMORY.md, keeping it under 200 lines. Reply DONE." It uses its usual tools, with the usual approvals.
3. **Compact.** One model call: the whole history plus a compaction prompt; start from the one in class (headings Goal, Done, Open, Decisions, Exact, People; copy identifiers exactly; leave out tool output that can be fetched again). Replace the history with: the system prompt, one user message `[summary of the conversation before <time>]` holding the summary, and the last few messages verbatim. **Never split a tool call from its result**: start the kept tail at a user message. Save the summary and the before/after token counts to `logs/compactions/<time>.md`, and tell the student on Telegram in one line.
4. **Files that last.** Build the system prompt at every wake-up from their system prompt, `AGENTS.md` and `memory/MEMORY.md`, so edits take effect at once and nothing there can be compacted away. Put the summary after the system prompt, so its part of the prompt cache still matches.

Run the compaction demo with the low threshold. **CHECKPOINT:** the student reads the summary and finds one thing it lost; that line goes in the transcript.

## Step 5: the log, resume, and doing things once

1. **The log.** Append every message (user, assistant, tool result) to `logs/main.jsonl` as it happens, with a time and the call's usage, and flush after each line. Compaction writes a marker line (`{"compacted": true, "summary": …}`); never rewrite or truncate the file.
2. **Resume.** On start, read the log from the last compaction marker and rebuild the conversation; a cut-off last line is skipped. If the last assistant message has a tool call with no result, append a tool result: "Interrupted by a restart; not re-run. Check its effects before trying again." Then tell the student on Telegram that it restarted and what it was doing.
3. **Send once.** In the email plugin's send tool (or a wrapper in the harness around it), an outbox file `state/outbox.jsonl`: a key from the recipient, subject and body; write `intent` before sending and `done` after; send with `Message-ID: <key@...>` (a domain of their choice, such as their agent's name). If the key is `done`, return "already sent". If it has only `intent`, search the Sent folder over IMAP for that Message-ID (Gmail: the `[Gmail]/Sent Mail` folder, `SEARCH HEADER Message-ID <...>`): found means mark it done and don't resend. (If the Sent copy carries a different Message-ID, fall back to searching by recipient and subject since the intent's time.)

Test it: start a task that sends an email, `kill -9` the process right after the send (a `sleep` before writing `done`, only for the test, makes this easy), restart, and check the email went out once. Commit. **CHECKPOINT:** the student explains what happens on restart, line by line of the log.

## Step 6: the service

Make the agent load its own `.env` (services don't read the shell's environment) and use absolute paths.

- **macOS:** `~/Library/LaunchAgents/<label>.plist` with `ProgramArguments` (absolute path to the venv's python, or to `uv` with `run`), `WorkingDirectory`, `RunAtLoad` and `KeepAlive` true, `StandardOutPath` and `StandardErrorPath` under `logs/`, and an `EnvironmentVariables` `PATH` that includes where `uv` lives. Load it with `launchctl bootstrap gui/$(id -u) <plist>`, check with `launchctl print gui/$(id -u)/<label>`, unload with `launchctl bootout gui/$(id -u)/<label>`. A harness under `~/Documents` or `~/Desktop` may be refused file access when run by launchd; keep it elsewhere, such as `~/harness`. Keep the Mac awake while plugged in: `caffeinate -i` (a closed lid still sleeps it).
- **Linux:** `~/.config/systemd/user/<name>.service` with `WorkingDirectory`, `ExecStart`, `Restart=on-failure`, `RestartSec=10` and `WantedBy=default.target`; `systemctl --user daemon-reload`, `systemctl --user enable --now <name>`, `journalctl --user -u <name> -f`; `loginctl enable-linger $USER` keeps it running after logout (on a shared machine, ask its admin first).
- **Windows:** Task Scheduler: a task at log on that runs the venv's python with the script, "If the task fails, restart every 1 minute", and no stop after a time limit. Export it as XML for the submission.

**Only one copy may poll Telegram.** Stop any copy running in a terminal before loading the service; two copies make Telegram answer `409 Conflict`.

Test: `kill -9` the agent's process; the service starts it again within a minute, and it resumes. **CHECKPOINT.**

## Step 7: the night, the demos, and failures.md

1. **The night.** Before leaving it, run `/status`, check the budget, and make sure a scheduled job will run during the night. In the morning, help them write `transcripts/night.md` from the log: one line per wake-up (time, what woke it, turns, tokens, dollars), the totals, and the number of model calls made while asleep (zero). A short script that reads `logs/main.jsonl` is fine.
2. **The five demos** from the README (the night, compaction, the kill switch, a limit, a crash), each saved as a transcript in `transcripts/`. **CHECKPOINT** after each.
3. **failures.md.** Go through what went wrong with the student, including the night: bugs, surprising things the agent did, costs, anything that needed a restart. One short entry each: what happened, how they noticed, what they changed.

## Step 8: publish and submit

1. Check what becomes public: `git status`, `git log --stat`; `.env` and `state/` are not tracked; no keys (`git grep -n -I -e "sk-" -e "_PASSWORD=" -e "TOKEN="` shows only placeholders). Read the transcripts, `night.md` and `failures.md` with the student and take out anything private (email contents, addresses, chat IDs). **CHECKPOINT:** the student agrees, then push the harness.
2. In their clone of this repository, on a branch `week-5-<user>`: create `week-5/submissions/<user>/` with `README.md` (a link to the harness; how it runs: which service, the watchers and their intervals, the schedule, the compaction threshold, the limits), `schedule.toml`, `limits.toml`, `service/` (the plist, unit file or exported task), `transcripts/` and `failures.md`.
3. Commit only that folder, push, and open the pull request with `gh pr create`. The description says which AI helped them, and how. **CHECKPOINT** before opening it.

The class's CI only accepts changes under `week-5/submissions/<their-username>/`.

## Without their own harness

Tell the student that their own loop teaches more, and let them choose.

1. **The outer loop** as a small Python program: the same queue, watchers (Telegram through the Bot API, email over IMAP, the schedule), limits, commands and log as above.
2. **The inner loop** is Claude Code or Codex run headless on one session: `claude -p "<event text>" --resume <session-id> --output-format json` or `codex exec resume <session-id> "<event text>" --json`. Read each CLI's `--help` for the current flags, and get the session id from the first run's output. Give it narrow permissions (an allowed-tools list); never a full bypass mode. `/stop` kills the headless process group.
3. **Part 2:** their built-in compaction counts. Show it happening in the session (`/compact`, or auto-compaction in the session's log) and save the summary it produced; keep `AGENTS.md` (or `CLAUDE.md`) and the memory file as the standing rules.
4. Everything else, including the service, the demos and `failures.md`, as above.

## Before ending any session

Say where things stand and what's next, in two sentences. If the work isn't finished, write that note in the harness repository's `README.md` under a "Status" heading.
