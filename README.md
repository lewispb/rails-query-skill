# rails-query

A Claude Code plugin that teaches Claude to answer data questions about a Rails 8.2+ application using the [`rails query`](https://github.com/rails/rails/pull/57156) command — locally or against a deployed environment via Kamal.

## What it does

When you ask Claude something like:

- "How many active users do we have?"
- "Show me the orders table structure."
- "What models reference `accounts`?"
- "Explain the query plan for this scope."

…Claude reaches for `rails query` instead of opening a console, SSHing to a server, or writing a one-off script. Queries are structured (JSON output), read-only by construction (writes blocked at the connection level), and safe to run against production with a read replica.

The skill covers:

- **Both paths** — local (`bin/rails query`) and remote via Kamal (`bin/kamal query -d <env>`)
- **All four input modes** — ActiveRecord expressions, raw SQL (`--sql`), schema/model introspection (`schema`, `models`), and `EXPLAIN` plans
- **Pagination** — how to page through large results, and why you shouldn't add your own `LIMIT`
- **The Kamal quoting rule** — the one gotcha that bites when your SQL has parens or stars, and the reliable fix

## Installation

From inside Claude Code:

```
/plugin install rails-query
```

Or install from this repository directly via a marketplace config.

## Requirements

- **Rails 8.2+** (the app you're querying). Earlier versions don't ship the `rails query` command.
- **`jq`** on your PATH if you want the skill's suggested extraction patterns to work (`jq '.rows[0][0]'`).
- **For remote use: Kamal** with an alias added to `config/deploy.yml`:

  ```yaml
  aliases:
    query: 'app exec -q --reuse -p -r console "rails query"'
  ```

  Your `console` role should be configured with read-replica DB access. See the skill body for the reasoning behind each flag.

## How triggering works

The skill is described broadly so Claude invokes it whenever a task would naturally be solved by querying a Rails database — not just when the user types `/rails-query:rails-query`. Try:

> "How many users registered last week?"

> "What's in the `mailboxes` table?"

> "Explain this query: `Post.published.limit(100)`"

## Further reading

- [rails/rails#57156](https://github.com/rails/rails/pull/57156) — the PR that added `rails query` to Rails 8.2. Read this for the full design, rationale, and command surface.
- [Kamal `app` commands](https://kamal-deploy.org/docs/commands/app/) — for wiring `bin/kamal query` to `rails query` on deployed environments.

## Contributing

Issues and PRs welcome at https://github.com/lewispb/rails-query-skill.

## License

MIT
