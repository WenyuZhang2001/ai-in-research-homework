---
name: check-solvability
description: Independently review one Harbor benchmark task for solvability - does a right answer exist, do the instruction and data determine it, and does a right answer reliably pass and a wrong one fail? Checks hidden conventions, stated versus hidden ranges, an honest reference solution, fresh draws or noise margins, whether the answer is identifiable and the key defensible, shortcuts and leaks, and resources; ends with a solvable or not_solvable verdict, the smallest fix, and authoring/solvability.md. Use for the week 6 homework in UCLA's "AI in Research" class, or whenever asked to check a task's solvability or fairness. Run it in a fresh session that did not help build the task.
---

# Is this task solvable?

A task is **solvable** when a right answer exists, the instruction and data determine it, a right answer reliably passes and a wrong one reliably fails. It doesn't have to be quick for a person: these tasks aim at more than a year of expert time, ideally ten. So never ask whether an expert could finish it soon. Ask whether unlimited expert time, with only what the agent gets, would lead to the right answer. A task that frontier agents fail tells you nothing until you know it is solvable.

## Your role

You are the independent reviewer of one task. The path is `week-1/submissions/<github-username>/<task-name>/` unless you're told otherwise.

- **You did not build this task.** If you helped build it in this session, say so and ask for a new session.
- **Don't trust the author's description.** Treat the README and `authoring/` as claims. Check them against `instruction.md`, `environment/`, `solution/` and `tests/`.
- **Don't change the task.** Scratch work goes in a temporary folder outside the repository, which you delete afterwards. The one file you write is `authoring/solvability.md`.
- **Separate what you ran from what you read.** The student may have no Docker. Run what you can with plain Python or `uv`. If you can't run a check, answer `unclear` and say why; never report a run you didn't do.

Don't try to solve the task, and don't judge whether it's hard enough. If a check depends on the field's knowledge and you aren't sure, ask the student a specific question and record the answer.

## The checks

Answer each one **yes**, **no** or **unclear**, with one reason and a file path or number as evidence.

### 1. Information
Is everything the tests need in the instruction, the data or standard knowledge of the field? Read every assertion in `tests/` and ask where a solver would learn it. Look for units, sign conventions, column names, file formats, tie-breaking rules and required precision that the tests enforce and the instruction doesn't state.

### 2. Stated and hidden ranges
Does every range, limit, unit and condition in the instruction cover the hidden data? The hidden set must never need something outside what's stated, and the stated range must not invite work the hidden set never rewards.

### 3. An honest reference
Does the reference solution get its answer from the agent's inputs? Read `solution/` for any use of `tests/`, truth files, generator seeds, `authoring/` or precomputed answers. Then copy the task to a scratch folder, delete `tests/` and `authoring/` from the copy, run the reference on the agent-visible inputs alone, and score its output with the original verifier.

The class allows a shipped reference when the real work took hours, years or research code. In that case, check that the task records how the output was made, and that no private data or unstated input went into it that the agent doesn't get. Given enough time, it must be reachable from what the agent gets. Say which case this is.

### 4. A right answer passes reliably
Pick the case that fits:
- **Generated data:** if the generator is available, draw at least 10 new seeds, regenerate, and run the reference and verifier on each. The reference should pass at least 95% of the time. Subsample if you must, and say so.
- **Real data:** do the pass windows sit outside the noise? Look for `authoring/evidence/calibrate.py`, then perturb the reference within the measurement's uncertainty or use a second defensible method. A correct method must not fail by bad luck.

### 5. A wrong answer fails reliably
Does a no-skill answer fail: doing nothing, a constant, a climatology, the textbook default, the simplest method? Compare the pass bar with those baselines. A bar that a weak method clears says little (one class task accepted an R² of 0.02). Try at least one wrong method a tired expert might use. Passing must mean the science was done.

### 6. Identifiable
Can the hidden structure be recovered from what the solver sees? If two different answers explain the data equally well and only the grader can tell them apart, the task is not solvable. For each thing withheld, name the evidence in the data that points to it.

### 7. A defensible key
Would a second expert's answer pass? Look for keys built on one person's unstated choices.
- **Detection or census:** rank every candidate in the data by strength and see where the expert's picks fall. A continuum with no gap, or picks that skip strong candidates, means the count is a judgement call.
- **Cleaning or reconstruction:** check for invented data in the reference, such as straight-line bridges, that a good method would reasonably fill differently.
- **Fix:** state the criterion and apply it to the key, or widen the tolerance around borderline cases.

### 8. Not misleading by construction
A task may offer tempting wrong paths, as real science does. It may not make the obvious self-check reward the wrong model while the data hold no clue against it. If the only way to pass is to distrust every available check for no visible reason, it's a trap.

### 9. One fact doesn't decide it
Does the score turn on one obscure fact or a formula the solver either happens to know or doesn't? That makes the task a coin flip on memory. Expert knowledge is fine when the data let a careful solver test the idea.

### 10. No shortcut, no leak
- Are answers absent from `environment/`, `instruction.md`, file names, metadata and timestamps?
- Can the verifier be satisfied without the work? Look for checks that only test a file's existence or format, and answers that can be derived from the test's own constants.
- **The agent has the internet.** This repository is public, so `tests/`, `solution/` and the README can be fetched. Is the answer also in a paper, a catalogue or a public dataset the agent could look up? Say what a run's trace should be checked for.

### 11. Limits
Does the computation a good method needs fit in `timeout_sec`, the CPUs, memory and storage in `task.toml`? Human time can far exceed the agent's limit, but computing time can't: a run that needs a month of CPU is not solvable in a 5-hour container. Does the instruction say what the solver needs to know about the limits? Can a real solver install and run what it needs, with no private network, license or account? Does the README say what `expert_time_estimate_hours` is based on? Record whether that time is thinking or repetition; repetition doesn't make a task hard, but it doesn't make it unsolvable either.

## The verdict

- **`solvable`**: every check is yes, or unclear for a stated reason that doesn't hide a real doubt.
- **`not_solvable`**: any check is no.

For `not_solvable`, give the **smallest concrete change** that makes the task fair while keeping what makes it hard. Don't redesign it.

## Write the report

Write `authoring/solvability.md`:

```markdown
# Solvability review

Reviewed: <date>, commit <short hash>, by <model and tool>, in a session that did not help build the task.
**Verdict: solvable | not_solvable**

| # | Check | Answer | Reason and evidence |
|---|---|---|---|
| 1 | Information | yes | ... |
| ... | | | |
| 11 | Limits | yes | ... |

## What I ran
Each command or script, and its result. What I could not run, and why.

## Problems
Each problem, most serious first.

## Smallest fix
The change that would make it solvable, or "none".

## What to check in a run's trace
For example: fetches of this repository, of a named paper or of a public catalogue.
```

End by telling the student the verdict in one sentence and the first thing to fix. An earlier report for the same task gets replaced; `git` keeps the history.
