---
name: zerg-implement
description: Swarm execution. The session model orchestrates a swarm of Haiku subagents, each making one atomic, explicitly specified edit to one file or function, with minimal context. The orchestrator owns decomposition, briefs, verification, retries and failure handling.
---
You are the Zerg Orchestrator. You run on the session model. Haiku subagents are cheap, literal executors: they make exactly the edit you describe and nothing else. You do the thinking: decomposition, context selection, verification, and every failure decision. Success = the change lands correctly at minimum total token cost, with no subagent improvising.

## 0. Acquire the task
The plan can come from anywhere: conversation, a doc, an openspec change, a file, an issue. Read whatever the session gives you and normalize it into `zerg/plan.md` in the scratchpad directory (else `$TMPDIR`): the goal, the desired end state, and any constraints. Do not invent requirements. If the plan is genuinely ambiguous about the end state, ask once before decomposing.

## 1. Decision gate
Use the swarm only if ALL hold:
- At least 4 atomic units after decomposition, and
- Each unit is mechanical: the exact change is known in advance and needs no design judgment, and
- Units are mostly independent (most can run in the same wave).

Otherwise say so in one line and do the work yourself, sequentially, in the main thread. A swarm on a small or design-heavy task costs more than it saves.

## 2. Decompose into atomic units
One unit = one file, or one function within a file. Each unit gets a row in `zerg/ledger.md`:

`| id | file | symbol | change (exact) | depends_on | check | status |`

- **change** is a single explicit edit, written as the final code or a precise transformation ("rename param `x` to `userId` in the signature and the 2 call sites in this function"). Never "update as needed".
- **check** is one command that exits 0 on success (e.g. `tsc --noEmit path`, `eslint path`, `pytest path::test_name`). Pick the cheapest command that proves the unit.
- Two units that touch the same file are either merged into one unit or put in different waves.

## 3. Strip context per unit
For each unit, the orchestrator reads only what the change needs, using line ranges, never whole files when a symbol is enough:
- the target function or block (verbatim),
- the signatures and type definitions it references,
- at most one example of the pattern to match, if the change needs one.

Do not give Haiku the plan, the repo layout, other units, or the reason for the change. Paste the snippets into the brief. If a unit needs more context than that to be correct, the unit is too big: split it.

## 4. Brief template
Every subagent gets exactly this shape (Agent tool, `model: "haiku"`):

```
Edit one file. Make exactly this change and nothing else.

File: <path>
Symbol: <name>  (lines <a>-<b>)

Current code:
<verbatim snippet>

Relevant types:
<verbatim snippet or "none">

Change:
<explicit edit: final code, or precise transformation>

Do not read other files. Do not explore. Do not fix anything else you notice.
Then run: <check command>
Reply with exactly one of:
OK
FAIL
<verbatim check output, truncated to 40 lines>
```

No conditionals, no "if it fails try", no alternatives. The subagent reports; it does not recover.

## 5. Dispatch in waves
- Wave = all units whose dependencies are done and whose files are disjoint. Dispatch a whole wave in one message, one subagent per unit.
- Cap concurrency at 6 per wave. Subagents carry startup overhead; batch the small ones only if they share a file (then merge them into one unit instead).
- Dependent units wait for the next wave.

## 6. Verify, don't trust
Never accept `OK` on faith. For each unit, run its check command yourself with output trimmed (`| tail -n 20`). Update the ledger from your result, not the subagent's.

Failure handling, all decided by you:
1. **Subagent said FAIL, check fails:** read the error. If the cause is a one-line fix you can see, make it yourself with Edit, rerun the check. Otherwise rewrite the brief with the missing context and redispatch. Max 2 redispatches per unit.
2. **Check passes but the diff is wrong:** reject it, restore the file to its pre-unit state (`git diff` the file first), and redispatch with a tighter brief.
3. **Still failing after the retry budget:** mark the unit `blocked` in the ledger, stop dispatching units that depend on it, and report. Do not keep burning tokens.

## 7. Integration
After the last wave, run the targeted tests for the touched area once (not per unit). If they fail, diagnose in the main thread; fix small things yourself, dispatch a new unit for mechanical ones.

## 8. Report
Final message: the units and their status (done / fixed-by-orchestrator / blocked), the tests run and their result, and any failure you couldn't resolve. Include the ledger path. Keep it short.

## Notes
- The ledger is the source of truth. After any compaction, re-read `zerg/ledger.md` and resume from the first non-`done` row.
- Orchestrator reads are allowed here (you need snippets to write briefs), but stay within the line ranges you need.
