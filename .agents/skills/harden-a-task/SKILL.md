---
name: harden-a-task
description: Guide a student in UCLA's "AI in Research" class (EPSS 254, Fall 2026) through the week 6 homework - hardening their week 1 Harbor benchmark task until GPT-5.6 Sol fails on the science, aiming at more than a year of human expert time. Reads what happened in round 1, gets an independent solvability review, applies what is known about how frontier agents fail (self-referential validation, missing hypothesis terms, out-of-support extrapolation, premature completion), picks real levers over more of the same, re-reviews, opens the pull request, and reads the GPT-5.6 Sol run the class does on it. Use when a student asks to harden or improve their task, or for help with this homework.
---

# Harden a task (AI in Research, week 6)

## Who you are working with, and your role

The person in front of you is in a UCLA graduate seminar on AI in research: a student, an auditor or a professor, often a physical scientist. The assignment is `week-6/README.md` in this repository; read it first. Their week 1 task is in `week-1/submissions/<github-username>/<task-name>/`.

The goal: **GPT-5.6 Sol doesn't solve the task. Aim for one that would take a human expert more than a year, ideally ten.** It has to stay solvable: a right answer exists, what the agent gets determines it, and the verifier can tell right from wrong.

- **The student owns the science.** You do the engineering and propose; they decide what the task asks, what counts as right, and what an expert would accept. Stop at every CHECKPOINT and wait for their answer.
- **You are the kind of system this task tests.** Use that. When you consider a design, say honestly how you would attack it, and which check you would write for yourself.
- **Never trade solvability for difficulty.** A hidden convention, a vague sentence or an impossible target would make any agent fail and would measure nothing.
- **Be honest about results.** Never edit a reward, fake an attempt or claim a run you didn't do. A solved task reported honestly is a good result.
- **Secrets stay out of this conversation.** Never ask for a key or password, and never print one.

Say hello, explain the homework in two sentences, and give the plan: (0) get the task on a branch, (1) read what happened, (2) solvability review, (3) choose how to harden, (4) build it, (5) second review, (6) pull request, where the class runs GPT-5.6 Sol, (7) read the attempts, (8) finish. Usually two or three sessions of 1–2 hours, plus waiting for runs.

## Step 0: get the task on a branch

```
git fetch upstream            # or origin, in a clone that has no fork yet
git switch week1-<task-name> || git switch -c week1-<task-name> upstream/main
git merge upstream/main
uv run tools/hw.py
```

`uv run tools/hw.py` is the same checklist as week 1. Any change to the task invalidates the student's earlier approvals, on purpose: they sign off again with `uv run tools/hw.py approve instruction`, `window` and `publish` in their own terminal. Never run these for them. **No task yet?** Follow `.agents/skills/build-a-task/SKILL.md` first, using this guide when you choose the design. Where that guide asks for a task a top expert could do, this week's goal replaces it.

## Step 1: read what happened (CHECKPOINT)

Read the task itself, then `https://class.xhelio.ai/homework/tasks/<task-name>` and the trace at `https://class.xhelio.ai/homework/traces/<task-name>`, with the student. Tasks without a run have no page; skip to Step 2.

Find the **decisive move**: what cue did the agent use, what made it fast, and did it verify itself? Then name what the result really shows:

| What you see | What it means |
|---|---|
| Solved quickly, method spelled out in the instruction | The task tested care. Turn the recipe into a question |
| Solved by a self-check that matches the grader | The agent could verify its own answer. Take that away |
| Solved, but the bar is low | The pass says little. Compare against a real baseline |
| Failed by a whisker, on a choice experts could make differently | Possibly the key's fault. This is a solvability problem first |
| Failed on the science, with everything it needed | The task holds. Check the verifier is fair, then keep it |

**CHECKPOINT:** say in three sentences what the task really measured. Ask if the student agrees.

## Step 2: the solvability review (CHECKPOINT)

The review must come from a session that didn't help build the task. Ask the student to open a **new session** of their agent and give it:

> Read `.agents/skills/check-solvability/SKILL.md` and review my task in `week-1/submissions/<github-username>/<task-name>/`.

When `authoring/solvability.md` exists, go through it together and fix each problem. Fixing comes first: hardening an unfair task only hides the unfairness. If the task's failure in round 1 came from its key, a fix may turn a failure into a pass. That's fine; it was never a real failure.

**CHECKPOINT:** the student agrees the science in each fix is right.

## Step 3: choose how to harden (CHECKPOINT)

### Think in years of expert time

A task a strong expert finishes in a week is too small. Aim for more than a year, ideally ten. Ask the student: what result took your group or your field that long? How long did it really take, and what took the time?

- **Count thinking, not repetition.** Years of models tried and rejected, methods built, instruments understood and expertise learned count. Ten years of repeating one step can take an agent minutes.
- **Base the estimate on history.** How long the work actually took is better evidence than a guess. Record it in `expert_time_estimate_hours` (a year is about 2,000 hours, ten years about 20,000) and in the README's Difficulty section.
- **The reference still has to score 1.** So build on work that's already done: a thesis, a group's years of analysis, a published campaign. Ship its output in `solution/data/`, and record how it was made. No expert has to redo it within the task's limits.
- **The agent still gets at most 5 hours,** and the computing a good method needs must fit in that. A year of human thought can fit; a month of CPU can't.

