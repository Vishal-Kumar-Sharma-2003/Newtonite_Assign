# Newtonite SWE Challenge: discussion prep + submission checklist
(Deadline: Mon 19 Oct 2026, 10:00 AM. Aim to submit by 9:30 AM.)

## Submit steps (do in this order)
1. Unzip `newtonite-ops-23131012452.zip`, `cd` in, run the app yourself (README Option B) and `python -m unittest -v`.
2. `git init`, then commit in several steps (the brief wants progress, not one giant commit), e.g.:
   schema+db -> auth+perms -> items service -> api -> worker+outbox -> tests -> UI -> docs/docker.
3. Create PUBLIC GitHub repo named `newtonite-ops-23131012452`, push.
4. Clone fresh into a new folder and follow your own README once.
5. Optional: 3-5 min demo video (login as lead_pay -> create -> claim as member_pay1 -> approve -> activity log).
6. Fill the Google Form (100-word "most important decision": use decision 1 or 2 from ENGINEERING_DECISIONS.md).

## 60-second architecture
Browser (static JS) -> Flask API -> SQLite (WAL) <- outbox worker (separate process).
Every write = one transaction: item change + activity event + idempotency record + outbox job.

## Be ready to explain / modify live
- Where is authorization? `perms.authorize` + `items.transition` rules. Add a new status: edit `STATUSES`, `TRANSITIONS`,
  and the CHECK constraint in `db.py`. "Let Leads reassign": already `assign` (lead+); change `ACTION_MIN_ROLE`.
- Claim race: one conditional UPDATE; only one row can match `assignee_id IS NULL`.
- Stale edit: `UPDATE ... WHERE version=?` -> 0 rows -> 409 with latest item.
- Retry/double-click: Idempotency-Key row stored in the same transaction; replay returns stored response.
- Failures: DB down -> request 500, nothing partial (transaction). Worker crash mid-job -> savepoint/txn rolls back,
  job stays pending, retried; handler idempotent via UNIQUE(outbox_id,user_id). Two users colliding -> 409.
- Why SQLite? One-command setup/testability; honest trade-off = one writer. 10x scale: PostgreSQL, SKIP LOCKED workers,
  read replicas, partition activity table, search engine/tsvector, real queue, websockets.
- Assumptions to confirm with a PM: who can approve, self-approval, can members reassign, SLA/due-date rules.
- Most important part: the transaction boundary in `api.run_write` (history can never disagree with the item).
