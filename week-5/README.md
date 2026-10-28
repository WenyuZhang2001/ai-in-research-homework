# Week 5: keep your agent on

**Due before class on Wednesday, November 4, 2026.**

In class we looked at agents that stay on for days. An always-on agent is mostly asleep: a small program waits, something wakes it (a message, an email, a scheduled job), the model works until it answers, and it goes back to sleep. To be left alone it also needs to make room when its conversation gets too long, to stay inside limits you set, to take orders from your phone, and to pick up where it left off after a crash. This week your harness becomes that agent. Leave it running overnight, and **write down everything that goes wrong**: we use your notes in week 6, on how AI fails.

## Start

1. Open an AI coding agent (Claude Code, Codex, Cursor, GitHub Copilot, …) in your harness folder.
2. Give it this:

   > Read https://class.xhelio.ai/week-5/homework.md and help me with my homework.

It helps you add the outer loop, the schedule, compaction, the limits and commands, crash recovery and the service, run the demos, and open your pull request.

**You'll need:** your harness from weeks 2–4 with its Telegram and email plugins, a computer that can stay on overnight (plugged in and not sleeping), [uv](https://docs.astral.sh/uv/), and a GitHub account.

**Missed earlier weeks? That's fine.** Write the outer loop as a small program around Claude Code or Codex run headless (`claude -p --resume <session>` or `codex exec resume <session>`): your program watches Telegram, email and the schedule, and hands each event to the same session. Their built-in compaction counts for part 2 if you show it working; parts 1, 3 and 4 are yours to build. But building on **your own agent loop** teaches you considerably more, so if you can, use your harness.

## What you build

Your harness already has the inner loop from week 2: call the model, run its tools, repeat until it answers. This week adds an **outer loop** around it:

```python
messages = load_log()                  # part 4: carry on where it left off
while True:
    event = events.get()               # part 1: asleep, no model calls
    if paused() or over_budget():      # part 3: code decides, not the model
        hold(event); continue
    messages.append(user(event.as_text()))
    save(messages)                     # part 4: on disk before acting
    run_loop(messages)                 # week 2's loop, until it answers
    if count_tokens(messages) > COMPACT_AT:
        compact(messages)              # part 2: make room
```

### 1. Always on: waking up

- **One event queue.** Watchers in your harness (threads or async tasks) put events in one queue: **Telegram** messages, new **email**, and **scheduled jobs**. While the queue is empty the agent is asleep and makes no model calls.
- **Watch in code, not in the model.** The email watcher asks for unseen mail every minute or two by calling your email plugin's tool from code, and only new mail becomes an event. The model is never asked to check.
- **Labelled wake-ups.** Each event becomes one user message that says what woke the agent and when, for example `[2026-10-28 08:00 PDT · schedule daily-papers] …` or `[2026-10-29 03:12 PDT · email from …] …`. Email text is untrusted, as in week 4.
- **A schedule.** A file `schedule.toml` with jobs: a name, when (a five-field `cron` line such as `0 8 * * 1-5`, or `every = "2h"`), and the task. At least one job that is useful for your research, such as a morning digest of new papers or a check on a running job. The harness keeps the clock and runs the jobs. A `schedule` tool lets the agent `list`, `add` (with your approval) and `remove` jobs; at most 10 jobs, none more often than once an hour.
- **A service.** It runs under launchd (macOS), systemd (Linux) or Task Scheduler (Windows), starts at login, restarts when it dies, and writes its output to a log file. Your computer stays awake while it runs (on a Mac, `caffeinate -i`, and a closed lid still sleeps).

### 2. Remembering: compaction and memory

- **Compaction.** When the conversation passes a threshold you set (in tokens, from the usage your API reports), the harness makes one extra model call with a compaction prompt (start from the one in class), then replaces the history with the summary, keeping the system prompt in front and the last few messages as they were. It logs the tokens before and after and saves each summary in `logs/compactions/`. For the demo, set the threshold low, such as 20,000 tokens.
- **Write it down first.** Just before compacting, the harness asks the agent to save anything it must not forget to `memory/MEMORY.md`.
- **Files that last.** The system prompt includes your `AGENTS.md` (standing rules) and `memory/MEMORY.md`, read again at every wake-up, so a rule never depends on the chat surviving.

