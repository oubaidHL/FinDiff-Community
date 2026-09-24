# FinDiff

**Self-hosted, read-only observability for the database tables that quietly run your financial products.**

FinDiff connects to your databases in **read-only mode**, maps raw schema
columns to language a compliance officer or product manager can actually
read, automatically detects and timestamps every change to configuration
data, and lets you (or an AI Copilot) ask "what changed, and when" without
ever touching production with a write.

This repository is the **public, community-facing home of FinDiff**: product
documentation, feature overview, and deployment instructions. It does not
contain FinDiff's source code — FinDiff ships as a single, official Docker
image ([`oubaidhl/findiff`](https://hub.docker.com/r/oubaidhl/findiff)) that
you run against your own infrastructure and your own data.

- Website: https://findiff.fr
- Docker Hub: https://hub.docker.com/r/oubaidhl/findiff
- Author: [Oubaid HL](https://github.com/oubaidHL) — [LinkedIn](https://www.linkedin.com/in/oubaidhlaimi)

---

## Table of contents

1. [About FinDiff](#about-findiff)
2. [The story behind it](#the-story-behind-it)
3. [Core guarantee: read-only, always](#core-guarantee-read-only-always)
4. [Features](#features)
5. [Supported database engines](#supported-database-engines)
6. [Quick start](#quick-start)
7. [Preparing a production deployment](#preparing-a-production-deployment)
8. [Trying it in a dev/throwaway environment](#trying-it-in-a-devthrowaway-environment)
9. [Configuration reference](#configuration-reference)
10. [Scaling to very large tables](#scaling-to-very-large-tables)
11. [Backups and data safety](#backups-and-data-safety)
12. [FAQ](#faq)
13. [Support](#support)

---

## About FinDiff

Most financial platforms have a handful of database tables that nobody fully
owns anymore — rate tables, risk tiers, fee schedules, eligibility rules —
that quietly drive pricing, risk, or compliance decisions. Nobody documents
them well, nobody notices when they change, and by the time someone asks
"who changed this, and when," the answer is usually a shrug.

FinDiff is a single tool built specifically for that gap. It sits **next
to** your production databases — never inside their write path — and gives
you:

- A living, human-readable map of what your raw schema actually means.
- A full, automatic audit trail of every change to that data, with old and
  new values, down to the row and column.
- A way to compare two environments (staging vs. production, region A vs.
  region B) and see exactly where their configuration has drifted apart.
- Alerts the moment something changes, and an AI assistant you can ask
  natural-language questions about what happened and why.

It's designed to be handed to a client, a compliance team, or your own
platform team as-is: **single Docker image, your own infrastructure, your
own data, your own choice of AI provider (cloud or fully local). No
dependency on the author, no data leaving your network unless you decide it
should.**

## The story behind it

FinDiff came out of years spent moving between IT consulting engagements,
cloud infrastructure work, and automation projects — the kind of role where
you get dropped into a different company's stack every few months. Across
almost every one of them, the same quiet, dangerous pattern kept showing up:
a handful of database tables nobody fully understood anymore, silently
driving pricing, risk, or eligibility decisions.

It's always some variation of the same story. A table called
`TBL_PARAM_RATES` that everyone is afraid to touch. A rate that got changed
on a Friday afternoon and nobody notices until a customer complains three
weeks later. A compliance team asking "who changed this, and when" and
getting a shrug. An engineer copy-pasting a spreadsheet of "what these
columns mean" that's been stale since the person who wrote it left the
company.

None of this happens because anyone is careless. It happens because there
was never a tool built specifically for this problem — one that sits
between a raw, opaque schema and the business people who actually need to
understand and govern it, without asking anyone to grant it write access to
production.

FinDiff is the answer to that gap: a self-hosted, read-only observability
layer for the configuration tables that quietly run financial products. It
maps raw columns to language a compliance officer or product manager can
read, detects and timestamps every change automatically, and lets an AI
reason over that history when something breaks and someone needs an answer
fast.

— Oubaid HL, Cloud Engineer, IT Consulting & Automation, creator of Termini

## Core guarantee: read-only, always

FinDiff is built so that it is **structurally incapable** of writing to a
database it monitors, not just configured that way by convention:

- Every query FinDiff issues against a monitored database passes through a
  statement guard that only allows `SELECT` statements — anything else
  (`INSERT`, `UPDATE`, `DELETE`, DDL, multi-statement tricks, comment-based
  bypass attempts) is rejected before it ever reaches the driver.
- Where the underlying driver supports it, FinDiff opens the connection in
  a read-only transaction mode as a second, independent layer of
  protection.
- FinDiff never issues schema changes (`ALTER TABLE`, `ALTER DATABASE`)
  against a monitored database. Advanced features that use engine-native
  change tracking (Oracle's `ORA_ROWSCN`, SQL Server's Change Tracking) only
  ever **read** metadata your DBA has already opted into — FinDiff never
  enables them itself.
- As defense-in-depth on top of all of the above, we strongly recommend
  connecting FinDiff with a dedicated, database-level **read-only role**
  (e.g. `GRANT SELECT ONLY` in Postgres/MySQL/MariaDB/MSSQL, a read-only
  profile in Oracle) — so that even a catastrophic bug in FinDiff itself
  would hit a database-level permission wall, not just an application-level
  one.

All of FinDiff's own state — mappings, snapshots, diff history, settings,
AI conversations — lives in its own local SQLite database, entirely
separate from anything it monitors.

## Features

**Schema mapping & documentation**
- In-app Schema Manager to map raw `schema.table.column` names to a
  business label, category, and free-text description.
- **Excel/CSV export → fill in → import round-trip**: export a template
  pre-filled with every live column FinDiff can see, hand it to a
  non-technical business owner to fill in offline, then re-import it — every
  row is validated against the live schema before anything is applied, with
  a clear per-row error report for anything unrecognized.
- **Table Groups**: cluster related tables into a named module (e.g. "Loan
  Origination"), with a one-click "Suggest Groups" feature that reads
  foreign keys and proposes groups automatically.

**Change detection**
- Row-level SHA-256 diffing between snapshots — every `INSERT`, `UPDATE`,
  and `DELETE` is detected and recorded with full before/after values.
- Configurable snapshot interval per database.
- **Live Diff Timeline** over WebSocket — new changes appear in the UI the
  moment they're detected, no page refresh needed.
- Filterable, paginated history: filter by change type, table name, date
  range (and by severity/module/database on the System Health page), with
  a 50/100/500/1000 page-size selector everywhere a list can grow long.

**Drift & integrity**
- **Drift View**: compare two environments (e.g. staging vs. production)
  side by side and see exactly which rows differ.
- **Data Integrity checks**: referential integrity (orphaned foreign keys)
  across all supported engines, plus Oracle-specific invalid PL/SQL object
  detection.

**Alerting & retention**
- Alert dispatch to Slack, Microsoft Teams, email (SMTP), or any generic
  HTTP webhook, fired automatically on every detected change.
- Configurable retention window with automatic export-to-archive before
  pruning old history, so nothing is silently lost.

**AI Copilot**
- Provider-agnostic: OpenAI, Anthropic Claude, Google Gemini, Ollama, LM
  Studio, or any other OpenAI-compatible endpoint — including fully local
  models with zero data leaving your network.
- **Root-cause chat**: ask natural-language questions about what changed
  and why, reasoning over both data changes and FinDiff's own internal
  health events.
- **Auto-Profile**: AI-suggested business labels/descriptions for unmapped
  columns — always surfaced for review, never auto-applied.
- **Persistent, named conversation history** — rename, revisit, and
  continue past conversations, each replaying recent context so follow-up
  questions build on what was already discussed.
- **File uploads**, including PDF text extraction, with a configurable max
  upload size and a clear on-screen indicator of whether a given file's
  content is actually being used by the model.
- **Configurable retention** for AI conversations and their uploaded files
  together (based on last activity, not creation date, so active
  conversations never expire mid-use).
- AI answers are rendered with proper syntax-highlighted code blocks and a
  one-click copy button, and can respond in your chosen UI language.

**Accounts & UI**
- Multi-user accounts: any logged-in user can invite another, who is forced
  through a password change on first login. The built-in `admin` bootstrap
  account and the last remaining account can never be deleted, so an
  instance can never lock itself out.
- Self-service password change for every account.
- Light / dark / system theme, and English / French language — both follow
  your operating system by default, and both are switchable at any time.
- A guided first-run onboarding tour for every genuinely new account.
- A Dashboard with a database activity timeline chart and summary cards.

**Database management**
- Full CRUD for monitored databases directly in the UI: add, edit
  (including renaming and pausing), or delete a database registration —
  changes take effect instantly, no restart required.
- Structured, per-engine connection forms (host/port/username/password/
  database, with an Oracle Service Name vs. SID toggle) instead of a single
  error-prone connection-string field.

## Supported database engines

| Engine | Monitoring | Notes |
|---|---|---|
| PostgreSQL | ✅ | Full support, including watermark and checksum fast-path |
| MySQL | ✅ | Full support |
| MariaDB | ✅ | Full support |
| Microsoft SQL Server | ✅ | Includes native Change Tracking integration (opt-in, DBA-enabled) |
| Oracle | ✅ | Includes native `ORA_ROWSCN` incremental scanning (opt-in) |
| SQLite | ✅ | Supported as a monitored source in addition to being FinDiff's own local store |

## Quick start

FinDiff ships as a single Docker image with the frontend embedded — there
is nothing else to install.

```bash
docker run -d \
  --name findiff \
  -p 8080:8080 \
  -e FINDIFF_MASTER_KEY="$(openssl rand -base64 32)" \
  -e FINDIFF_ADMIN_USERNAME=admin \
  -e FINDIFF_ADMIN_PASSWORD="change-this-immediately" \
  -v findiff_data:/data \
  oubaidhl/findiff:latest
```

Then open `http://localhost:8080`, log in with the admin credentials you
set above, and add your first monitored database from **Databases → Add
database**.

> **Important:** `FINDIFF_MASTER_KEY` encrypts every monitored database's
> connection details at rest. Generate it once, store it somewhere safe
> (a secrets manager, not source control), and never change or lose it —
> losing it makes every stored connection unrecoverable.

## Preparing a production deployment

1. **Generate and store the master key.**
   ```bash
   openssl rand -base64 32
   ```
   Keep this in your secrets manager (Vault, AWS Secrets Manager, a
   Kubernetes `Secret`, etc.) and inject it as `FINDIFF_MASTER_KEY`. Treat
   it with the same care as a database root password — anyone with it and
   access to FinDiff's data directory can decrypt every stored connection
   string.

2. **Create a dedicated read-only role on every database you plan to
   monitor**, and use those credentials — not an admin/superuser account —
   when registering the database in FinDiff. This gives you a real,
   database-enforced safety net on top of FinDiff's own read-only guard.

   ```sql
   -- PostgreSQL example
   CREATE ROLE findiff_reader WITH LOGIN PASSWORD '...';
   GRANT CONNECT ON DATABASE your_db TO findiff_reader;
   GRANT USAGE ON SCHEMA public TO findiff_reader;
   GRANT SELECT ON ALL TABLES IN SCHEMA public TO findiff_reader;
   ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO findiff_reader;
   ```

3. **Persist the data directory.** FinDiff's own SQLite database (schema
   mappings, diff history, settings, encrypted credentials, AI chat
   history and uploaded files) lives under `/data` inside the container.
   Mount a real named volume or bind mount — never run production on an
   ephemeral filesystem.

4. **Put a reverse proxy with TLS in front of it.** FinDiff does not
   terminate TLS itself; run it behind nginx, Caddy, Traefik, or your
   platform's ingress/load balancer, and only expose HTTPS externally.

5. **Set the bootstrap admin account** via `FINDIFF_ADMIN_USERNAME` /
   `FINDIFF_ADMIN_PASSWORD` on first run (only used if no user exists yet
   — safe to leave set on every restart). Change the password immediately
   after first login, and invite named accounts for every other person who
   needs access from **Settings → Users** rather than sharing the bootstrap
   login.

6. **Configure retention** (Settings → Retention, and Settings → AI
   Copilot for chat/upload retention) to a window appropriate for your
   compliance requirements — FinDiff exports data to an archive before
   pruning it, so nothing is silently destroyed.

7. **Configure alerting** (Settings → Alerts) so changes to critical tables
   reach the right Slack channel, Teams channel, email distribution list,
   or internal webhook receiver.

### Minimal `docker-compose.yml` for production

```yaml
services:
  findiff:
    image: oubaidhl/findiff:latest
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      FINDIFF_MASTER_KEY: "${FINDIFF_MASTER_KEY}"
      FINDIFF_ADMIN_USERNAME: "${FINDIFF_ADMIN_USERNAME}"
      FINDIFF_ADMIN_PASSWORD: "${FINDIFF_ADMIN_PASSWORD}"
    volumes:
      - findiff_data:/data

volumes:
  findiff_data:
```

Keep `FINDIFF_MASTER_KEY`, `FINDIFF_ADMIN_USERNAME`, and
`FINDIFF_ADMIN_PASSWORD` in a `.env` file (or your secrets manager) that is
never committed to version control.

## Trying it in a dev/throwaway environment

Since FinDiff's source isn't published, "development" for most users means
trying the app itself against a disposable database — not building it from
source. The fastest way:

```bash
# 1. Start a throwaway Postgres database for testing
docker run -d --name findiff-demo-db \
  -e POSTGRES_PASSWORD=findiff \
  -e POSTGRES_DB=findiff_demo \
  -p 5433:5432 \
  postgres:16

# 2. Start FinDiff, pointed at a local scratch data directory
docker run -d --name findiff-dev \
  -p 8080:8080 \
  -e FINDIFF_MASTER_KEY="$(openssl rand -base64 32)" \
  -e FINDIFF_ADMIN_USERNAME=admin \
  -e FINDIFF_ADMIN_PASSWORD=devpassword \
  -v findiff_dev_data:/data \
  oubaidhl/findiff:latest
```

Then log in at `http://localhost:8080`, add the throwaway Postgres database
from **Databases → Add database** using host `host.docker.internal`
(or your Docker network's DNS name), port `5433`, database `findiff_demo`,
and scan its schema. Create a table, insert a few rows, map it in Schema
Manager, trigger a manual snapshot, mutate a row, snapshot again, and watch
the change appear live in the Diff Timeline — a self-contained way to see
every core feature without touching anything real.

When you're done, tear both containers down:

```bash
docker rm -f findiff-dev findiff-demo-db
docker volume rm findiff_dev_data
```

## Configuration reference

All configuration is via environment variables.

| Variable | Required | Default | Purpose |
|---|---|---|---|
| `FINDIFF_MASTER_KEY` | **Yes** | — | Base64-encoded 32-byte key used to encrypt monitored database credentials at rest. FinDiff refuses to start without it. |
| `FINDIFF_ADMIN_USERNAME` | No | — | Username for the bootstrap admin account, created only if no user exists yet. |
| `FINDIFF_ADMIN_PASSWORD` | No | — | Password for the bootstrap admin account. If neither admin variable is set and no user exists, login will fail until you set them and restart. |
| `FINDIFF_PORT` | No | `8080` | Port the HTTP server listens on inside the container. |
| `FINDIFF_DATA_DIR` | No | `./data` | Where FinDiff's local SQLite store, encrypted credentials, and AI attachment uploads live. Always mount this as a persistent volume. |

Everything else (alert channels, retention windows, AI provider and model,
upload size limits, per-database snapshot intervals) is configured from the
Settings page inside the app itself, and takes effect immediately without a
restart.

## Scaling to very large tables

FinDiff is built to monitor tables from a few dozen rows up to
multi-terabyte, billion-row tables without unbounded memory or storage
growth on the FinDiff side:

- **Keyset pagination**: every table scan pages through rows by primary
  key rather than loading a whole table into memory, so scanning a huge
  table costs bounded memory regardless of table size.
- **In-database checksum fast path**: on most engines, an unchanged table
  is detected with a single lightweight server-side hash computation and
  zero row data transferred, before any per-row comparison is attempted.
- **State storage that doesn't multiply**: FinDiff keeps one current-state
  row per (database, table, primary key), updated in place, rather than a
  fresh full copy of the table on every snapshot — so storage grows with
  the number of distinct rows, not with the number of snapshots taken over
  time.
- **Incremental scanning** via a watermark column (any engine) or
  engine-native change tracking (Oracle `ORA_ROWSCN`, SQL Server Change
  Tracking) for tables where re-scanning everything every interval isn't
  practical — both are strictly opt-in and read-only.

For very large or very hot tables, we recommend starting with a longer
snapshot interval and, where available, enabling watermark or native
change-tracking scanning from that table's settings in Schema Manager.

## Backups and data safety

FinDiff's own data directory (`FINDIFF_DATA_DIR`, `/data` by default in the
Docker image) contains everything FinDiff knows: schema mappings, full diff
history, encrypted connection credentials, settings, and AI conversation
history/uploads. Back it up like you would any other database:

- Snapshot or back up the volume on a schedule appropriate to your
  retention and compliance requirements.
- Keep `FINDIFF_MASTER_KEY` backed up **separately** from the data
  directory itself (e.g. in your secrets manager) — the encrypted
  connection strings inside the data directory are unrecoverable without
  it.
- FinDiff never modifies the databases it monitors, so restoring a FinDiff
  backup only ever affects FinDiff's own observability history — it has no
  effect on the monitored databases themselves.

## FAQ

**Does FinDiff ever write to my database?**
No. See [Core guarantee: read-only, always](#core-guarantee-read-only-always)
above. We recommend connecting with a database-level read-only role as an
additional, independent safety net.

**Can I use FinDiff with a fully local/offline AI model?**
Yes. The AI Copilot works with Ollama, LM Studio, or any other
OpenAI-compatible local server — no data has to leave your network unless
you choose a cloud provider instead.

**Is the source code available?**
FinDiff's source is not publicly published. It's distributed as an
official, versioned Docker image so you always run tested, supported
builds. This repository exists to document the product for the community —
feature requests and bug reports are welcome via GitHub Issues here.

**Can more than one person use the same FinDiff instance?**
Yes — any logged-in user can invite another from Settings → Users. This is
a flat, shared-trust model (no roles/permission levels) intended for a
small team sharing one instance, not a multi-tenant permission system.

**What happens if I lose `FINDIFF_MASTER_KEY`?**
Every monitored database's stored connection details become permanently
unrecoverable — you'd need to re-add each database with a new key. Back
this value up separately from the data directory.

## Support

This repository is the public hub for FinDiff. Use Discussions for questions, feature ideas, and bug reports:

- 💬 [Join the Discussions](https://github.com/oubaidHL/FinDiff-Community/discussions)
- 🐞 [Report a bug](https://github.com/oubaidHL/FinDiff-Community/discussions/categories/report-a-bug)
- 💡 [Share an idea](https://github.com/oubaidHL/FinDiff-Community/discussions/categories/ideas)
- ❓ [Ask a question](https://github.com/oubaidHL/FinDiff-Community/discussions/categories/q-a)

Docker image: https://hub.docker.com/r/oubaidhl/findiff · Website: https://findiff.fr
