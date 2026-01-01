## Concrete story example for GitHub-like collaboration data modeling case study

**Scenario:** 
- Ali opens PR, 
- adds 3 commits, 
- Merve requests changes, 
- Ali commits the fix,
- Merve approves,
- code-owner team approval required,
- code-owner approves
- merge happens

---
* **Repo:** `OurRepo`
* **Base branch:** `main` is protected (requires reviews + passing checks).
* **Branch protection also requires code owner review** when owned files change.
* **CODEOWNERS exists on the base branch** (important: PRs use CODEOWNERS from the base branch to trigger requests).

**People**

* **Ali** opens PR
* **Merve** reviews (regular reviewer)
* **Backend team** is the code owner for `lib/search/**`

**Change**

* PR modifies `lib/search/query_parser.dart` (example file that changes)

**Key GitHub behaviors**

* A **draft PR cannot be merged**, and code owners are **not** automatically requested until it is “ready for review”
* Review states exist like **APPROVED / CHANGES_REQUESTED / COMMENTED / DISMISSED / PENDING**, and “CHANGES_REQUESTED” is considered *blocking*.
* A review can be **PENDING** before it is submitted (draft comments visible only to the reviewer).

---

## Step 0 — Competency questions for *this* story

(These are the “tests” for our model.)

1. What repo is PR #42 in, and who authored it?
2. Which commits are in PR #42?
3. Which files changed, and which CODEOWNERS rules matched them?
4. Who was requested for review (user vs code-owner team)?
5. What reviews were submitted, with what **state** and **timestamp**?
6. Why is the PR blocked *right now* (draft? changes requested? missing code owner approval? failing checks)?
7. When did it become mergeable, and when was it merged (and by whom)?

---

## Step 1–4 — Extract primitives + decide what is an event

Quickly applying the algorithm: nouns => entities, verbs => relationships/events, attributes => properties

### Entities (stable “things”)

`User`, `Team`, `Repo`, `PullRequest`, `Commit`, `FilePath`, `CodeOwnerRule`

### Verbs that must become **events**

Because they have their own time/state payload (“reify the verb”)

* “reviewed” => `ReviewSubmitted(state, at, body...)`
* “requested review” => `ReviewRequested(requestee, at, reason=CODEOWNERS|manual)`
* “merged” => `PRMerged(merged_at, merged_by, method...)`
* “checks finished” => `CheckRunCompleted(conclusion, at)`

---

# The core: **Event timeline** (facts we append)

Think “event sourcing”: store the events, derive the current state by replaying them.

### Event log (canonical truth)

| id | time (UTC) | actor | event_type | key payload(minimal) |
| --- | --- | --- | --- | --- |
| E1 | 2025-12-30 10:00 | Ali | PR_CREATED | repo=OurRepo, pr=42, base=main, head=feature/search-fix, **isDraft=true** |
| E2 | 2025-12-30 10:01 | Ali | COMMITS_ATTACHED | pr=42, commits=(c1,c2,c3) |
| E3 | 2025-12-30 10:02 | Ali | FILES_CHANGED_RECORDED | pr=42, files=(`lib/search/query_parser.dart`) |
| E4 | 2025-12-30 11:00 | Ali | PR_MARKED_READY_FOR_REVIEW | pr=42, isDraft=false |
| E5 | 2025-12-30 11:00 | system | REVIEW_REQUESTED | pr=42, requestee=BackendTeam, reason=CODEOWNERS |
| E6 | 2025-12-30 11:05 | Ali | REVIEW_REQUESTED | pr=42, requestee=Merve, reason=manual |
| E7 | 2025-12-30 12:00 | Merve | REVIEW_DRAFT_STARTED | pr=42, state=PENDING (draft) |
| E8 | 2025-12-30 12:10 | Merve | REVIEW_SUBMITTED | pr=42, state=CHANGES_REQUESTED (blocking) |
| E9 | 2025-12-30 13:00 | Ali  | COMMIT_PUSHED | commit=c4, branch=head |
| E10 | 2025-12-30 13:01 | system | PR_COMMIT_LINKED | pr=42, commit=c4 |
| E11 | 2025-12-30 14:00 | Merve | REVIEW_SUBMITTED | pr=42, state=APPROVED |
| E12 | 2025-12-30 15:00 | BackendTeamMember | REVIEW_SUBMITTED | pr=42, state=APPROVED |
| E13 | 2025-12-30 15:10 | CI | CHECKRUN_COMPLETED | pr=42, conclusion=success |
| E14 | 2025-12-30 15:15 | Ali | PR_MERGED | pr=42, merged_by=Ali, state=MERGED |