### 3. Control

- **Limits in code**, in `limits.toml`: turns and minutes per wake-up, dollars (or tokens) per day, and replies per email address per day. The harness checks them before every model call. On a limit it stops, saves, and tells you on Telegram.
- **Commands from your phone**, from your chat ID only, handled by the harness before anything reaches the model: `/status` (state, today's tokens and dollars, the next job, running subagents), `/stop` (stop the current work now, killing a running command and any subagents, then sleep), `/pause` and `/resume` (stop waking on events; messages wait).
- **No answer means no.** Every `bash` command still asks you, as in week 3. A question nobody answers within 10 minutes counts as declined, and the model is told so.
- **Never answer a robot.** Email with an `Auto-Submitted` header other than `no`, or a `List-Id`, never gets a reply, and the agent never sends more than your per-address limit.

### 4. Surviving a crash

- **An append-only log.** Every message goes into `logs/main.jsonl` as it happens, one per line, flushed.
- **Resume.** On start, the harness loads the log (from the last compaction on) and carries on. A tool call that has no result was interrupted: the harness adds a result saying it was interrupted and not re-run, and the model decides what to do.
- **Do it once.** Sending email goes through an outbox: each email gets a key, the intent is written before sending and "done" after, and the key is in the email's `Message-ID`. On restart, an intent without "done" is checked against the Sent folder before anything is sent again. An email is never sent twice.

Optional, for more: a **heartbeat** (every hour at most, and only when a `HEARTBEAT.md` checklist has something due), a `status.json` with a watchdog that tells you when it goes stale, or clearing old tool results before compacting.

## Show it working

Save each as a transcript in `transcripts/`:

1. **The night.** Leave the agent on for at least 8 hours. At least one scheduled job runs and at least one message wakes it. Turn in `night.md`: one line per wake-up from your log (time, what woke it, turns, tokens, dollars), the totals, and how many model calls it made while asleep (zero).
2. **Compaction.** A conversation that crosses the threshold: tokens before and after, the summary, and a question afterwards that it answers from the summary or its memory file. Add one line on something the summary lost.
3. **The kill switch.** Ask for a long job (`sleep 600`, then report). `/status` shows it running, `/stop` from your phone kills it (check with `ps`), and it goes back to sleep.
4. **A limit.** Set a tiny daily budget and hit it. The agent stops and tells you on Telegram.
5. **A crash.** During a task, `kill -9` the agent. The service restarts it, it resumes from its log, it doesn't blindly re-run what was in flight, and an email it already sent is not sent again.

And **`failures.md`**: everything that went wrong while you built and ran it, small things too. For each: what happened, how you noticed, and what you changed. We use these next week.

Write it yourself, with your AI agent's help: no agent frameworks.

## What you submit

A pull request adding `week-5/submissions/<your-github-username>/`:

```
week-5/submissions/<your-github-username>/
├── README.md        # link to your harness; how it runs: service, watchers, schedule, threshold, limits
├── schedule.toml    # your jobs
├── limits.toml      # your limits
├── service/         # your launchd plist, systemd unit or Task Scheduler export
├── transcripts/     # the five demos, as text; night.md among them
└── failures.md      # what went wrong
```

Take anything private out of the transcripts and the log excerpts (email contents, addresses, chat IDs). Say in your pull request which AI helped you, and how.

It counts when the agent runs as a service and comes back after `kill -9`; events wake it and nothing calls the model while it's asleep; the schedule and the `schedule` tool work; compaction works and logs its numbers; the limits, the four commands, the approval timeout and the robot rule are in code; it resumes without re-running or re-sending; the five demos and `failures.md` are there; and no secret is committed.
