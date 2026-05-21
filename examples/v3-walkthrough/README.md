# v3 Walkthrough — directory-watcher CLI

Status: **draft scenario**. Demonstrates AIF v3 mechanics. Not normative.

This walkthrough exercises the `DESIGN-v3-draft.md` rules end-to-end on a
small project: build a Python CLI that watches a directory and POSTs file
change events to a webhook.

## Scenario topology

```
USER ──NLU──▶ main ──AIF──┬──▶ agent-pm@opus-4.7        (planner)
                          ├──▶ agent-rd-a@sonnet-4.6    (impl stream A)
                          ├──▶ agent-rd-b@sonnet-4.6    (impl stream B)
                          ├──▶ agent-reviewer@opus-4.7  (code review)
                          ├──▶ agent-rd@sonnet-4.6      (fixer)
                          └──▶ agent-pm@opus-4.7        (final review)
```

## Message map

User originally described the cycle as 10 messages. With the locked v3
rules it expands to **14**:

| # | TYPE     | From → To                     | Reason for added msg          |
|---|----------|-------------------------------|-------------------------------|
| 1 | TASK     | main → pm                     |                               |
| 2 | DELIVER  | pm → main                     |                               |
| 3 | TASK     | main → rd-a                   | split: 1 sub = 1 TASK         |
| 4 | TASK     | main → rd-b                   | split (was msg 3 combined)    |
| 5 | DELIVER  | rd-a → main                   |                               |
| 6 | DELIVER  | rd-b → main                   |                               |
| 7 | TASK     | main → reviewer               | invariant: all spawns logged  |
| 8 | DELIVER  | reviewer → main               |                               |
| 9 | SUMMARY  | main → main (msgs 5,6)        | invariant: summaries logged   |
|10 | TASK     | main → rd (fixer, sonnet)     |                               |
|11 | DELIVER  | rd → main                     |                               |
|12 | SUMMARY  | main → main (msgs 1,2,10,11)  | invariant: summaries logged   |
|13 | TASK     | main → pm (final review)      |                               |
|14 | DELIVER  | pm → main                     |                               |

## Things to notice while reading `handoff_book.md`

1. **All messages use the v3 header schema** (§3 of `DESIGN-v3-draft.md`):
   `# Title; ## Description; *TO:*; *FROM:*; *TYPE:*; *DATE:*; *ID:*;
   *IN-REPLY-TO:*; *MESSAGE:*`.
2. **`IN-REPLY-TO:` rule applied**:
   - Present for: every DELIVER, fixer TASK (msg 10 replies to review), final
     PM TASK is omitted (it derives from a SUMMARY, not a single prior msg).
   - Absent for: the initial TASK (msg 1), spawn-with-fan-in TASK (msg 7
     and msg 13), and SUMMARY msgs.
3. **`INPUTS:` carries fan-in** when `IN-REPLY-TO:` can't (single value).
   See msg 7 (`INPUTS: [msg-5, msg-6]`) and msg 13 (`INPUTS: [msg-12]`).
4. **Pure forward is still logged** (msg 7): no new instruction content
   beyond INPUTS + role-implied behavior, but the TASK exists in the book.
5. **SUMMARY messages** (msg 9, msg 12) are `main → main` with
   `SOURCE_IDS:` listing the messages summarized. They are real entries
   main can re-read, not transient prompt scaffolding.
6. **PM verdict is PARTIAL** because original plan had 5 milestones but
   only M1-M3 went through dispatch. The walkthrough stops at "phase 1
   closed", which is the same place the user's original scenario stopped.

## Body schemas

TASK / DELIVER / SUMMARY body shapes are now **normative**, defined in
`DESIGN-v3-draft.md §4`. The walkthrough's bodies conform to those
schemas. QUERY / CLARIFY bodies are still pending (see `§7 item 1`).

### Message terminator

`===END_OF_MESSAGE===` on its own line — also **normative** (see
`DESIGN-v3-draft.md §3` Message terminator subsection).

## Mini-walkthrough: QUERY / CLARIFY

See `query_clarify.md` for a 4-message scenario where the PM finds the
original goal ambiguous and asks main for a decision before producing
the plan:

| # | TYPE     | From → To       | Note                                          |
|---|----------|-----------------|-----------------------------------------------|
| 1 | TASK     | main → pm       | plan a directory-watcher CLI                  |
| 2 | QUERY    | pm → main       | `BLOCKING: true`; dotfile handling unclear    |
| 3 | CLARIFY  | main → pm       | answers FIELDS in QUERY                       |
| 4 | DELIVER  | pm → main       | `IN-REPLY-TO` = msg 1 (not msg 3)             |

Key invariants exercised:
- QUERY's `IN-REPLY-TO` points to the **TASK** the sub is working on
- CLARIFY's `IN-REPLY-TO` points to the **QUERY**
- DELIVER's `IN-REPLY-TO` still points to the **original TASK** —
  QUERY↔CLARIFY is a side-branch, the TASK→DELIVER main chain is
  unaffected
- `BLOCKING: true` means the PM holds the plan until receiving CLARIFY;
  `BLOCKING: false` would mean PM proceeds with a best-guess decision
  and surfaces the assumption in DELIVER notes

## Token cost (rough)

`handoff_book.md` for this scenario ≈ 11 KB / ~3,000 tokens. Main agent's
working state at any given step is much smaller because most messages
live on disk and are re-read by ID rather than held in context.
