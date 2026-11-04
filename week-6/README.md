# Week 6: make your task hold

**Due before class on Wednesday, November 18, 2026.**

In week 1 you built the hardest task you could, and we ran a frontier agent on it. Claude Opus 5.5 solved 6 of the 8 tasks, five of them in under 7 minutes. You now know much more about how these agents work, how they fail and how they get measured. Go back to your task and use all of it.

**The goal: a task that GPT-5.6 Terra and GPT-5.6 Sol don't solve. Aim for one that would take a human expert more than a year, ideally ten.** It has to stay solvable: a right answer exists, what the agent gets determines it, and your verifier can tell right from wrong. Making a task impossible, vague or unfair is easy, and it measures nothing.

## Start

1. Open an AI coding agent (Claude Code, Codex, Cursor, GitHub Copilot, …) in your clone of this repository.
2. Give it this:

   > Read https://class.xhelio.ai/week-6/homework.md and help me with my homework.

It goes through what happened to your task, has a fresh session review its solvability, hardens it with you, and opens your pull request.

**You'll need:** your week 1 task, [uv](https://docs.astral.sh/uv/), the GitHub CLI and an AI coding agent. No Docker: the checks run on GitHub.

**No week 1 task yet?** Build one with the [week 1 guide](../week-1/README.md), aiming at this week's goal from the start, then come back here.

## What you do

### 1. Read what happened

Open your task's page on [class.xhelio.ai/homework](https://class.xhelio.ai/homework/) and the agent's full trace. Find the decisive move: what did it notice, and what made it fast? If it failed, was that the science, or your task?

### 2. Fix its solvability

A task is **solvable** when a right answer exists, the instruction and data determine it, and a right answer reliably passes while a wrong one fails. It doesn't have to be quick for a person. Open a **new session** of your agent, so it doesn't know your reasoning, and give it this:

> Read `.agents/skills/check-solvability/SKILL.md` and review my task in `week-1/submissions/<your-github-username>/<task-name>/`.

Then fix what it finds: conventions the tests enforce but the instruction never states, ranges that don't cover the hidden data, an answer key built on one expert's unstated choices, a pass bar that a no-skill answer clears, or a shortcut that passes without the science.

### 3. Harden it

Change the task so the agent fails on the science. In class and in our own runs, agents rarely fail on code. They fail on one scientific judgement, and the test they use to check it is one they built themselves, carrying the same wrong assumption.

**Think in years of expert time.** A task a strong expert finishes in a week is too small. Aim for more than a year, ideally ten: the kind of result that took your group or your field that long. Count years of thinking: models tried and rejected, methods built, expertise learned. Ten years of repeating one step can take an agent minutes. Your reference solution still has to score 1, so build on work that's already done, such as your thesis, your group's years of analysis or a published campaign. The agent still gets at most 5 hours.

| What tends to work | Why |
|---|---|
| **Withhold the graded truth.** Hidden labels, a held-out or shifted test set | With no real check, the agent has only its own simulator or cross-validation |
| **Use real, observed data** | Real instruments have gaps, artifacts and systematics that no one wrote down |
| **Ask for the hidden cause.** The agent sees effects and recovers causes, parameters or history | It has to choose a model, and the obvious one may fit and be wrong |
| **Need a term it won't assume.** An instrument response, a coupling, a background | The right term isn't in the family of models it tries first, but the data show it |
| **Grade beyond the visible range**, with the hidden range stated | Models that fit equally well inside the range disagree outside it |
| **Chain the steps**, and grade the end | An early mistake quietly corrupts everything after it |

| What doesn't work | Why |
|---|---|
| More data or more cases | Agents script the pile. Their effort grows with the number of distinct methods, not the size of the data |
| A huge search space with a cheap check | Agents search very well when they can check |
| A recipe in the instruction | It then tests care, and careful rule-following is what coding agents do best |
| Blocking the one shortcut you saw | There's usually another one |
| Hidden conventions, vague wording, one obscure fact | That makes a task unfair or a coin flip: hard for the wrong reason |

Your agent has the details, in `.agents/skills/harden-a-task/SKILL.md`. If the agent solved your task quickly, adding to the same design rarely helps: keep your science and change the task's structure.

### 4. Review its solvability again

Hardening is how good tasks become unfair. When you think you're done, have another new session review the final version, and fix what it finds. Save its report as `authoring/solvability.md`.

### 5. Run GPT-5.6 Terra and Sol

One run of each, at `xhigh` effort. **Solved** means reward 1. Terra costs about half as much as Sol, so run it first; if Terra solves it, go back to step 3.

- **Ask the instructor.** Say so in your pull request, and we'll run them and post the results there.
- **Run them yourself**, if you have Docker and an OpenAI API key. A long run can cost tens of dollars.

  ```
  uvx harbor@0.23.0 run -p <task-folder> -a codex -m openai/gpt-5.6-terra --ak reasoning_effort=xhigh --ak web_search=disabled
  uvx harbor@0.23.0 run -p <task-folder> -a codex -m openai/gpt-5.6-sol --ak reasoning_effort=xhigh --ak web_search=disabled
  ```

- **A rough check, without Docker.** If your ChatGPT plan includes these models, your agent can set up a blind run with the Codex CLI and score it. It helps you iterate, but only a Harbor run counts as the result.

### 6. Read the attempts

Read each run before you believe its reward. A failure counts only if the agent had everything it needed and failed on the science; if it failed on a gap in your task, fix the gap and run again. If it solved the task, find out how, and go back to step 3.

Holding against Terra and Sol is the floor. If you get there, try a stronger model.

## What you submit

A pull request that updates your task in place, in `week-1/submissions/<your-github-username>/<task-name>/`:

- **The hardened task**: instruction, data, reference solution and verifier.
- **`authoring/solvability.md`**: the review of the final version, from a session that didn't help build it.
- **`authoring/attempts.md`**: one row per run, with what happened in your own words. Note what you changed and why under **Status**.
- **`README.md`**: updated Difficulty, Reference solution and Verification sections. Difficulty says what takes a human expert so long.
- **`task.toml`**: `expert_time_estimate_hours` (a year is about 2,000 hours, ten years about 20,000) and `class_level` (2 once it's honest, 3 if Terra and Sol fail on the science).

It counts when your reference solution scores 1, an agent that does nothing scores 0, the automatic checks pass, the final review says *solvable*, and you've reported a Terra run and a Sol run, or asked us for them.

**Still solved at the deadline?** Submit it anyway, with the run that beat it and what you'd try next. An honest task that was solved teaches more than an unfair one that wasn't.
