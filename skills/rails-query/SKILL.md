---
name: rails-query
description: |
  Answer any question that requires live data from a Rails 8.2+ app using `rails query`:
  counts, lookups, "how many X", ad-hoc investigation, schema exploration, EXPLAIN plans,
  and aggregates — whether locally or against a deployed environment via Kamal. Trigger
  broadly whenever the user asks for data from a Rails app's database, mentions
  `rails query` / `bin/rails query` / `bin/kamal query`, asks to inspect schema, tables,
  or models, wants to run SQL or ActiveRecord against the database, needs an EXPLAIN
  plan, or wants to query production/staging/replica. Prefer this skill over opening a
  Rails console, SSH-ing to a server, or writing a one-off script for data questions —
  `rails query` is faster, structured (JSON output), audit-friendly, and read-only by
  construction.
---

# rails-query

`rails query` is a Rails 8.2+ command for running read-only queries against the database. Input is a single expression, output is a single JSON object, and writes are blocked at the connection level. With a read replica configured it hits the replica automatically, so it's safe to point at production.

Prefer it over a console, a script, or SSH for any data question. One invocation, structured output, nothing to clean up.

## The flow

For any data question, work through these steps. Skip the ones you don't need — but don't skip step 3 when the schema might surprise you.

### 1. Know where you're running

- **Local** (`bin/rails query`) — for development, tests, or anywhere you have the app checked out and a local DB.
- **Remote via Kamal** (`bin/kamal query -d <destination>`) — when the app is deployed with Kamal. Requires a small alias in `config/deploy.yml` (see "Kamal setup" below).

If you don't know whether Kamal is configured, check `config/deploy.yml` for an `aliases:` block. No alias → pass SSH or ask the user.

### 2. Decide: Ruby expression or raw SQL

**Default: Ruby (ActiveRecord).** The expression is `eval`'d in the app context. Everything you'd type in a console works — scopes, associations, finders, aggregates. Model logic (encryption, default scopes, polymorphism) is honored.

**`--sql`:** use for raw schema access, cross-table joins without models, or aggregates that are awkward to express in AR.

A telltale sign you forgot `--sql`: `SyntaxError: unexpected *; no anonymous rest parameter`. That means your SQL got parsed as Ruby. Add `--sql` and retry.

### 3. Inspect the schema if it's unfamiliar

When the app is new to you, or the table/column names are ambiguous, find out what's actually there before guessing.

In-repo with source access, you can read `db/schema.rb` or model files directly. But `rails query` has three first-class introspection modes that work the same way locally and remotely — crucial when you only have Kamal access:

```bash
# List every table
bin/rails query schema

# Full detail for a table: columns + indexes + enums + associations
bin/rails query schema users

# Every ActiveRecord model with its table and associations
bin/rails query models
```

Enums matter: if a column is an enum, `User.where(status: "active")` works but `User.where(status: 0)` might silently misfire. `schema <table>` tells you.

**For remote (Kamal) exploration, cache the schema locally before `jq`-ing it.** Each `bin/kamal query` round-trip is 10-30s (SSH + container exec + Rails boot), so re-running `bin/kamal query -d staging '"schema users"' | jq ...` three times across a task can cost you a minute of wall time for zero reason — the schema hasn't changed between calls.

Dump once, read from disk many times:

```bash
# Pick a cache dir tied to the destination so different envs don't collide.
CACHE=/tmp/rails-query-cache-staging
mkdir -p "$CACHE"

# Warm the cache — one round-trip per file.
bin/kamal query -d staging '"schema"'       2>/dev/null > "$CACHE/tables.json"
bin/kamal query -d staging '"models"'       2>/dev/null > "$CACHE/models.json"
bin/kamal query -d staging '"schema users"' 2>/dev/null > "$CACHE/schema-users.json"

# Now `jq` against the cache — zero network.
jq '.columns[] | {name, type}' "$CACHE/schema-users.json"
jq '[.rows[][0]] | length'     "$CACHE/tables.json"
```

Rules of thumb:

- Skip caching for a single-query task. Caching is pure overhead when you only look once.
- Cache is session-scoped. If a migration ships during your exploration, blow away `$CACHE` and re-dump.
- Locally (`bin/rails query`), don't bother caching — there's no SSH overhead and it's already fast.

### 4. For expensive queries, `EXPLAIN` first

```bash
# Ruby relation
bin/rails query explain 'User.where(active: true).order(:created_at).limit(100)'

# Raw SQL
bin/rails query explain 'SELECT * FROM users WHERE active = 1' --sql
```

Do this before running full-table scans against production.

### 5. Run the query

