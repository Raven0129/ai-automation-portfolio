# Jira Automation System — Python Replacement for Native Automation Rules

Python-based replacement for native Jira Automation rules on a high-volume project. Native Jira Automation caps branch execution at ~1,000 issues per branch, but this project's parent tickets can have thousands of subtasks — so ticket classification, assignee assignment, and daily reset/recycle are handled here instead, using `nextPageToken`-paginated REST calls with no upper bound.

Repo: `<your-org>/<your-group>/<your-repo-name>`
Working folder: `<your-working-folder>/`

---

## Contents

| File | Purpose |
|---|---|
| `0-confluence_pools.py` | Shared module — reads assignee pools live from Confluence (POC / Analyst / Approver lists) |
| `1-parent_assignment.py` | Counts subtasks, classifies parent ticket by size tier, assigns a POC |
| `2-autocreation_assignment.py` (Flow A) | Assigns unassigned "auto-creation approval" subtasks |
| `3-manual_review_assignment.py` (Flow B) | Assigns unassigned review-request subtasks |
| `4-reassign_customer_review.py` (Flow E) | Reassigns unassigned subtasks stuck in an approval status |
| `5-reset_recycle.py` (Flow D) | Daily reset/recycle of subtasks (replaces native automation's 1,000-issue cap); exposes `run()` for a single ticket and `reset_all()` for every in-progress parent |
| `main.py` | Orchestrator — `--reset`, `--assignment`, and `--scheduled` modes |
| `reporter.py` | Excel report generation for every flow |

---

## How it fits together

```
                ┌────────────────────────┐
   1:00 AM      │  main.py --reset       │   (Flow D, via main.py orchestrator)
   daily        │  reset_all() resets/   │
                │  recycles subtasks for │
                │  every in-progress     │
                │  parent ticket         │
                └───────────┬────────────┘
                            │
   3:00 AM      ┌───────────▼────────────┐
   (currently   │  main.py --scheduled   │
   <cadence>,   │  for each in-progress  │
   may move to  │  parent ticket:        │
   daily — see  │    Flow A → B → E      │
   open items)  └────────────────────────┘

   Every 2 hrs  ┌────────────────────────┐
                │  main.py --assignment │
                │  for each "To Do",     │
                │  unassigned parent:    │
                │    1-parent_assignment │
                └────────────────────────┘
```

- **Flow D (1:00 AM)** must finish before Flows A/B/E run at 3:00 AM. The buffer between these two runs is currently unconfirmed as sufficient for full-volume subtask processing — open item.
- **Flow D only scopes to in-progress parent tickets** — not "To Do". A "To Do" parent in this project's workflow is a container/epic-type ticket that hasn't been through subtask assignment yet, so structurally it can't have subtasks in a reviewable state for Flow D to reset. (In-progress parents are one issue type; "To Do" parents are another — confirmed by direct JQL testing. Your project's issue-type split may differ.)
- **Assignment run** (every 2 hrs) picks up new/unassigned parent tickets and classifies + assigns them before Flow D/A/B/E ever touch their subtasks.

---

## Classification logic (`1-parent_assignment.py`)

- Counts all subtasks under the parent via `parent = <TICKET_KEY>` JQL, paginated with `nextPageToken`.
- **Threshold: `<YOUR_THRESHOLD>` subtasks** → above threshold = larger tier, below = smaller tier. (Example used in this project: 200.)
- Writes back:
  - `<YOUR_SUBTASK_COUNT_FIELD_ID>` — Count of Sub-tasks (number field)
  - `<YOUR_CLASSIFICATION_FIELD_ID>` — classification label (single-select dropdown; requires `{"value": "..."}` format, case-sensitive)
- Assigns the parent to the next person in the matching **POC** pool (round-robin).

All downstream flows (A, B, E) read the classification field off the parent to decide which Confluence pool to pull from. If it's empty, they fail loudly rather than guessing.

---

## Flow reference

| Flow | Script | Subtask filter | Action | Status after |
|---|---|---|---|---|
| Parent Assignment | `1-parent_assignment.py` | n/a — operates on parent | Classify + assign POC | n/a |
| A | `2-autocreation_assignment.py` | summary matches "auto-creation approval" pattern AND unassigned | Assign approver | Approval status |
| B | `3-manual_review_assignment.py` | summary matches review-request pattern AND NOT auto-creation pattern AND unassigned | Assign analyst | Analysis status |
| E | `4-reassign_customer_review.py` | subtask AND summary matches review pattern AND status = approval status AND unassigned | Assign approver | Approval status (unchanged) |
| D | `5-reset_recycle.py` | 3 branches per parent ticket (see below) | Reset/recycle | — |

