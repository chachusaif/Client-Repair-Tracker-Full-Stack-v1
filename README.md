# Repair Tracker

A job tracking system for an independent phone and computer repair shop.
Staff book in repairs, move them along a fixed pipeline and record notes;
customers check the status of their own repair with a ticket reference, without
logging in or phoning the shop.

Built with Node.js, Express and PostgreSQL. No front-end framework.

---

## What it does

**Staff (login required)**
- Book in a repair: customer, contact number, device, fault, quoted price
- Every job gets a customer-facing reference (`TSH-0001`)
- Move a job through the pipeline: booked in → diagnosed → in repair → ready → collected
- Record technician notes
- Filter jobs by status
- See the full history of every status change, with who made it and when

**Customers (no login)**
- Enter a ticket reference and see the current stage of their repair

---

## Design decisions worth knowing

**Status changes are a state machine, enforced on the server.**
A job cannot jump from *booked in* to *collected*. The allowed moves live in one
object in `routes/jobs.js`, and the API rejects anything else with a 409. The
browser only ever shows the buttons the server says are legal.

**Every status change is appended to `status_history`, never overwritten.**
The `jobs` table holds current state; `status_history` holds the audit trail.
"Who marked this ready, and when?" is answerable months later.

**Status changes run inside a transaction with `SELECT ... FOR UPDATE`.**
Two staff clicking at the same moment would otherwise both read the old status
and both write a change. The row lock makes the read-check-write sequence atomic.

**Prices are integer pence.**
`4999`, not `49.99`. Binary floating point cannot represent decimal fractions
exactly, and the errors accumulate.

**The public endpoint returns almost nothing.**
`GET /api/track/:ref` returns the reference, device and status. No name, no phone
number, no price. References are sequential and therefore guessable, so the
endpoint is also rate limited to 20 lookups per IP per 10 minutes.

**There is no field for a device passcode.**
Paper tickets in repair shops routinely record them. Storing unlock credentials
for other people's devices is a liability with no upside, so the schema has
nowhere to put one. The intake form says so explicitly.

**Passwords are bcrypt hashes at 12 rounds.**
The plaintext is never stored or logged. Login returns the same error message
whether the username or the password was wrong, so accounts cannot be enumerated.

**Sessions live in Postgres, not in memory.**
The default in-memory session store loses everything on restart and leaks memory.
The cookie is `httpOnly`, `sameSite=lax`, and `secure` in production, and holds
only a session id — no user data.

---

## Running it locally

Requires Node.js 18+ and PostgreSQL 14+.

```bash
npm install
createdb repair_tracker
cp .env.example .env          # then edit it
psql -d repair_tracker -f schema.sql
npm run hash                  # creates your first staff account
npm start
```

Open <http://localhost:3000>. Staff login is at `/login`.

Optional sample jobs:

```bash
psql -d repair_tracker -f seed.sql
```

Generate a session secret with:

```bash
node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"
```

---

## API

| Method | Route                      | Auth   | Purpose                          |
|--------|----------------------------|--------|----------------------------------|
| GET    | `/api/track/:ref`          | public | Status of one ticket             |
| POST   | `/api/login`               | public | Start a session                  |
| POST   | `/api/logout`              | staff  | End a session                    |
| GET    | `/api/me`                  | staff  | Is this session still valid?     |
| GET    | `/api/jobs`                | staff  | List jobs (`?status=` optional)  |
| GET    | `/api/jobs/:id`            | staff  | One job, its history, next moves |
| POST   | `/api/jobs`                | staff  | Book in a repair                 |
| PATCH  | `/api/jobs/:id/status`     | staff  | Move along the pipeline          |
| PATCH  | `/api/jobs/:id/notes`      | staff  | Save technician notes            |

---

## Data protection

The system holds customer names, phone numbers and device details, which are
personal data under UK GDPR.

- Collect nothing that is not needed to do the repair
- Never record device passcodes, PINs or patterns
- Agree a retention period with the shop and delete collected jobs after it
- Take database backups, and keep them somewhere access-controlled

---

## Roadmap

1. Per-user roles: owner sees revenue figures, technicians do not
2. SMS notification when a job reaches *ready* (Twilio)
3. Parts and IMEI tracking for second-hand stock
4. Dashboard: average turnaround, most common faults, busiest days
5. Automated tests, then CI on GitHub Actions
