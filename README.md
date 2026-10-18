# Operations Under Pressure — internal work-item tracker

A small web app where teams create, claim, move through an approval workflow, and resolve operational
work items (incidents, customer issues, payments, compliance, tasks). The focus is correct behaviour
under concurrency, server-side authorization and an audit trail, not feature count.

- **Stack:** Python 3.12, Flask, SQLite (WAL), vanilla-JS single page UI, `unittest`.
- **Critical behaviours implemented (all tested):** concurrent claim, stale-update handling (optimistic
  locking), idempotency keys, server-enforced workflow + lead-only approval, resource-level authorization,
  transactional outbox + retrying worker, optimistic UI with rollback.

## Run it

### Option A — Docker (one command)
```bash
docker compose up --build
```
Open http://localhost:8000. The API container seeds the database on first start (11 users, 4 teams,
5,000 items); a second container runs the outbox worker.

### Option B — plain Python (3.10+)
```bash
python -m venv .venv && source .venv/bin/activate     # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python -m app.seed                                     # creates ops.db with demo data
python -m flask --app "app:create_app" run --port 8000 # API + UI at http://localhost:8000
python -m app.worker --loop                            # in a second terminal: outbox worker
```
Copy `.env.example` to `.env` only if you want a different DB path (`OPS_DB`).

## Demo logins (password for all: `password123`)

| username | teams → role |
|---|---|
| `admin` | all four teams → admin |
| `lead_pay` / `lead_comp` / `lead_sup` / `lead_plat` | Payments / Compliance / Support / Platform → lead |
| `member_pay1`, `member_pay2` | Payments → member |
| `member_comp` | Compliance → member |
| `member_sup` | Support + Platform → member |
| `viewer_pay` | Payments → viewer (read-only) |
| `multi` | Support → lead, Payments → viewer (different role per team) |

## Run the tests
```bash
python -m unittest -v          # no extra dependencies; `pytest` also works if installed
```

## API summary
`POST /api/login` · `GET /api/me` · `GET /api/dashboard` ·
`GET /api/items?status=&priority=&type=&assignee=me|<id>&q=&view=&team_id=&limit=&cursor=` ·
`POST /api/items` · `GET|PATCH /api/items/:id` · `POST /api/items/:id/claim|assign|transition|comments` ·
`GET /api/items/:id/activity` · `GET /api/notifications`.
`view` is one of `assigned_to_me`, `unassigned_urgent`, `awaiting_my_approval`. All writes accept an
optional `Idempotency-Key` header; edits/assign/transition require the item's current `version`.

Workflow: `open → in_progress → awaiting_approval → resolved → closed` (plus reject: awaiting_approval →
in_progress, reopen: resolved → in_progress). See `ENGINEERING_DECISIONS.md` for the role matrix and rules.

## Known limitations
- SQLite allows one writer at a time; it is a deliberate trade-off for one-command setup (see decisions).
  Moving to PostgreSQL is a contained change (SQL is standard apart from `BEGIN IMMEDIATE`, `INSERT OR IGNORE`).
- Search is `LIKE` on title/description (fine for thousands of rows, not for millions).
- Tokens are opaque and expire after 12 h; there is no logout-everywhere/revocation, rate limiting or lockout.
- Reading an item in a team you do not belong to returns 403 (which confirms the id exists).
- The UI is intentionally plain; it is only lightly tested by hand. The Docker files were written but the
  author's sandbox had no Docker, so verify `docker compose up` on your machine (Option B is the tested path).
- Failed requests are not cached under an idempotency key (only successes), so a retry re-runs the logic.
- No assignee picker in the UI (reassignment is API-only); no email/SSO/attachments/websockets by design.
