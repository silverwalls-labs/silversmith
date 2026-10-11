# SQL injection

An attacker injects SQL syntax through untrusted input that is
concatenated into a query string. The database executes the modified
query — reading, modifying, or deleting data the application never
intended to expose. Parameterized queries eliminate the vulnerability
entirely: the query structure is fixed at compile/prepare time, and
values are bound separately, so the database never parses attacker input
as SQL.

The principle is language-agnostic: every database driver in every
language supports parameterized queries. The examples below use Python;
the same pattern applies to Java (PreparedStatement), Node.js
(`pg`/`mysql2` with `$1` / `?`), Go (`database/sql`), C# (SqlCommand
with `@param`), etc.

## Before (vulnerable)

```python
# UNSAFE — user input interpolated directly into the query string.
# An attacker can supply: username = "admin' OR '1'='1"
# and bypass authentication, or worse.

def get_user(conn, username):
    query = f"SELECT * FROM users WHERE username = '{username}'"
    cursor = conn.execute(query)
    return cursor.fetchone()
```

String concatenation, f-strings, `%`-formatting, and `str.format()` all
produce the same result: the database receives a query where the
attacker's input is parsed as SQL syntax.

## After (safe)

```python
# SAFE — parameterized query. The database driver sends the query
# structure and the value separately; the database never parses
# the value as SQL.

def get_user(conn, username):
    query = "SELECT * FROM users WHERE username = ?"
    cursor = conn.execute(query, (username,))
    return cursor.fetchone()
```

The placeholder (`?` for sqlite3, `%s` for psycopg/MySQLdb, `$1` for
asyncpg) is driver-specific — use whatever the driver documents. The
critical invariant: the query string is a **static literal** with
placeholders; values are passed as a separate tuple/list/dict.

## Key points

- **Every query, every time.** There is no "this value is just a number"
  exception — validate types upstream, but still parameterize.
- **ORM escape hatches reintroduce risk.** Django's `raw()`,
  SQLAlchemy's `text()`, Prisma's `$queryRawUnsafe` — apply the same
  parameterization rules to every raw-SQL call. (Note: Prisma's
  `$queryRaw` uses tagged template literals that auto-parameterize —
  it is the `$queryRawUnsafe` variant that accepts plain strings and
  is vulnerable to injection.)
- **Table and column names cannot be parameterized.** If these come from
  user input, validate them against an allowlist of known identifiers —
  never interpolate them directly.
- **Stored procedures are not inherently safe.** A procedure that
  internally concatenates its arguments into dynamic SQL is just as
  vulnerable — parameterize inside the procedure too.
