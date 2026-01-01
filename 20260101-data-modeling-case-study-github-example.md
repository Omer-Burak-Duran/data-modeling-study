# GitHub-like collaboration

**Scenario:** Repos, issues, PRs, commits, reviewers, approvals, code ownership.

## Step 0 — Competency questions (what must the model answer?)

1. For a given PR, **who authored it**, **which repo**, **which branch** (base/head)?
2. **Which commits** are included in the PR? In what order?
3. Who **reviewed** the PR, with what **state** (approved / changes requested / commented / pending / dismissed), and **when**?
4. What is the PR’s current **review decision** and **mergeability** (blocked by changes requested? blocked by missing code-owner review?).
5. Which files changed, and **which CODEOWNERS rules** apply (which owners should be requested)?
6. For a user/team: “What did they review last week, and what got merged?”
7. “Which repos does Team X effectively own (via CODEOWNERS patterns), and what PRs are waiting on them?”
8. Auditing: “Show me the full timeline that explains *why* PR #123 is blocked.”

These set of questions imply we will need **history** + **relationship attributes** + **deep traversals** => event reification + graph-friendly modeling. 

---

## Step 1 — Extract nouns / verbs / adjectives

### Nouns (candidate entities)

* User, Team, Repo
* PullRequest (PR), Issue
* Commit, Branch/Ref
* Review, ReviewComment
* FilePath, CodeOwnerRule
* CheckRun / CI status

### Verbs (candidate relationships or events)

* author, push, open PR, add commit to PR, review, approve, request changes, comment, dismiss review, merge
* “own code”
* “request review”

### Adjectives/adverbs (attributes)

* timestamps (created_at, submitted_at, merged_at)
* states (PR state open/closed/merged; review state)
* role-like attributes (reviewer role, team vs user, permission level)
* “base branch”, “head branch”

---

## Step 2 — Relationship vs Event (reify the verbs)

**If a “relationship” needs attributes (time/state/text), reify into an event/entity**. 

### Clear “events”

* **Review** is not just `(User)-[:REVIEWED]->(PR)` because it has:
  * state (approved/changes requested/...), submitted_at, body text, maybe “dismissed by”, and so on.
* **PR <=> Commit membership** often needs “added_at” (or at least a stable join) => model as join/edge with optional metadata.
* **Merge** is an event (merged_at, merger, merge_commit_sha, method).

### “Relationships” that can stay simple

* PR => Repo (belongs-to)
* Commit => Repo (exists-in)
* User => Team membership (but membership changes over time => might become event-sourced if we care historically)

---

## Step 3 — Identity rules (what makes “the same thing” the same?)

This step prevents “duplicate truth” bugs.

* **Commit** identity: `commit_sha` (content-addressed hash in Git’s object model).
* **Pull Request** identity: `(repo_id, pr_number)` is natural; we may still use a surrogate `pr_id`.
* **Review** identity: `review_id` (surrogate). A reviewer can submit multiple reviews on the same PR over time.
* **Repo** identity: `repo_id` (or `(owner, name)` as natural key).
* **User** identity: `user_id` (not username; usernames can change).
* **CODEOWNERS rule** identity: `(repo_id, branch, pattern, owners_set, order_index)` because:
  * CODEOWNERS applies per branch and must be on the **base branch** to trigger requests.

---

## Step 4 — Cardinalities + constraints (our model’s “physics”)

These matter because they dictate table shape *or* edge multiplicity.

* Repo **1 => N** PRs
* PR **N <=> M** Commits (a commit can appear in multiple PR contexts; PR has multiple commits)
* PR **1 => N** Reviews; Review **N => 1** PR
* User **1 => N** Authored PRs
* User **1 => N** Reviews; each Review has exactly **1** author
* PR “review decision” is **derived state**, not a primitive fact. GitHub exposes review states as an enum with values like APPROVED / CHANGES_REQUESTED / COMMENTED / PENDING / DISMISSED.

If we enforce constraints:

* “Every Review must reference exactly one PR and one reviewer.”
* “Every PR belongs to exactly one Repo.”
* “Commit SHA must be unique.”

---

## Step 5 — Normalize (avoid redundancy of truth)

Relationally, we normalize so we do not have duplicates:

* user display names/email across every event row
* repo metadata across PRs/commits
* code-owner “truth” across PRs

Graph-wise, same principle: do not copy-paste truth into many node properties; prefer shared nodes/edges.

---

## Step 6 — Time, state, and “why is it blocked?”

This domain needs explicit time.

**Key modeling idea:** treat the collaboration system as **eventful**, then derive current states:

* PR state = open/closed/merged (plus timestamps)
* Review decision = function of latest relevant review events (e.g., latest review per reviewer, dismissals, etc.)
* Mergeability = depends on branch protection rules, required code-owner reviews, CI checks, etc.

