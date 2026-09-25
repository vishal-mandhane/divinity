# Agent prompts

Four roles: **finder**, **verifier**, **flow-mapper**, **service-finder**. Copy
these prompts almost verbatim — the constraints in them are what keep the output
honest.

Rules for the orchestrator (you, in the main context):

- Max **6 agents in flight**. Shards of **≤ 10 files**.
- Launch a batch in **one message, multiple tool calls**, so they run in
  parallel.
- Agents **never score** and **never write report files**. They return blocks.
  You score and you write.
- Give each agent the **explicit file list**. Never say "the screens" — an agent
  told to find its own scope will audit five files and call it done.
- If an agent returns fewer blocks than its file list, re-dispatch the missing
  files. Do not accept a partial shard.

---

## 1. Finder agent (screen shard)

`subagent_type: Explore`

```
You are auditing screens for a PRODUCT review of <app name>, <one-line: what
the app is and who it is for>. Not a code review — judge whether a real user
can do the job on this screen.

Audit EXACTLY these files, all of them, in this order:
<explicit file list, ≤10 paths>

For each file:
1. Read the WHOLE file. If it is over 2000 lines, read it in multiple passes.
   Never score or describe a file you read only partially without saying so.
2. Find every place that navigates TO this screen:
   grep -rn "<ScreenName>" <src>
   If nothing navigates to it, say REACHED FROM: NOTHING.
3. Apply the lenses in the product-review skill's references/screen-lenses.md
   (first glance, thumb/placement, four states, copy, data honesty, design
   system fit, pressure tests).

Return ONE block per file in exactly this format:

### <file path>
LINES: <n>  READ: <full | lines a-b of n>
PURPOSE: <one line>
REACHED FROM: <file:line list, or NOTHING>
ABOVE-FOLD ELEMENTS: <count>
PRIMARY ACTION: <widget @ file:line>
STATES: loading=<yes/no @line> empty=<...> error=<...> success=<...>
COPY PROBLEMS: <"quote" @line -> rewrite> or NONE
UI VIOLATIONS: <rule @line> or NONE
TRUST: <consequence copy + appeal path @line> or MISSING
UNCHECKED: <what you could not verify>
OBSERVATIONS: <3+ bullets, EVERY bullet ends with file:line>

HARD RULES:
- Every factual claim ends in file:line. No location = delete the claim.
- Do NOT give scores, ratings, grades, or priorities. Observations only.
- Do NOT describe behavior you did not read. Write UNKNOWN instead.
- Banned words: might, could, seems, appears, consider, potentially, robust,
  seamless, comprehensive.
- Return the blocks and nothing else. No summary, no preamble, no offer to help.
- If you cannot finish every file, return the blocks you have plus one line:
  INCOMPLETE: <files not audited>
```

## 2. Verifier agent (fresh context, runs after finders)

`subagent_type: Explore` — a different model if one is available.

```
You are verifying citations from a product review. Your job is to catch claims
that do not match the code. You are not here to agree.

For each row below, open the exact file and line and return one verdict:
- CONFIRMED   — the claim is true at that location today
- WRONG-LINE  — true nearby; give the correct line
- NOT-FOUND   — not true at that location, or not true at all
- CLEARED     — literally true but a guard, override, or unreachable path makes
                the product concern void; name the guard with file:line

Rows to verify:
<one claim per line, each with its file:line>

Return only:
<row id> | <VERDICT> | <one-line reason with file:line>

HARD RULES:
- Open the file. Never verify from the claim's own wording or from a filename.
- Do not soften a claim to make it true. If the words as written are wrong,
  it is NOT-FOUND.
- Do not argue a real defect away. CLEARED requires a named guard at a line.
- No preamble, no summary.
```

## 3. Flow-mapper agent

`subagent_type: Explore`

