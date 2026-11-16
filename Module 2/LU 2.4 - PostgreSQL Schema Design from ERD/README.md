# Helpdesk Schema Repair — LU 2.4 Assignment

## What This Is

You are a backend engineer on the Helpdesk team. A colleague wrote the first draft of the PostgreSQL schema in `migrations/001_initial_schema.sql`. It has **intentional errors** — wrong column types, missing constraints, wrong ON DELETE behaviour, and a rejected table shape that should have never been there.

Your job: **fix the schema so it correctly implements the ER diagram** from the assignment question. The ER diagram is the source of truth. The schema must match it exactly.

When you're done, all 36 pgTAP tests must pass.

---

## Prerequisites

Make sure you have the following installed before starting:

| Tool | Why | Check |
|------|-----|-------|
| PostgreSQL (v13+) | Runs the test database | `psql --version` |
| pgTAP | Test framework for schema validation | `pg_prove --version` |
| pg_prove | pgTAP test runner | comes with pgTAP |
| Node.js + npm | Runs the helper scripts | `node --version` |

### Installing pgTAP

**macOS (Homebrew):**
```bash
brew install pgtap
```

**Ubuntu / Debian:**
```bash
sudo apt-get install postgresql-15-pgtap
# replace 15 with your postgres version
```

**Windows (via pgxn):**
```bash
pgxn install pgtap
```

After installing, enable it in your database:
```sql
CREATE EXTENSION IF NOT EXISTS pgtap;
```

---

## Setup

### Step 1: Clone or fork the repo

```bash
git clone <repo-url>
cd helpdesk-schema-repair
```

### Step 2: Make sure PostgreSQL is running

```bash
# macOS (Homebrew)
brew services start postgresql

# Ubuntu
sudo service postgresql start

# Docker (if you prefer)
docker run --name helpdesk-pg -e POSTGRES_PASSWORD=postgres -p 5432:5432 -d postgres:15
```

### Step 3: Create the test database and apply the schema

```bash
npm run db:reset
```

This command does three things in sequence:
1. Drops the `helpdesk_test` database if it exists
2. Creates a fresh `helpdesk_test` database
3. Applies `migrations/001_initial_schema.sql` to it

> ⚠️ If you see a "role does not exist" error, your PostgreSQL user might need to be set. Run `psql -U postgres` first and make sure you can connect. Then update the `db:reset` script in `package.json` to add `-U postgres` to each psql command.

### Step 4: Run the tests

```bash
npm test
```

The first time you run this, **most tests will fail**. That's expected — the schema is broken. Your job is to fix it until all 36 pass.

---

## What the Tests Check

The 36 pgTAP tests verify:

| Category | Tests | What they check |
|----------|-------|-----------------|
| Table existence | 6 | All 6 tables exist (including the 2 you need to add) |
| Column types | 16 | UUIDs are UUID not INTEGER, timestamps are TIMESTAMPTZ not TIMESTAMP, booleans are BOOLEAN not TEXT |
| NOT NULL | 11 | Required fields are actually marked NOT NULL |
| Nullable | 3 | Optional fields are correctly nullable |
| Column absence | 1 | The rejected `tags TEXT[]` column is gone |
| UNIQUE | 1 | `agents.email` is UNIQUE |

---

## The File to Fix

**Only edit this file:**

```
migrations/001_initial_schema.sql
```

Every broken line has a `-- wrong:` comment explaining what the problem is. Read each comment carefully.

---

## Hints

Work table by table. Don't try to fix everything at once.

**Hint 1 — Types:**
The ERD uses conceptual types. Translate them:
- `uuid` → `UUID`
- `string` → `TEXT` (not VARCHAR)
- `text` → `TEXT`
- `boolean` → `BOOLEAN` (not TEXT)
- `timestamp` → `TIMESTAMPTZ` (not TIMESTAMP — timezone matters in production)

**Hint 2 — Primary Keys:**
Every `SERIAL PRIMARY KEY` should be `UUID PRIMARY KEY DEFAULT gen_random_uuid()`. Check if you need to enable the `pgcrypto` extension first.

**Hint 3 — NOT NULL:**
If the ERD shows a field without `"nullable"` in quotes, it is required. Add `NOT NULL`. Go through every field in every entity in the ERD.

**Hint 4 — CHECK constraints:**
If the ERD attribute note shows pipe-separated values like `"starter|pro|enterprise"`, that means a CHECK constraint with exactly those allowed values. Do not use an ENUM type — use `TEXT NOT NULL CHECK (column IN (...))`.