### How agents fail

From reading many runs of frontier agents on hard science tasks. **The failures are almost never bad code.** The engineering checks usually pass. The agent fails on one scientific judgement, and the test it uses to check that judgement is one it built itself, carrying the same wrong assumption.

| How it fails | What happens |
|---|---|
| **It validates against itself** | It writes its own simulator or cross-validation. Its test data come from its own assumptions, so its self-tests are near perfect and the hidden data fail. The most common mode |
| **It sees the clue and explains it away** | The decisive evidence appears in its own output. It notices, and offers an explanation that leaves its model unchanged |
| **It commits early** | A key call in the first minutes; the next hour builds on it and never returns to it |
| **It works on the wrong thing** | Time goes to checks already at full score, and the step that loses the points gets little |
| **It lacks a term** | The right answer needs a structural term it never considered: not noise handling, a term absent from its model family |
| **It extrapolates** | It validates inside the data's range and predicts outside it, where models that fit equally well diverge |
| **It stops early** | It submits once its local checks are green, often far inside the time limit |

### What tends to work

Each lever makes these failures possible while staying fair. Pick what suits the science, and chain them.

- **Withhold the graded truth.** This is the strongest one. Ask: *while working, can the agent check a candidate against what's graded?* In the Terminal-Bench-Science tasks we studied, those where it could were nearly all solved, however large the search space. Almost all the difficulty sat in tasks whose truth was hidden, leaving only proxies. Agents search very well when they can check.
- **Real, observed data.** In our runs, tasks built on observed data held about twice as often as tasks built on simulations or existing code. If the data come from a generator, the agent can often infer the generator. Prefer real data with its gaps, artifacts and systematics, where the reference comes from your own paper, or from an independent measurement the data can't reveal directly.
- **The reverse problem.** The agent sees effects and must recover causes, parameters, sources or history.
- **A term it won't assume.** An instrument response, a coupling, a background, a selection effect. It has to be visible in the data for someone who looks, but absent from the first model anyone would fit.
- **Grade beyond the visible range.** The hidden cases lie in a stated range outside the visible data. Models that agree inside disagree there.
- **Chain the steps; grade the end.** Each step uses an earlier step's result, so an early wrong call corrupts the final answer without an error. Separate checks side by side add little, because they don't compound.
- **Answers and a method.** Where the system allows, ask for the answers and a script the verifier runs on fresh hidden draws, with their distribution stated. This defeats memorizing and special-casing.
- **Make brute force not fit.** The stated hidden sizes put the naive method about 10 times over the limits, while the smart method fits easily. Keep this honest: the smart method must exist and be in the reference.
- **Ask for something per item that one rule can't supply.** With a large pile, the agent's effort stays constant, because it finds one rule and scripts it. Effort grows only if each item needs its own argument.

### What doesn't work

- **More of the same.** More cases, more data or a longer run. Agent time follows the number of distinct methods needed, not the data's size. To harden, add a step that needs a new method.
- **A huge search space with a cheap check.** It looks hard, but agents are good at it.
- **Blocking the one shortcut you found.** Another usually appears. If a physical quantity changes monotonically, you can't remove every clock the agent could read. Change the design, not the symptom.
- **Transforms that lose nothing.** Resampling, reordering or renaming can be undone. Only a truly lossy step removes information, and it can remove the expert's route too.
- **White noise and crops.** Usually undone by a blur or by whatever remains.
- **A tighter window alone.** It adds no science and starts rejecting good answers.
- **Hidden conventions, vague wording, one obscure fact, an impossible target.** That's hard for the wrong reason, and a task review will reject it.

### Pivot or patch

- **Pivot, by default, if the agent solved it quickly.** A design it finished in minutes rarely becomes slow by adding to it. Keep the student's system and field; change the question, the deliverable, the structure and the grader.
- **Patch if it was a genuine near miss on the science.** Add more hidden cases of the kind it missed, and remove the cue it relied on, fairly.

**CHECKPOINT:** propose two or three concrete designs from the student's own science. For each, give one line each on what goes in, what comes out, how it's graded, how long a human expert would need and what takes that time, which failure modes it invites, and *how you yourself would attack it*. The student picks one. Then write down, for each lever, what makes it fair: the evidence a careful expert would use.

## Step 4: build it

Work as in the week 1 guide (`.agents/skills/build-a-task/SKILL.md`): update `instruction.md`, `environment/`, `solution/`, `tests/`, `task.toml` and the README sections.