```bash
bin/rails query 'User.count'
bin/rails query --sql 'SELECT COUNT(*) FROM users'
```

Watch for `"has_more": true` in the response — that means pagination is truncating. Re-run with `--page N` or raise `--per`. Don't add your own `LIMIT`; see "Pagination" below.

### 6. Extract what you need

The output is JSON on stdout. Pipe through `jq`:

```bash
# Single scalar
bin/rails query 'User.count' | jq '.rows[0][0]'

# Column as an array
bin/rails query 'User.pluck(:email)' | jq -r '.rows[][0]'

# Pretty table
bin/rails query 'User.limit(5).pluck(:id, :email)' | jq '.columns as $c | .rows[] | [$c, .] | transpose | map(join(": ")) | .[]'
```

## Command reference

```bash
bin/rails query [OPTIONS] '<expression>'
bin/kamal query -d <destination> [OPTIONS] '<expression>' 2>/dev/null
```

| Flag | Default | Meaning |
|------|---------|---------|
| `--sql` | off | Treat `<expression>` as raw SQL instead of Ruby/AR |
| `--db <name>` | — | Explicit database config (e.g. `primary_replica`, `analytics`) |
| `-e <env>` | `development` | Environment (`test`, `production`, …) |
| `--page N` | 1 | Page number (1-indexed) |
| `--per N` | 100 | Rows per page (max 10000) |

### Special expressions (no `--sql` needed)

| Expression | What it returns |
|------------|-----------------|
| `schema` | Every table name |
| `schema <table>` | Columns, indexes, enums, and associations for that table |
| `models` | Every AR model with its table and associations |
| `explain <expr>` | `EXPLAIN` plan. Pair with `--sql` for raw SQL |
| `-` (or piped stdin) | Read expression from stdin — useful for long multi-line SQL |

## Ruby return types and how they're shaped

When the Ruby expression returns…

| Return type | Output columns / rows |
|-------------|-----------------------|
| `ActiveRecord::Relation` | SQL is executed; paginated rows returned |
| `ActiveRecord::Result` | Returned as-is |
| `Hash` | Two columns: `key`, `value` |
| `Array` | One column per element: `column_0`, `column_1`, … |
| Anything else (integer, string, boolean) | Single `result` column with one row |

## Output shape

Every successful query returns one JSON object on stdout:

```json
{
  "columns": ["id", "email"],
  "rows": [[1, "alice@example.com"], [2, "bob@example.com"]],
  "meta": {
    "row_count": 2,
    "query_time_ms": 3.4,
    "page": 1,
    "per_page": 100,
    "has_more": false,
    "sql": "SELECT id, email FROM users LIMIT 101"
  }
}
```

Errors are JSON on stderr with a non-zero exit:

```json
{"error": "ActiveRecord::StatementInvalid: ...", "meta": {"query_time_ms": 0}}
```

## Pagination

`rails query` paginates automatically. It internally appends `LIMIT per+1` to detect whether more rows exist, which drives `meta.has_more`.

**Don't add your own `LIMIT`.** If the SQL already contains `LIMIT`, the command won't add one — which suppresses the truncation detector. For raw SQL queries, omit `LIMIT` and let pagination handle it.

**To page through results:**

```bash
bin/rails query --sql 'SELECT id, email FROM users ORDER BY id' --page 1
bin/rails query --sql 'SELECT id, email FROM users ORDER BY id' --page 2

# Or bump per-page (max 10000)
bin/rails query --per 500 'User.order(:id)'
```

Always order explicitly when paginating — without `ORDER BY`, page 2 can overlap page 1 depending on the engine.

## Kamal setup

If the app deploys with Kamal, add this to `config/deploy.yml`:

```yaml
aliases:
  query: 'app exec -q --reuse -p -r console "rails query"'
```

What each flag does:

- `-q` — quiet mode: only the query's JSON lands on stdout. Without it, Kamal prints SSH status lines that corrupt downstream `jq`.
- `--reuse` — runs inside an existing container instead of booting a fresh one. Fast.
- `-p` — pins to the primary host. A role spanning multiple hosts would otherwise emit duplicate JSON (one per host).
- `-r console` — runs on the `console` role. Your `console` role should be configured with read-replica DB access so queries don't touch the writer.

Then:

```bash
bin/kamal query -d production 'User.count' 2>/dev/null
bin/kamal query -d production --sql 'SELECT COUNT(*) FROM users' 2>/dev/null
```

`2>/dev/null` suppresses SSH noise; the JSON result goes to stdout.

### The Kamal quoting rule

