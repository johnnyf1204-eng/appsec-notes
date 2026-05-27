# SQL Injection (SQLi)

## What is it
When an app pastes user input directly into a SQL query, letting you 
change what the query does.

## Why it works
The app treats user input as code instead of data. No separation 
between the query structure and the data inside it.

## How to find it
- Add a single quote `'` and see if the app throws an error
- Try `' OR 1=1--` and see if behavior changes
- Look for any input that might touch a database (login, search, filters)

## UNION attacks
Used to extract data from other tables.

**Rules for UNION to work:**
- Must match the exact number of columns as the original query
- Data types must be compatible per column

**How to find column count:**
- `ORDER BY 1--`, `ORDER BY 2--` etc. until it breaks
- Or add NULLs: `UNION SELECT NULL--`, `UNION SELECT NULL,NULL--` etc.

**How to find string columns:**
- Replace NULLs with `'a'` one at a time until it works

**My insight:** if only one column accepts strings but you need 
two values, concatenate them in that column:
`UNION SELECT NULL,username||'~'||password FROM users--`

## Labs completed
- WHERE clause hidden data
- Login bypass
- UNION determine column count
- UNION find string column
- UNION retrieve data from other tables
- UNION retrieve multiple values in one column — figured out column 
count first, then which column accepts strings, then concatenated:
  `' UNION SELECT NULL,username||'~'||password FROM users--`

## How to fix it
Parameterized queries / prepared statements. Always. Never 
concatenate user input into a query string.

---

## Examining the database

### Getting the database version
The syntax differs depending on the database type:

| Database | Query |
|---|---|
| Oracle | `SELECT * FROM v$version` |
| MySQL / MSSQL | `SELECT @@version` |
| PostgreSQL | `SELECT version()` |

Useful because once you know the database type you know exactly what 
syntax to use for everything else.

### Listing tables
To see all tables in the database:

- **Oracle:** query `all_tables` — returns `table_name`
- **Everything else:** query `information_schema.tables` — returns `table_name`

### Listing columns
Once you have a table name, get its columns:

- **Oracle:** `SELECT column_name FROM all_tab_columns WHERE table_name='target_table'`
- **Everything else:** `SELECT column_name FROM information_schema.columns WHERE table_name='target_table'`

### The full flow every time
```
List tables
    ↓
Spot the interesting one
    ↓
List columns of that table
    ↓
Dump the columns you want
```

### Finding the right table
PortSwigger randomizes table and column names on purpose — so you 
can't just guess `users` and move on. You have to actually enumerate.

My process when the table list has nothing obvious: search for anything 
containing `user`, then `auth`, then `member`, then variations until 
something looks right. It's not always the first thing you check — that's 
the point. Real databases have hundreds of tables and the one you want 
won't announce itself.

**My insight:** the randomization isn't just an obstacle, it's teaching 
the actual skill. In a real target the table is called whatever the 
developer named it three years ago. Enumeration is the job.

## Labs completed (examining the database)
- Query database type and version — Oracle
- Query database type and version — MySQL/MSSQL
- Listing database contents (non-Oracle) — listed tables via 
`information_schema.tables`, found `users_aopibe`, listed its columns 
via `information_schema.columns`, dumped credentials
- Listing database contents — Oracle version using `all_tables` 
and `all_tab_columns`