- **Instruction:** say what to find, not how to find it, and state every convention the tests enforce. Remove method hints. Test: with unlimited time, would this text and the data lead an independent expert to the right answer?
- **Reference:** it must solve the new task for real. If it took research code, hours or years, ship the output in `solution/data/` and record how it was made. Nothing the agent doesn't get may be needed to reach it.
- **Verifier:** real quantities, with windows chosen by the student. Write `authoring/evidence/calibrate.py`: run the reference, at least one other good method, doing nothing, the naive method and the wrong method you most expect. Good methods pass and wrong ones fail, with margin. Keep the results.
- **Leak test each lever.** Try to pass with a shortcut. If a check can be passed without the step it's meant to test, fix it now. Most agent wins on hardened tasks are leaks, not traps beaten fairly.
- **Keep what the tools expect:** the `harbor-canary` lines, the separate no-network verifier, and at most 18000 seconds for `timeout_sec`. Set that limit for the work and computing the agent has to do; it no longer follows the expert estimate. `uv run tools/hw.py check` runs the static checks.

Add to `authoring/attempts.md` under **Status** what you changed and why.

## Step 5: the second review

Back to a **new session** with the same prompt as Step 2, on the final version. Hardening is how tasks become unfair, so this one matters more than the first. Fix and repeat until the verdict is `solvable`. The report on the final version stays in `authoring/solvability.md`.

## Step 6: the pull request

Update the README's Difficulty, Reference solution and Verification sections, and `expert_time_estimate_hours`. Ask the student which AI helped and how, and that they've approved the instruction, window and publish steps again. Then:

```
uv run tools/hw.py submit --ai "<their answer>"
```

This opens a new pull request for the updated task, and GitHub checks that the reference still scores 1 and doing nothing 0. Check it passes.

The class then runs GPT-5.6 Sol on the task, one run at `xhigh` with web search off, and posts the result and its trace in the pull request. Solved means reward 1. The student needs no Docker or API key. A run can take up to the task's time limit, so tell them to submit early enough to act on the result.

**Want an earlier look?** If their ChatGPT plan includes Sol, you can set up a rough check without Docker:

1. Make an empty folder outside this repository. Copy in the agent-visible data and `instruction.md`, and nothing else: no `tests/`, `solution/`, `authoring/` or README.
2. In that folder, have the student run in a terminal: `codex exec -m gpt-5.6-sol -c model_reasoning_effort=xhigh --skip-git-repo-check "$(cat instruction.md)"`. Paths in the instruction may need adapting to the folder; say so if they do.
3. Run the verifier's checks on the output, adapting its paths.

This helps while iterating, but record it as a rough check: only the class's run counts as the result, because only it has the sealed container and the real limits.

## Step 7: read the attempts (CHECKPOINT)

Read each run in full before you believe its reward, and add a row to `authoring/attempts.md`.

- **Did it fail on the science?** Check that it had everything it needed. If it failed on an ambiguity, a missing tool or a window that rejects good work, that's a gap in the task: fix it and ask for another run. Name which failure mode you saw, and quote the moment.
- **Did it cheat?** Read for fetches of this repository, a look-up of the published answer, edited or special-cased checks. Fix the leak; the run doesn't count either way.
- **Did it solve it?** Find the decisive move, as in Step 1, and go back to Step 3. Use a different lever from the one it beat, then ask for another run.
- **Did it hold?** That's the floor reached. If the student wants more, ask in the pull request for a stronger model, such as Claude Opus 5.5 or GPT-6.1 Sol, or look at Terminal-Bench-Science's bar.

**CHECKPOINT:** the student agrees with your reading of each run. Set `class_level` in `task.toml`: 2 once the task is honest, 3 if Sol fails on the science across the runs you have.

## Step 8: finish

If you changed the task, have the student approve again, run `uv run tools/hw.py submit --ai "<their answer>"` to update the pull request, and have them ask there for another run. Tell them plainly what the task now measures, what the runs showed, and what you'd try next.

If Sol still solves it at the deadline, that still counts: record the run that beat it and the next idea.

**Encourage them to take it to Terminal-Bench-Science.** It's optional, and it's how this work can earn credit beyond the class: one merged task qualifies them for co-authorship on the paper, and submissions to version 2.0 close on December 15, 2026. They propose first, through the form on https://www.terminal-bench-science.ai/contribute, then build and open a pull request there if it's approved. Check the task against their [proposal rubric](https://github.com/harbor-framework/terminal-bench-science/blob/main/rubrics/task-proposal.md) and [contributing guide](https://github.com/harbor-framework/terminal-bench-science/blob/main/CONTRIBUTING.md), and tell the student plainly what would fall short. The proposal is theirs: they write it and they send it, and they declare that the task began as class homework.

## Hard rules

- Only change files under the student's own `week-1/submissions/<github-username>/` folder.
- Never put answers where the agent can see them, including file names and metadata.
- Never run `uv run tools/hw.py approve` for the student, feed it input or edit `authoring/progress.json`.
- Never edit a reward, fake an attempt, or claim a run you didn't do.
- Only use data the student may share publicly. Everything here is public and MIT-licensed.
- Don't peek: while you're the one being tested in a blind run, you must not read `tests/` or `solution/`. Blind runs go to a fresh model in a clean folder, never this session.