When your expression contains shell metacharacters — especially `(`, `)`, `*`, `;`, `&`, `|`, `<`, `>` — the remote shell will eat a single layer of quoting. The argument passes through your local shell, Ruby arg parsing in Kamal, SSH, and finally `bash -c` on the remote host; one of those strips quotes.

**The reliable pattern:** outer single quotes, inner double quotes.

```bash
# WORKS — inner double quotes survive to the remote shell and protect the parens
bin/kamal query -d production --sql '"SELECT COUNT(*) FROM users"'
bin/kamal query -d production '"User.where(active: true).count"'

# FAILS — bash: -c: syntax error near unexpected token '('
bin/kamal query -d production --sql 'SELECT COUNT(*) FROM users'
bin/kamal query -d production --sql "SELECT COUNT(*) FROM users"
```

For SQL containing single-quoted string literals, break out and re-enter the outer quote:

```bash
bin/kamal query -d production --sql '"SELECT id FROM users WHERE email = '"'"'alice@example.com'"'"'"'
```

…or switch to the Ruby form, which is usually cleaner:

```bash
bin/kamal query -d production '"User.where(email: \"alice@example.com\").pick(:id)"'
```

**Locally, single-layer quoting works as normal** — the nested-quote dance is purely a Kamal remoting artifact:

```bash
bin/rails query --sql 'SELECT COUNT(*) FROM users'
```

## Common patterns

### Counts and aggregates

```bash
bin/rails query 'User.count'
bin/rails query 'User.where(active: true).count'
bin/rails query 'User.group(:role).count'
bin/rails query 'User.pick("COUNT(*), AVG(age), MIN(created_at)")'
```

### Lookups

```bash
bin/rails query 'User.find(123).as_json'
bin/rails query 'User.find_by(email: "alice@example.com")&.as_json'
bin/rails query 'User.where(active: true).pluck(:id, :email, :created_at)'
```

### Joins and scopes

Ruby form lets you reuse existing scopes and encryption without reimplementing them:

```bash
bin/rails query 'Post.published.joins(:author).where(authors: { verified: true }).count'
```

### Schema-first discovery

```bash
# What tables exist?
bin/rails query schema | jq -r '.rows[][0]'

# What columns on orders?
bin/rails query schema orders | jq '.columns[] | {name, type, null}'

# Which models touch the accounts table?
bin/rails query models | jq '.[] | select(.table_name == "accounts") | .model'
```

### Long SQL via stdin

Locally (stdin doesn't forward cleanly through the Kamal alias as written):

```bash
bin/rails query --sql - <<'SQL'
  SELECT u.id, u.email, COUNT(o.id) AS order_count
  FROM users u
  LEFT JOIN orders o ON o.user_id = u.id
  WHERE u.created_at > '2024-01-01'
  GROUP BY u.id, u.email
  ORDER BY order_count DESC
SQL
```

### Extracting a value into a shell variable

```bash
COUNT=$(bin/rails query 'User.count' | jq -r '.rows[0][0]')
```

## Safety model

- Writes are blocked at the connection level. The command uses `while_preventing_writes` or (when configured) `connected_to(role: :reading)`. `INSERT` / `UPDATE` / `DELETE` — including via AR methods — raises instead of executing.
- With a read replica configured, queries hit the replica automatically.
- `--db <name>` overrides the connection — useful for targeting a specific configured database (e.g. `--db primary_replica`, `--db analytics`).

This means you can safely run `rails query` against production without worrying about accidental mutations. If you see `ActiveRecord::ReadOnlyError`, that's the safety net firing — your query tried to write something. Rework the expression to be read-only.

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `SyntaxError: unexpected *` | SQL passed without `--sql` | Add `--sql` |
| `bash: -c: syntax error near unexpected token '('` | Kamal path, single-layer quoting | Switch to `'"..."'` nested quotes |
| `ActiveRecord::ReadOnlyError` | Expression tried to write | Rework to be read-only; `rails query` is read-only by design |
| Empty JSON or duplicated output over Kamal | Missing `-p` or `-q` in the alias | Add them to `config/deploy.yml` |
| `LIMIT 101` in `meta.sql` unexpectedly | Default pagination probe | Expected — drives `meta.has_more` |
| `has_more: true` but you wanted all rows | Default per-page hit | Raise `--per` (max 10000) or paginate with `--page` |
| `ActiveRecord::ConnectionNotEstablished` with `--db` | Database key not in `database.yml` | Check the env's `database.yml` for the exact key |

## Further reading

- Rails source: `railties/lib/rails/commands/query/query_command.rb` in the Rails repo
- Kamal aliases: https://kamal-deploy.org
