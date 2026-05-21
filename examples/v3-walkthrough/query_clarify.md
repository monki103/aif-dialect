# Title: Plan project: directory-watcher CLI;
## Description: non;
*TO:* agent-pm@opus-4.7;
*FROM:* main-agent;
*TYPE:* TASK;
*ID:* 20260521-100000-00;
*MESSAGE:*
```json
{
  "goal": "A Python CLI that watches a directory and POSTs file-change events to a configurable webhook URL.",
  "constraints": [
    "python>=3.11",
    "single-file install (pipx friendly)"
  ],
  "deliverable": {
    "format": "milestone-list",
    "required_fields": ["id", "name", "depends_on", "est_effort_hours", "acceptance_criteria", "stream"],
    "acceptance": "each milestone <= 4h, RD-implementable"
  }
}
```
===END_OF_MESSAGE===

# Title: QUERY: dotfile handling for watcher;
## Description: blocking question before finalising plan;
*TO:* main-agent;
*FROM:* agent-pm@opus-4.7;
*TYPE:* QUERY;
*ID:* 20260521-100045-00;
*IN-REPLY-TO:* 20260521-100000-00;
*MESSAGE:*
```json
{
  "question": "Should events fire on hidden files (dotfiles) and inside directories starting with a dot?",
  "blocking": true,
  "fields": ["dotfile_emit_events", "dotdir_recurse"],
  "context_needed": [
    "watchdog library defaults to emitting events for all paths; the plan needs an explicit project decision so M2 acceptance criteria can be written"
  ]
}
```
===END_OF_MESSAGE===

# Title: CLARIFY: dotfile handling for watcher;
## Description: non;
*TO:* agent-pm@opus-4.7;
*FROM:* main-agent;
*TYPE:* CLARIFY;
*ID:* 20260521-100110-00;
*IN-REPLY-TO:* 20260521-100045-00;
*MESSAGE:*
```json
{
  "answer": {
    "dotfile_emit_events": false,
    "dotdir_recurse": false
  },
  "notes": "expose --include-hidden CLI flag (default false) so users can opt in."
}
```
===END_OF_MESSAGE===

# Title: Milestone plan for directory-watcher CLI;
## Description: non;
*TO:* main-agent;
*FROM:* agent-pm@opus-4.7;
*TYPE:* DELIVER;
*ID:* 20260521-100330-00;
*IN-REPLY-TO:* 20260521-100000-00;
*MESSAGE:*
```json
{
  "status": "COMPLETED",
  "summary": "5 milestones; dotfile handling per CLARIFY 20260521-100110-00 (off by default, --include-hidden opt-in)",
  "result": {
    "milestones": [
      {
        "id": "M1",
        "name": "project skeleton + --include-hidden flag",
        "depends_on": [],
        "est_effort_hours": 1,
        "acceptance_criteria": "pipx install + --help lists --include-hidden",
        "stream": "A"
      },
      {
        "id": "M2",
        "name": "watcher core (hidden filter applied unless --include-hidden)",
        "depends_on": ["M1"],
        "est_effort_hours": 3,
        "acceptance_criteria": "unit tests cover dotfile-excluded and --include-hidden paths",
        "stream": "A"
      },
      {
        "id": "M3",
        "name": "webhook client (httpx async POST with retry+backoff)",
        "depends_on": ["M1"],
        "est_effort_hours": 3,
        "acceptance_criteria": "unit tests mock endpoint; retry on 5xx, no retry on 4xx",
        "stream": "B"
      },
      {
        "id": "M4",
        "name": "wire watcher + webhook + CLI flags",
        "depends_on": ["M2", "M3"],
        "est_effort_hours": 2,
        "acceptance_criteria": "integration test ends with mocked webhook receiving POST",
        "stream": "merge"
      },
      {
        "id": "M5",
        "name": "env-var config + README + smoke test",
        "depends_on": ["M4"],
        "est_effort_hours": 1,
        "acceptance_criteria": "WATCHER_URL env var respected; smoke test passes",
        "stream": "final"
      }
    ],
    "clarifications_applied": [
      {
        "source_query": "20260521-100045-00",
        "source_clarify": "20260521-100110-00",
        "decision": "dotfiles excluded by default; --include-hidden opt-in"
      }
    ]
  }
}
```
===END_OF_MESSAGE===