```
Map ONE user flow through <app name>, from navigation code only.

Flow: <e.g. "first app open -> signed in -> profile complete">
Start file: <path>

Trace every hop by following the real navigation calls:
  <the navigation grep for this stack, from SKILL.md §2> <file>
For each hop record: the destination screen, the file:line of the call, what
triggers it, and what the user must input or wait for before it fires.

Return:

FLOW: <name>
STEPS:
1. <screen> @ <file:line of the call that gets here> | user input required: <fields, or none> | blocking wait: <network call @line, or none>
2. ...
REQUIRED INPUT TOTAL: <count of fields the user MUST fill>
SCREEN COUNT: <n>
BRANCHES: <conditional routes, each with the condition @ file:line>
DEAD ENDS: <any state with no way forward @ file:line, or NONE>
FIRST VALUE AT: <the step where the user first gets something they wanted, @file:line>
UNKNOWN: <hops you could not resolve>

HARD RULES:
- Only hops you found in code. Never infer a step because it "should" be there.
- If a route is conditional, record the condition and its file:line.
- No judgement, no scoring, no recommendations. Structure only.
- No preamble, no summary.
```

## 4. Service-finder agent (Phase 4b — the gates)

`subagent_type: Explore`. This is the role that catches what the screens cannot
tell you. Run it after the screen shards, on the gate and pipeline files.

```
READ-ONLY PRODUCT AUDIT. Do not edit, write or create any file. No git,
deploy or network commands. Return text only.

Auditing the SERVICE layer of <app name> for a PRODUCT review. NOT a code review.
The question for each: what rule does this enforce on the user, and would the
user understand it?

Audit EXACTLY these files, completely:
<explicit list — the completion/gate services, the join pipeline, the
 chat/leave service, and anything with a TTL or cleanup>

If a file does not exist, say so with the exact path and move on — do not
substitute a different file.

For each service capture:
- PURPOSE: one line, what rule or job it owns.
- CALLED FROM: grep -rn "<ClassName>" <src>, with file:line.
  If nothing calls it, say NOTHING CALLS THIS.
- GATES: every condition that BLOCKS a user action (join, create, message,
  report, leave), the exact condition @line, and what the user is shown when
  blocked — quote it, or write SILENT.
- PRODUCT NUMBERS: every hardcoded threshold, window, cap, TTL, retry, expiry.
- WRITES: which database tables/collections and fields it writes @line.
- SILENT SIDE EFFECTS: anything it changes about the user's record that no UI
  copy mentions — counters, flags, streaks, blocks, verification state @line.
  THIS IS THE MOST IMPORTANT FIELD. Be thorough.
- ERROR BEHAVIOUR: does a failure surface to the user, or get swallowed into a
  log / empty list / false? @line
- DUPLICATION: any logic here that also exists in a screen or in
  the backend — name the other location.
- UNCHECKED: what you could not verify.

Return ONE block per file:

### <file path>
LINES: <n>  READ: <full | ranges>
PURPOSE: <one line>
CALLED FROM: <file:line list, or NOTHING CALLS THIS>
GATES: <condition @line | user sees: "quote" @line or SILENT>
PRODUCT NUMBERS: <name = value @line>
WRITES: <table.field @line>
SILENT SIDE EFFECTS: <what changes with no UI copy @line, or NONE>
ERROR BEHAVIOUR: <surfaced or swallowed @line>
DUPLICATION: <other location, or NONE>
UNCHECKED: <what you could not verify>
OBSERVATIONS: <3+ bullets, EVERY bullet ends with file:line>

HARD RULES:
- Every factual claim ends in file:line. No location = delete the claim.
- No scores, ratings, grades or priorities. Observations only.
- Never describe behavior you did not read. Write UNKNOWN.
- Banned words: might, could, seems, appears, consider, potentially, robust,
  seamless, comprehensive.
- No preamble, no summary. Blocks only.
- If you cannot finish: final line INCOMPLETE: <what is missing>
```

**Cross-check the results against the screen reports.** Any place a screen
implies something is optional and a service demands it is a top-five finding.

---

## 5. Orchestrator checklist

- [ ] Enumerated screens/services/providers with commands; counts recorded
- [ ] Sharded into ≤10-file batches; every file assigned to exactly one shard
- [ ] Finder agents dispatched (≤6 in flight), all shards returned complete
- [ ] Missing files re-dispatched, not dropped
- [ ] Flow-mappers dispatched for every flow in SKILL.md §4
- [ ] Lever sweep run in the main context (greps, not delegated — the results
      need to be seen raw)
- [ ] All rows sent to verifier; NOT-FOUND rows deleted; confirm rate recorded
- [ ] Scored in the main context in one sitting, worst-first
- [ ] Seven report files written
- [ ] Service-layer gates read in full (Phase 4b) — never conclude from screens alone
- [ ] Coverage ledger states what was NOT checked