**Hint 5 — Foreign Keys:**
All FK columns in the broken schema are `INTEGER`. They should all be `UUID`. Also: every FK needs an explicit `ON DELETE` clause. Look at the assignment question for the correct ON DELETE behaviour per FK (the table is provided there).

**Hint 6 — The Rejected Shape:**
There's an `ALTER TABLE` at the bottom of the broken schema adding `tags TEXT[]` to tickets. This is the wrong design. Remove that line entirely. Then add two new tables: `tags` and `ticket_tags`. Their columns are in the ER diagram.

**Hint 7 — Timestamps:**
All `created_at` columns should have `NOT NULL DEFAULT NOW()`. All nullable timestamp columns (`resolved_at`, `deactivated_at`) should have no DEFAULT and no NOT NULL.

---

## Workflow

The recommended loop while fixing:

```bash
# 1. Edit the schema
nano migrations/001_initial_schema.sql   # or use your editor

# 2. Reset the database (re-applies the schema from scratch)
npm run db:reset

# 3. Run the tests and see how many pass
npm test

# 4. Repeat until all 36 pass
```

Each time you run `db:reset`, the database is wiped and recreated from your current schema file. This means each run is a clean slate — no leftover state.

---

## Common Errors

**`ERROR: type "uuid" does not exist`**
→ Add `CREATE EXTENSION IF NOT EXISTS "pgcrypto";` at the top of your SQL file.

**`ERROR: function gen_random_uuid() does not exist`**
→ Same fix — pgcrypto extension is not enabled.

**`ERROR: column "tags" of relation "tickets" does not exist` (when running tests)**
→ You still have the `ALTER TABLE tickets ADD COLUMN tags TEXT[]` line. Remove it.

**`ERROR: there is no unique constraint matching given keys for referenced table`**
→ A FK references a column that is not a primary key or unique. Check your FK definitions.

**`pg_prove: command not found`**
→ pgTAP is not installed or not on your PATH. See prerequisites above.

**`psql: error: connection to server failed`**
→ PostgreSQL is not running. Start it with `brew services start postgresql` (macOS) or `sudo service postgresql start` (Linux).

---

## Submission

Once all 36 tests pass:

1. Commit your changes to a branch called `schema-repair`:
```bash
git checkout -b schema-repair
git add migrations/001_initial_schema.sql README.md
git commit -m "fix: repair helpdesk schema to match ERD"
```

2. Update `README.md` (this file) with exactly **3 ERD-to-schema decisions** and **1 rejected table shape** using this format:

```markdown
## ERD-to-Schema Decisions

### Decision 1: [Column type decision]
ERD said: ...
Schema decided: ...
Reason: ...

### Decision 2: [ON DELETE decision]
ERD said: ...
Schema decided: ...
Reason: ...

### Decision 3: [Default value decision]
ERD said: ...
Schema decided: ...
Reason: ...

## Rejected Table Shape

### Shape: [name what you removed or rejected]
What it was: ...
Why rejected: ...
What would break: ...
```

3. Push and open a PR:
```bash
git push origin schema-repair
```

4. In the PR description, paste your `npm test` output showing all 36 tests passing.

---

## What You May Not Do

- Change the ER diagram (it is the source of truth)
- Add columns that don't exist in the ER diagram
- Skip `ON DELETE` on any FK — every FK must have an explicit `ON DELETE` clause
- Use `SERIAL` or `INTEGER` for any PK or FK column
- Leave the `tags TEXT[]` array column on the tickets table
- Use ENUM type instead of TEXT + CHECK

---

## Project Structure

```
helpdesk-schema-repair/
├── migrations/
│   └── 001_initial_schema.sql   ← the broken schema — only file you edit
├── tests/
│   └── schema.test.sql          ← 36 pgTAP tests — do not edit
├── README.md                    ← this file — update with your decisions
└── package.json                 ← npm scripts — do not edit
```


---

## ERD-to-Schema Decisions

