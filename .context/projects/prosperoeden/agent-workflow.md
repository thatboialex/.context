# Agent workflow for ProsperoEden phases

How a thread runs one phase. Docs are the memory: a thread must never need chat history.

## Rules

1. One thread per phase (or per tightly related group the plan names). Read, work, test, review, hand off, stop. Do not carry on into the next phase.
2. Read in this order, nothing else first: `projects/prosperoeden/README.md`, your phase file, `findings.md` (once phase 0 exists), then the code files the phase names. Load `ai-rules.md` and `glossary.md` only as `CLAUDE.md` says; this project's README rules override the web-service defaults.
3. Work in the branch the coordinator gives you. Code goes in `thatboialex/ProsperoEden`, docs and status in `thatboialex/.context`. Open draft PRs; mark ready when the done criteria pass.
4. Pick the cheapest tier that fits (below). If a Haiku or Sonnet thread hits something the phase file marks **escalate**, stop, write what you found in the phase file's "Notes for the next session", and tell the coordinator. Do not guess.
5. Before finishing: update the phase file (state, what changed, what is unverified), the plan's status table, and any doc the change makes wrong. In the same PR.
6. Say plainly what was not run. "Host tests pass; console unverified" is a complete, honest status.

## Tiers

| Tier | Use for | Do not use for |
|---|---|---|
| Haiku | Docs from known facts, changelog lines, log-format edits, renames, copying a pattern already in the repo | Anything needing a decision, anything touching Eden patches or threads |
| Sonnet | Normal C++/Python changes in `headless/` and `tools/`, host tests, launcher work, reviewing a Haiku PR, reading Eden source for findings | Debugging a hang or crash that logs do not explain |
| Opus | Reading JIT/kernel/service code to explain a failure, designing an Eden patch, performance analysis, reviewing an Eden patch | Mechanical edits, docs |

A thread may start as Sonnet and hand an **escalate** item to a separate Opus thread. Reviews: a Sonnet thread reviews Haiku PRs; an Opus thread reviews every Eden patch and anything on a hot thread (emulation, GPU, audio). A review is a fresh thread with the PR diff and the phase file, not the author's chat.

## Testing

| Level | How | Who |
|---|---|---|
| Host | `make test` (Linux). Add a `*_check.cpp` or `tools/check-*.py` for new logic, as the repo does (`headless/mods_check.cpp` is the model) | Any thread, if the cloud container can build; phase 0 records whether it can |
| Static | Read the diff against the phase's done criteria; run `validate-context` on doc changes | Reviewer |
| Console | [hardware-test-protocol.md](./hardware-test-protocol.md), results posted by Jason in the thread | Jason |

## Concurrency

Threads in the same wave may share a prompt cache, but each still reads only its own docs. If two threads need the same file, the earlier owns it; the later rebases. Never edit another phase's file except its status line.

## Handoff block

End every phase with this at the bottom of its phase file:

```
## Result
State: Not started | Draft PR | Merged, unverified | Closed (console-verified)
PR: <link>
Changed: <files, one line each>
Unverified: <what only the console can show>
Notes for the next session: <facts the next phase needs, not chat history>
```