**Flow D's three branches** (all scoped to a single parent ticket, run via `run()` for one ticket or `reset_all()` for every in-progress parent):
| Branch | Subtask filter | Action |
|---|---|---|
| 1 — Analysis Reset | `status = "<ANALYSIS_STATUS>"` | Clear assignee → transition to "To Do" |
| 2 — Auto-Creation Approval Reset | `status = "<APPROVAL_STATUS>"` AND summary matches auto-creation pattern | Clear assignee → transition to "To Do" |
| 3 — Customer Review Recycle | `status = "<APPROVAL_STATUS>"` AND summary matches review pattern | Clear assignee only (status stays unchanged) |

**Quota (Flows A/B/E):** auto-calculated, not dialog-based —
`quota = pool_size × hours_allotted × target_per_hour`
If fewer unassigned subtasks exist than quota, all available ones are assigned and a warning is printed; nothing fails.

**Round-robin state:** persisted in a shared `rr_state.json`, one key per flow × classification tier. State only advances on a *successful* write — a failed assignment rolls the index back so the next run retries the same person.

---

## Confluence-backed assignee pools (`0-confluence_pools.py`)

Pools are **not** hardcoded in `.env` — they're read live from a Confluence page (page ID `<YOUR_CONFLUENCE_PAGE_ID>`) so team leads can edit membership without touching code.

- Fetched via `GET /wiki/api/v2/pages/{page_id}?body-format=storage` (Basic Auth, direct site URL — **not** the OAuth `api.atlassian.com/ex/confluence/` path).
- **Must use `storage` format, not `markdown`** — some Confluence Cloud sites return HTTP 400 for markdown on the v2 API; storage format is universally supported.
- Pools are matched by heading text (H1–H6, case/whitespace-tolerant) immediately followed by a table with `Display Name` / `Account ID` columns.
- Pool name constants: define one heading per tier × role (e.g. `Small Watchlist POC`, `Big Watchlist Approver List`) matching your own classification/role scheme.
- If a heading, table, or the Confluence page itself can't be resolved, the module raises a custom pool error and the calling flow **fails loudly** — no silent fallback to an empty or stale pool.
- A `get_pool_with_names_by_label()` helper returns full `{"display_name", "account_id"}` rows instead of just IDs, so console/report output shows a real name instead of a raw account ID string.

---

## Auth & environment

| Variable | Required | Notes |
|---|---|---|
| `JIRA_URL` | hardcoded | e.g. `https://<your-site>.atlassian.net` — not read from `.env` |
| `JIRA_USERNAME` | yes | Atlassian account email, runtime env var injected by CI/CD |
| `JIRA_API_TOKEN` | yes | Jira API token, runtime env var injected by CI/CD |
| `RR_STATE_FILE` | no | Defaults to `rr_state.json` next to the scripts |
| `REPORT_DIR` | no | Defaults to `reports/` next to the scripts |

- Auth is **Basic Auth** (username + API token) — OAuth 3LO is retired for the running scripts, though some docstrings referencing OAuth scopes are stale leftovers and should be cleaned up.
- `python-dotenv` truncates colon-separated `.env` values after the first token — don't rely on it for OAuth scope strings; hardcode instead if ever needed.

### Known auth gotcha
Jira Cloud returns **404** (not 401) on valid ticket keys when the API token is expired or invalid — it masks auth failures as "not found." If a script suddenly can't find a ticket it found yesterday, check the token first.

---

## Reporting (`reporter.py`)

Every flow run produces a timestamped `.xlsx` report in `reports/` (or `$REPORT_DIR`):

- `ParentAssignment_<timestamp>.xlsx`
- `FlowA_<timestamp>.xlsx`, `FlowB_<timestamp>.xlsx`, `FlowE_<timestamp>.xlsx`
- `ScheduledRun_<timestamp>.xlsx` / `AssignmentRun_<timestamp>.xlsx` (orchestrator-level summaries from `main.py`)

Styling: header color, alternating row color, success/failure row highlighting — customize to your own brand palette.

**Open item:** report delivery method (CI artifacts vs. shared drive vs. wiki vs. email) is a design decision left to the implementer.

---

## Orchestrator (`main.py`)

```
python main.py --reset         # daily: in-progress parents → Flow D reset_all()
python main.py --assignment    # every 2 hrs: To Do + unassigned parents → 1-parent_assignment.py
python main.py --scheduled     # currently <cadence>; may move to daily — see open items
                                # after Flow D: in-progress parents → Flow A → B → E
```