### Decision 1: [Column type decision]
ERD said: Entities in the ER diagram specified conceptual types: `uuid id PK` across all tables, `string plan "starter|pro|enterprise"`, `string role "agent|supervisor|admin"`, `string priority "low|medium|high|urgent"`, `string status "open|pending|resolved|closed"`, `boolean internal`, and `timestamp` fields.
Schema decided: Used PostgreSQL `UUID` with `gen_random_uuid()` for all primary keys and foreign keys, `TEXT NOT NULL CHECK (...)` for constrained string columns, `BOOLEAN NOT NULL DEFAULT FALSE` for `internal`, and `TIMESTAMPTZ` (timestamp with time zone) for all temporal fields (`created_at`, `deactivated_at`, `resolved_at`, and `added_at`).
Reason: `TIMESTAMPTZ` stores exact instants in UTC, avoiding timezone skew and ambiguity for multi-region support systems where SLA and resolution timestamps are audited. Using `TEXT` with `CHECK` constraints enforces domain validity without the migration friction and table-locking hazards associated with native PostgreSQL `ENUM` types. `UUID` prevents predictable integer enumeration attacks and facilitates distributed ID generation.

### Decision 2: [ON DELETE decision]
ERD said: Specified relationships between entities (`Organization ||--o{ Agent`, `Organization ||--o{ Ticket`, `Organization ||--o{ Tag`, `Agent ||--o{ Ticket : "created_by"`, `Agent o|--o{ Ticket : "assigned to"`, `Agent ||--o{ Comment : "authored"`, `Ticket ||--o{ Comment`, `Ticket ||--o{ TicketTag`, `Tag ||--o{ TicketTag`, `Agent ||--o{ TicketTag : "added_by"`), but conceptual ER diagrams do not specify physical deletion semantics.
Schema decided: Configured explicit `ON DELETE` rules across all 10 foreign keys:
- `RESTRICT` on `agents.org_id`, `tickets.org_id`, `tickets.created_by`, `comments.author_id`, and `ticket_tags.added_by`.
- `SET NULL` on `tickets.assignee_id`.
- `CASCADE` on `comments.ticket_id`, `tags.org_id`, `ticket_tags.ticket_id`, and `ticket_tags.tag_id`.
Reason: In customer service, compliance and auditing require an immutable record of who filed a ticket, wrote a response, or applied a tag (`RESTRICT`). If an assigned staff member is removed, the ticket should remain intact and merely become unassigned (`SET NULL`) so work is not lost. Conversely, dependent entities that cannot exist independently of their parent—such as comments on a ticket or join entries linking tickets and tags—must be automatically cleaned up (`CASCADE`) to avoid orphaned records.

### Decision 3: [Default value decision]
ERD said: Conceptual attributes such as `id PK`, `created_at`, `status "open|pending|resolved|closed"`, `internal "visible to agents only"`, and `added_at` did not define SQL defaults.
Schema decided: Implemented database-level `DEFAULT` values: `id DEFAULT gen_random_uuid()`, `created_at DEFAULT NOW()`, `added_at DEFAULT NOW()`, `status DEFAULT 'open'`, and `internal DEFAULT FALSE`.
Reason: Pushing default generation into PostgreSQL guarantees data consistency regardless of which client application or microservice inserts records. Automatic timestamp assignment (`NOW()`) and UUID generation (`gen_random_uuid()`) prevent clock-skew errors or omitted IDs from application code. Defaulting status to `'open'` and comment visibility to `FALSE` (customer-facing) ensures new tickets and comments enter the system in a known, safe state without requiring explicit boilerplate on every insert.

## Rejected Table Shape

### Shape: Tags stored as array on ticket (`ALTER TABLE tickets ADD COLUMN tags TEXT[];`)
What it was: A denormalized array column (`tags TEXT[]`) on the `tickets` table storing tag names as plain text strings.
Why rejected: It violates First Normal Form (1NF) and completely breaks the ER diagram's relational model. In the ERD, `Tag` is an organization-scoped entity (`Organization ||--o{ Tag : "defines"`) with its own properties (`name`, `color_hex`). A ticket's relationship to tags is Many-to-Many ($M:N$), which requires an associative join entity (`TicketTag`) containing audit metadata (`added_by`, `added_at`).
What would break:
1. Multi-tenancy and tag management: Organizations could not define, restrict, or enforce a uniform set of tags; any arbitrary string could be added.
2. Metadata loss: Visual UI attributes like `color_hex` and audit trails (`added_by`, `added_at`) cannot be tracked on individual array elements.
3. Query performance & integrity: Searching or aggregating tickets by tag would require expensive unnesting or GIN index scans, and renaming or deleting an organization tag would necessitate updating millions of array elements across the `tickets` table instead of a simple, cascade-backed relational operation.