---

# Derived “current state” snapshots (replay the log)

This is the “state from events” idea: state is not stored as the *main truth*; it is computed from the event history.

Important derived fields:

* `PR.state` = {OPEN, CLOSED, MERGED}
* `PR.isDraft` (draft PRs cannot be merged)
* `PR.reviewDecision` = {REVIEW_REQUIRED, CHANGES_REQUESTED, APPROVED}
* `blockReasons[]` (our own derived explanation list)

### S0 — After E1–E3 (draft opened)

* state = OPEN
* isDraft = true
* reviewDecision = (does not matter yet; draft blocks merging anyway)
* blockReasons = [`DRAFT_PR`]

### S1 — After E4–E6 (ready + reviewers requested)

* state = OPEN
* isDraft = false
* reviewDecision = REVIEW_REQUIRED
* blockReasons = [`NEED_REVIEW`, `NEED_CODEOWNER_REVIEW`]

### S2 — After E8 (Merve requests changes)

* reviewDecision = CHANGES_REQUESTED
* blockReasons = [`CHANGES_REQUESTED`]

### S3 — After E9–E10 (Ali pushes fix commit)

* reviewDecision often remains CHANGES_REQUESTED until a new approval comes in 
  * model choice: we keep “blocking until cleared”
* blockReasons = [`CHANGES_REQUESTED`]

### S4 — After E11 (Merve approves)

* reviewDecision might still be REVIEW_REQUIRED because code-owner requirement is not satisfied yet (branch protection rule effect)
* blockReasons = [`NEED_CODEOWNER_REVIEW`]

### S5 — After E12–E13 (code owner approved + checks passed)

* reviewDecision = APPROVED
* blockReasons = [] (mergeable now)

### S6 — After E14 (merged)

* state = MERGED

---

## What this allows

* We can answer “**why blocked**?” by pointing to *specific events* (E8 = CHANGES_REQUESTED, E12 missing earlier, E13 checks missing earlier).
* We can audit: “who did what when” (event sourcing).
* We can flex between SQL/graph/RDF because the **meaning layer** is stable; only storage changes.

---

## How this maps to storage

### Relational (minimal)

* `PullRequest(pr_id=42, repo_id, is_draft, state, base_ref, head_ref, ...)`
* `Commit(sha)`
* `PullRequestCommit(pr_id, sha)` (E2/E10)
* `Review(review_id, pr_id, reviewer_id, state, submitted_at)` (E7/E8/E11/E12)
* `ReviewRequest(pr_id, requestee_type, requestee_id, reason, requested_at)` (E5/E6)
* `CheckRun(pr_id, name, conclusion, completed_at)` (E13)

### Property graph (minimal)

* (Ali)-[:AUTHORED]->(PR42)
* (PR42)-[:HAS_COMMIT]->(c1...c4)
* (Merve)-[:SUBMITTED_REVIEW {state, at}]->(PR42)
* (BackendTeamMember)-[:SUBMITTED_REVIEW {state, at}]->(PR42)
* (PR42)-[:CHANGED]->(File `lib/search/query_parser.dart`)
* (CodeOwnerRule)-[:MATCHES]->(File) and (CodeOwnerRule)-[:OWNED_BY]->(BackendTeam)

---