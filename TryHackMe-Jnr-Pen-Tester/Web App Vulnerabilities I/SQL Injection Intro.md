# SQL Injection: A General Guide

SQL injection (SQLi) happens when user input is inserted into a database query without proper sanitization, letting an attacker alter the query's logic or structure. Below are the main categories, with the general method for each.

## 1. Union-Based SQLi (In-Band)

Used when query results are reflected directly on the page.

**General method:**
- **Find the column count.** Append `UNION SELECT` with an increasing number of placeholder values (`1`, `1,2`, `1,2,3`...) until no error occurs. This works because a `UNION` requires both queries to return the same number of columns.
- **Make output visible.** Use an ID/value that returns no legitimate row (e.g. `0`) so only your injected values render on the page.
- **Identify which column reflects.** Note which position appears in visible content — that's your extraction column.
- **Enumerate the database.** Use built-in functions and metadata tables:
  - `database()` – current database name
  - `information_schema.tables` – list tables (filter by `table_schema`)
  - `information_schema.columns` – list columns (filter by `table_name`)
  - `group_concat()` – merge multiple results into a single field when only one output column is available
- **Extract data.** Query the target table/columns directly (e.g. credentials).

## 2. Authentication Bypass

Used against login forms where a query decides access based on whether it returns a row.

**General method:**
- Inject logic that makes the `WHERE` clause always true, e.g. `' OR 1=1`.
- Comment out the rest of the query (e.g. `-- ` or `#`) so trailing conditions (like a password check) are never evaluated.
- If the app logs in whenever a row is returned, this typically authenticates as the first matching user.

## 3. Boolean-Based Blind SQLi

Used when the page gives no data back, but a visible true/false signal exists (e.g. a status message, different page content, or a JSON flag).

**General method:**
- Confirm the injection point works with an always-true condition and observe the "true" signal.
- Extract data one character at a time using `LIKE` with wildcards, narrowing down each position:
  - Test each candidate character/prefix (e.g. `'a%'`, `'b%'`, ...) until the true signal appears.
  - Once found, fix that character and move to the next position.
- Apply this technique progressively to: database name → table names → column names → actual data (usernames, passwords, etc.), using `information_schema` to enumerate structure before extracting content.

## 4. Time-Based Blind SQLi

Used when there is *no* observable difference in the response at all — same content, same status, regardless of true/false.

**General method:**
- Use a delay function (e.g. `SLEEP(n)`) inside the injected query.
- Find the column count first, since `SLEEP()` only executes if the `UNION` structure is valid — a delay confirms both injection and correct column count.
- Extract data the same character-by-character way as boolean-based SQLi, but read the *response time* instead of response content:
  - A delay = condition true (character matches)
  - An immediate response = condition false
- This is the slowest technique, since every character requires a separate timed request, but it works when no other signal is available.

## Key Takeaways

- Escalate techniques based on what feedback the application gives you: **visible data → union-based**, **true/false page behavior → boolean-blind**, **nothing at all → time-based**.
- `information_schema` is the universal tool for discovering database/table/column names before extracting real data.
- Comment sequences (`--`, `#`) are essential for neutralizing the rest of a query after your injection.
- In real engagements, tools like SQLmap automate character-by-character extraction — doing it manually helps you understand *why* it works and how to adapt when automated tools get blocked or rate-limited.
