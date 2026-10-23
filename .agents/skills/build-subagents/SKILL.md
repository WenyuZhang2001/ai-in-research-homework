---
name: build-subagents
description: Guide a student in UCLA's "AI in Research" class (EPSS 254, Fall 2026) through part 4 of the week 4 homework - a plain subagent system in their own agent harness - agent type files in agents/, a delegate tool (hand over and wait for the report), a subagent tool that starts, lists, messages, waits for, reads and stops background subagents, the limits the harness enforces, and the five demos for the pull request. Use when the user wants to start, continue, test or submit the subagent part of their week 4 homework.
---

# Subagents in your own harness (AI in Research, week 4, part 4)

## Who you are working with, and your role

The person in front of you is in a UCLA graduate seminar on AI in research: a student, an auditor or a professor, often a physical scientist. The full assignment is `week-4/README.md` in this repository; read it first, especially part 4. In week 2 they built a harness (a loop with one `bash` tool), in week 3 it learned to load plugins, and in parts 1–3 of this week it got an email plugin. Now it learns to start copies of its own loop. In class they learned that a subagent is one tool call for the main agent and a whole loop inside: a fresh context, a task, and a report; anything not in the task, the subagent doesn't know, and anything not in the report, the main agent never learns.

- **They must understand what they build**: in small steps, each explained in a sentence or two. At the end they should be able to point at where a subagent starts, where its report comes back, and how `stop` reaches it.
- **Stop at every CHECKPOINT** and wait for their answer.
- **Secrets stay out of this conversation.** Never ask for a key or password in the chat, and never print one.
- **Be honest about failures.** Show the key line of an error, say what it means, fix it.
- **No agent frameworks.** No LangChain, OpenAI Agents SDK, Claude Agent SDK or similar: writing the subagent system is the homework.

Say hello, explain part 4 in two sentences, and give the plan: (1) read their harness together, (2) turn the loop into something that can run many times, (3) agent types, (4) `delegate`, (5) background subagents with the `subagent` tool, (6) the five demos, (7) submit. Usually one or two sessions of 1–2 hours.

## Step 0: read their harness first

1. Find their harness repository and `git pull`. No harness of their own? Go to "Without their own harness" at the end.
2. Parts 1–3 not done yet? Demo 1 needs the email plugin. Let them choose: do the email part first (`.agents/skills/build-an-email-plugin/SKILL.md`), or build subagents first and come back for demo 1.
3. Read the loop, the tool dispatch, the approval prompt, the plugin loader and the interface (web or terminal; Telegram mode). Find out whether it is **synchronous or async**: the week 3 MCP client is often async. Tell the student in a few sentences how their harness is put together, and what part 4 will change. **CHECKPOINT.**

## Step 1: one loop, many agents

Make the loop a function of an `Agent` object instead of global state. An `Agent` holds: an id (`main`, `sa-1`, `sa-2`, …), its type, its system prompt, its message list, its tool list, its state, an inbox of pending messages, its limits (`max_turns`, a deadline), its token usage, and the path of its log file. The main agent is an `Agent` too.

Run the harness afterwards: nothing should look different. Commit. **CHECKPOINT:** ask the student what a subagent will share with the main agent (the code, the model client, the plugins, the approval prompt, the working folder) and what it won't (the conversation, and the tools it isn't given).

## Step 2: agent types

1. Load `agents/*.md` at start: frontmatter `name`, `description`, `tools` (a comma-separated list or a YAML list), optional `model` and `max_turns`; the body is the system prompt. A missing `tools` line means `bash` plus every plugin tool. A tool name that doesn't exist is an error at start, saying which file and which name.
2. `delegate` and `subagent` are never given to a subagent, whatever its file says.
3. In the main agent's system prompt, list each type's name and description, the way skills are listed.
4. Write `agents/mail-reader.md` as in the README (only the two read tools of their email plugin, which are named `<server>__<tool>` in their harness; check the real names). Then a research type with the student: **CHECKPOINT**, they choose what it does and which tools it gets. Read-only is best. Tell them the last lines of a type's prompt matter most: they say what goes in the report.

## Step 3: `delegate`, the hand-over

1. A tool `delegate(agent, task)`. Its description, written for the model: the subagent sees **nothing** of this conversation, so the task must say everything it needs, and exactly what to put in the report; use it for one job whose answer you need before going on.
2. It builds a new `Agent` from the type (first user message: the task, nothing else), runs the loop to the end in the same thread or task, and returns the report with one line of stats, such as `[sa-1 · mail-reader · 4 turns · 9,812 tokens · 21 s]`.
3. Approval questions for a subagent's `bash` commands name the subagent: `[sa-1 · mail-reader] ls data/`. The week 3 allow list still applies.
4. Save every step of the subagent to its log, `logs/<session>/<id>.jsonl`: each message, tool call and result, with a time. Add its tokens to the session's totals.
5. `max_turns` (default 20) and a time limit (default 10 minutes): over either, the subagent ends as `failed`, and the tool result says which limit and how far it got.

Test demo 1: "What's new in your mail?" **CHECKPOINT:** open the main agent's conversation (or print its messages) with the student and confirm the only trace of the inbox is the report; then open the subagent's log and see everything the main agent didn't.

## Step 4: the `subagent` tool, in the background

One tool with an `action` field (`start`, `list`, `send`, `wait`, `read`, `stop`) and the parameters each action needs, all described for the model: use `start` and `wait` for independent jobs that can run side by side; say in every task what to report.

**Running in the background.** Match the harness:

- **Synchronous harness:** one `threading.Thread(daemon=True)` per subagent, a `queue.Queue` inbox, a `threading.Event` for stop, and one lock around shared state and the approval prompt. If the MCP client is async, call its tools from the threads with `asyncio.run_coroutine_threadsafe(coro, loop).result(timeout=...)` on the loop that owns the MCP sessions; never open a second session per thread.
- **Async harness:** one `asyncio.Task` per subagent, an `asyncio.Queue` inbox, `task.cancel()` for stop, and an `asyncio.Lock` around the approval prompt. Run the model call without blocking the loop (the async client, or `asyncio.to_thread`).

**`bash` must be stoppable.** Start each command with `subprocess.Popen(..., start_new_session=True)` (on Windows, `creationflags=subprocess.CREATE_NEW_PROCESS_GROUP`) and keep the handle on the `Agent`, so `stop` can kill the whole process group (`os.killpg(proc.pid, signal.SIGKILL)`; on Windows `taskkill /T /F /PID <pid>`), not just the shell.

**The actions:**

- `start(agent, task)`: refuse with a clear error if 4 are already running ("4 subagents are running: wait for one or stop one"). Otherwise create it, start it, return the id at once.
- `list`: one line per subagent: id, type, state, turns, tokens, time running, the first words of its task.
- `send(id, message)`: put the message in its inbox. A running subagent takes everything in its inbox as one user message before its next model call. An idle one goes back to `running` with its conversation intact. A stopped or failed one: an error saying so, and that `start` makes a new one.
- `wait(ids, timeout)`: block until every listed id is idle, stopped or failed, or the timeout runs out (default 60 s, at most 600 s); for each id, return its state and its report (cut at about 4,000 characters, with a note that the full report is in its log).
- `read(id, last)`: its last `last` steps from its log (default 5), each cut at about 500 characters.
- `stop(id)`: set the stop event or cancel the task, kill the running command, mark it `stopped`, and return what it was doing.

**States:** `running` → `idle` (it gave its report) → `running` again on `send`; ends as `stopped` or `failed`. When the session ends, stop every subagent.

**Notes to the main agent.** When a background subagent goes idle, stops or fails, add a note to a thread-safe list on the main agent. Before the main agent's next model call, take all the notes and add them as one user message, such as `[subagent notes] sa-2 (paper-reader) is idle after 7 turns: <report>`. Show the note in the interface too, so the student sees it even when the main agent is waiting for them.

**The interface.** A small panel or a `/agents` command showing what `list` shows; approval questions labeled with the subagent; in Telegram mode, the same labels in the chat. Keep it small.

Commit. **CHECKPOINT:** the student explains, pointing at the code, what happens between `start` and the note that arrives later, and how `stop` reaches a command that is running.

## Step 5: the five demos

Run each with the student, and save each as a transcript in `transcripts/` (a short script that turns the main agent's log, and the logs of the subagents it ran, into readable Markdown is fine):

1. **Hand over your inbox:** "What's new in your mail?" through `delegate` to `mail-reader`.
2. **Fan out:** a research question with three independent parts; three `start`s, a `wait`, a combined answer. Check the three really ran at the same time (their logs' times overlap).
3. **Follow up:** `send` one of them a follow-up; it answers without redoing its work (its log shows no new reading for what it already had).
4. **Stop:** a subagent on `sleep 600`, then report; the main agent `list`s and `read`s it, then `stop`s it. Check with `ps` that the `sleep` is gone.
5. **A limit:** a type or task with `max_turns: 3` and a job that needs more; it ends as `failed`, and the note says why.

Fix what breaks. **CHECKPOINT** after each demo.

## Step 6: publish and submit

1. Check what becomes public: `git status`, `git log --stat`; `.env` is not tracked; no keys (`git grep -n -I -e "sk-" -e "_PASSWORD="` shows only placeholders). Read the transcripts with the student and take out anything private (email contents they don't want public, personal addresses). **CHECKPOINT:** the student agrees, then push the harness.
2. In their clone of this repository, on their branch `week-4-<user>` (the same branch and pull request as the email part, if they already have one): copy `agents/` to `week-4/submissions/<user>/agents/` and the transcripts to `week-4/submissions/<user>/transcripts/`, and add to `week-4/submissions/<user>/README.md` a paragraph on how their subagents run: threads or async tasks, where their state lives, how `stop` kills a command.
3. Commit only that folder, push, and open or update the pull request with `gh pr create` (or just push, if it's open). The description says which AI helped them, and how. **CHECKPOINT** before opening it.

The class's CI only accepts changes under `week-4/submissions/<their-username>/`.

## Without their own harness

Claude Code and Codex already hand over. Tell the student so, and that their own loop teaches more; let them choose.

1. **Demo 1:** `mail-reader` as a native subagent, `.claude/agents/mail-reader.md` (tools: the email plugin's two read tools, named `mcp__<server>__<tool>`) or `.codex/agents/mail-reader.toml`. Read the current docs for the format.
2. **The `subagent` tool** as a fourth plugin, `plugins/<user>-subagents/`, an MCP server whose tools are the six actions. `start` launches a headless agent as a background process (`claude -p … --output-format stream-json` or `codex exec --json`), writing its output to a log; `send` to an idle one continues it by session id (`claude -p --resume <id>`, `codex exec resume <id>`), and to a running one is queued until it's idle (headless runs take no input mid-run: say so in the tool description); `wait`, `read` and `list` read the process state and the logs; `stop` kills the process group. Read each CLI's `--help` for the current flags. Give headless runs narrow permissions (an allowed-tools list, read-only where possible); never `--dangerously-skip-permissions` or a full bypass mode.
3. Demos 2–5 with that plugin, and submit the plugin folder too.

## Before ending any session

Say where things stand and what's next, in two sentences. If the work isn't finished, write that note in the harness repository's `README.md` under a "Status" heading.