GitHub’s APIs indicate that review actions/states are first-class (approve / request changes / comment, and pending until submitted).

CODEOWNERS adds another “derived requirement”: code owners get review requests when CODEOWNERS is present on the base branch and patterns match changed files.

---

# Knowledge-level model (meaning-first)

Think in concepts + rules, no storage yet:

### Entities (stable “things”)

* User, Team, Repo
* PR, Commit
* CodeOwnerRule

### Events (changing “happenings”)

* PullRequestOpened
* CommitAddedToPR
* ReviewSubmitted(state, at, body)
* ReviewDismissed(at, by_whom, reason)
* PRMerged(merged_at, merged_by, merge_commit)

### Derived state (computed “current truth”)

* PR.reviewDecision
* PR.requiredApproversSatisfied?
* PR.blockReasons[] (changes requested, missing code-owner review, failing checks, etc.)

---

# Operational level A — Relational schema sketch (SQL style)

Clean, normalized core:

* `User(user_id PK, login, ...)`

* `Team(team_id PK, org_id, name, ...)`

* `TeamMember(team_id FK, user_id FK, from_ts, to_ts)`

* `Repo(repo_id PK, owner, name, default_branch, ...)`

* `PullRequest(pr_id PK, repo_id FK, pr_number, author_id FK, base_ref, head_ref, created_at, closed_at, merged_at, state)`

  * Unique constraint: `(repo_id, pr_number)`

* `Commit(commit_sha PK, repo_id FK, author_id FK, authored_at, message, tree_sha ...)`

* `PullRequestCommit(pr_id FK, commit_sha FK, added_at, PRIMARY KEY(pr_id, commit_sha))`

* `Review(review_id PK, pr_id FK, reviewer_id FK, state, submitted_at, body)`

  * `state` values align with GitHub’s model (APPROVED / CHANGES_REQUESTED / ...).

* `FileChange(pr_id FK, path, status, additions, deletions, PRIMARY KEY(pr_id, path))`

* `CodeOwnerRule(rule_id PK, repo_id FK, branch, pattern, order_index)`

* `CodeOwnerRuleOwner(rule_id FK, owner_type, owner_id, PRIMARY KEY(rule_id, owner_type, owner_id))`

**Why this works:**

* M:N (for example, PR <=> Commit) uses a bridge table (our “micro-pattern”). 
* Review, for example, is reified because it has its own attributes.

---

# Operational level B — Property graph sketch (Neo4j style)

### Nodes

`(:User {id})`, `(:Team {id})`, `(:Repo {id})`, `(:PR {id, number})`, `(:Commit {sha})`, `(:Review {id})`, `(:CodeOwnerRule {id})`, `(:File {path})`

### Edges (relationships)

* `(u)-[:AUTHORED]->(pr)`
* `(pr)-[:IN_REPO]->(repo)`
* `(pr)-[:HAS_COMMIT]->(c)`
* `(r:Review)-[:ABOUT]->(pr)`
* `(u)-[:WROTE]->(r)`
* `(r)-[:STATE {value, at}]->(:ReviewState)`
* `(pr)-[:CHANGED]->(f:File)`
* `(rule)-[:OWNS]->(f)` and `(rule)-[:OWNER]->(u/team)`

Graph is useful when we ask things like: “show me the neighborhood of this PR”: commits => authors => reviewers => teams => ownership rules.

---

# Operational level C — RDF / Knowledge graph sketch (RDF style)

Triples are binary, so **Review** becomes an n-ary node (same reify rule). 

Example:

* `(pr42, inRepo, repo7)`
* `(pr42, authoredBy, user7)`
* `(pr42, hasCommit, commitABC)` 
* `(review9, type, PullRequestReview)`
* `(review9, aboutPR, pr42)`
* `(review9, reviewer, user12)`
* `(review9, reviewState, "CHANGES_REQUESTED")`
* `(review9, submittedAt, "2026-01-01T10:00:00Z")`

CODEOWNERS:

* `(rule3, pattern, "/apps/backend/**")`
* `(rule3, owner, teamX)`
* `(rule3, appliesOnBranch, "main")`

If we later want validation, we would express “every Review must have exactly 1 reviewer and 1 PR” as SHACL shapes.

---

## Answering competency questions

Pick one hard question and see if the structure answers it:

**Q:** “Why is PR #123 blocked?”

We traverse:

PR => reviews (latest states) + required owners (CODEOWNERS rules matched by changed files) + missing approvals.

* In SQL: join PR => FileChange => CodeOwnerRule => RuleOwner + join PR => Review (aggregate latest).
* In graph: PR neighborhood traversal + filtering by timestamps.
* In RDF: path queries + SHACL validation for “missing required approval”.

If answering this feels difficult, then our model or our primitives are bad. Maybe we failed to reify a verb, or we ignored time etc. 