- Dynamically loads sibling flow scripts via `importlib`, registering each in `sys.modules` first (required so each script's internal `sys.modules[__name__]` lookups resolve).
- Calls each flow's `run(ticket_key, jira_url, auth)` entry point — **flow scripts must expose `run()`, not just `main()`.**
- `--reset` loads `5-reset_recycle.py` and calls its `reset_all()`, which fetches its own ticket list and saves its own per-ticket reports — `main.py` doesn't duplicate that logic, just delegates.
- JQL for ticket fetchers excludes container/epic-type issues and subtasks — adjust the exact issue-type filter to match your own project's scheme.
- Status names must match your Jira workflow's exact configured names (including any project-specific prefixes).

### ⚠️ Known bug — `subTaskIssueTypes()`
`subTaskIssueTypes()` in JQL was found to be unreliable for this project's issue type scheme and is suspected to cause zero-result bugs in dry-run/audit scripts. Prefer explicit filters like `issuetype != Sub-task` instead. **Status: unresolved** — last known state was testing the JQL directly in Jira's own search UI. This may or may not reproduce on other Jira Cloud instances/schemes — worth testing before assuming it applies to your setup.

### ⚠️ Known bug — multi-value `status IN (...)` clauses
A multi-value `status IN ("A", "B")` clause was found to silently return **zero results** against this particular Jira instance, even when an equivalent equality query (`status = "A"`) against the exact same data returned real matches. Confirmed directly: a query for `parent = <KEY> AND status = "<STATUS_A>"` returned hundreds of matches for a ticket, while `parent = <KEY> AND status IN ("<STATUS_A>", "<STATUS_B>")` on the same ticket returned zero.

**Fix: avoid `IN (...)` on the `status` field. Use OR'd equality instead:**
```
status = "<STATUS_A>" OR status = "<STATUS_B>"
```
**Status: worked around, not root-caused** — unclear whether this is a Jira Cloud `/search/jql` beta-endpoint quirk or something specific to this instance's config. Test against your own instance before assuming it applies.

---

## Pagination rules (applies everywhere)

- `/rest/api/3/search` is **deprecated** (returns 410) — always use `/rest/api/3/search/jql`.
- No `startAt` / `total` field on `/search/jql` — page with `nextPageToken` + `isLast` only. Do not compare running counts to a "total."

---

## Diagnostics / dry-run tooling

Not part of the production cronjob chain — for manual pre-checks before running a real reset.

- **`flowd_dry.py`** — read-only preview of Flow D. Runs the exact same 3 branch queries as `5-reset_recycle.py` but only counts matches instead of clearing assignees/transitioning statuses. Prints per-ticket output in the same `"Flow D — Summary (TICKET)"` format the real flow uses, plus a grand-total summary and an Excel report. Defaults to scanning in-progress parents (matching Flow D's real scope). Has a `--debug` flag that prints the exact JQL used plus a set of probe queries and an auth/identity check (`/rest/api/3/myself`) — useful if a query starts returning unexpectedly empty results.
  ```
  python flowd_dry.py                 # preview against in-progress parents
  python flowd_dry.py --debug          # + diagnostic probes
  ```

**Recommended flow:** run `flowd_dry.py` first to see what a real `--reset` run would touch → then run `main.py --reset` (or `5-reset_recycle.py` directly) to actually apply it.

---

## Git workflow

Never push directly to `main` — have a designated reviewer merge via MR/PR.

```bash
git add .
git stash
git checkout -b <your-feature-branch>
git stash pop
git add .
git commit -m "..."
git push origin <your-feature-branch>
# open MR/PR into main
```

If the branch already exists locally: `git branch -D <your-feature-branch>` first.
Run all git commands from the repo root, **not** from the working subfolder.

---

## Open items / on the horizon

- [ ] Resolve `subTaskIssueTypes()` JQL reliability issue for dry-run/audit scripts
- [ ] Root-cause the multi-value `status IN (...)` JQL bug (currently just worked around with OR'd equality)
- [ ] Finalize report delivery approach (CI artifacts / shared drive / wiki / email)
- [ ] Confirm the reset→reassignment buffer window is sufficient for full-volume subtask processing daily
- [ ] Confirm whether the reassignment flows (A/B/E) should move to a daily cadence to match the daily reset — if reset runs daily but reassignment doesn't, there will be gaps where subtasks are reset with nothing downstream to reassign them until the next scheduled reassignment run
- [x] ~~Flow D originally reset one hardcoded ticket at a time — needed a "reset all" mode~~ — done: `reset_all()` handles every in-progress parent, wired into `main.py` as `--reset`

---

## Key roles

- **Automation owner** — builds/maintains this automation system
- **Infra/DevOps contact** — owns CI/CD pipeline config, scheduler config, the protected `main` branch, and server-side cronjob setup
- **Team leads** — maintain assignee pools directly on the Confluence page (no code changes needed)
