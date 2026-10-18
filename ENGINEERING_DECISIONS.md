# Engineering decisions

## Assumptions (what I would confirm with a product manager first)
1. A work item belongs to exactly one team; roles are per team (`viewer < member < lead < admin`).
2. A *member* may create, edit, comment, claim unassigned open items, and move **only items assigned to them**.
   A *lead/admin* may reassign and move any item in their team. *Viewers* are read-only.
3. Approval (`awaiting_approval → resolved`) needs a lead/admin **who is not the assignee** (separation of duties).
4. Claiming is "open and unassigned → assigned to me and in progress" in one atomic step.
5. Role x action matrix is in `app/perms.py` (actions) and `app/items.py::TRANSITIONS` (per-transition role).

## 1. Concurrency: atomic claim + optimistic locking
**Context:** two people can click "Claim" at the same time; two editors can overwrite each other.
**Options:** (a) read, check in app code, write; (b) pessimistic row locks; (c) conditional UPDATE / version column.
**Choice:** claim is `UPDATE ... WHERE id=? AND assignee_id IS NULL AND status='open'`; 0 rows changed means 409.
Edits/transitions/assignments send the `version` they read and update `WHERE version=?`, then increment it.
A mismatch returns 409 with the latest item so the UI can say "this item changed, reload".
**Trade-off:** clients must carry `version`, and a conflict costs the user a reload. In exchange there are no
held locks, no lost updates, and the logic is one statement that is easy to test (parallel-claim test).

## 2. History is written in the same transaction as the change
**Context:** history that can disagree with the item (change saved, event lost) destroys trust.
**Choice:** every command runs inside one `BEGIN IMMEDIATE` transaction opened in `api.run_write`; the item
update, `activity_events` rows, the idempotency record and any outbox job all commit or roll back together
(`AtomicHistory` test forces a failure after the update and asserts nothing persisted).
**Trade-off:** the activity table grows with every change and is append-only; at scale it would be partitioned
by time. Comments are stored as `comment` events rather than a separate table to keep one timeline.

## 3. Idempotency keys stored with the change
**Context:** double clicks and client retries must not create duplicate items or double-apply actions.
**Choice:** optional `Idempotency-Key` header. The key, a fingerprint of method+path+body, and the response are
stored in `idempotency_keys` in the same transaction as the change. A repeat returns the stored response
(`Idempotent-Replay: true`); the same key with a different body returns 422.
**Trade-off:** only successful responses are stored (errors are retried for real); keys are never purged yet
(a TTL job would handle that). Because writers are serialized, even *parallel* retries create one item (tested).

## 4. Authorization lives in one server-side function
**Choice:** `perms.authorize(user, team_id, action)` is called by the service layer for every read and write,
and list queries are scoped with `team_id IN (user's teams)`. The UI only hides buttons as a convenience.
Transition rules (lead-only approval, members only move their own items, no self-approval) are in `items.transition`.
**Trade-off:** 403 (not 404) for items in other teams is simpler to reason about and debug, but reveals that an
id exists. For a stricter system I would return 404 for non-members.

## 5. Transactional outbox for notifications
**Context:** "notify the new assignee / approvers" must not be lost if the process dies, and must not double-send.
**Choice:** the job row is inserted in the same transaction as the change. `app/worker.py` polls due jobs, runs the
handler inside a savepoint, retries failures with exponential backoff and stops after `max_attempts` (status
`failed`, error kept). Handlers are idempotent via `UNIQUE(outbox_id, user_id)` + `INSERT OR IGNORE`, so a job
delivered twice still yields one notification (tested, including rollback of a failed attempt's side effects).
**Trade-off:** polling adds up to a second of latency, and the demo worker processes one batch under a single write
lock. With PostgreSQL I would use `SELECT ... FOR UPDATE SKIP LOCKED` to run many workers in parallel.

## 6. SQLite instead of PostgreSQL (and keyset pagination)
**Context:** the brief suggests PostgreSQL but allows any stack. I wanted a one-command, dependency-free test run.
**Choice:** SQLite in WAL mode with `BEGIN IMMEDIATE` write transactions. Lists use keyset (cursor) pagination on
`id DESC` with composite indexes `(team_id, status, id)`, `(assignee_id, status, id)` and a partial index for
"unassigned urgent"; the query plan was checked to use them with 5,000 seeded items.
**Trade-off:** one writer at a time and weaker search. At 10x scale: PostgreSQL (row locks, `SKIP LOCKED`, partial
indexes, `tsvector` search), read replicas for dashboards, partition `activity_events`, a real queue for jobs.

## What I chose not to build
Sign-up/SSO, email delivery, attachments, websockets (the UI refetches after each action), an assignee picker UI,
rate limiting, token revocation.
