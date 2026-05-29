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

---

## Blind SQLi

Regular SQLi gives you data back directly in the response. Blind SQLi 
means the query still runs but nothing comes back — you have to infer 
the result from something else in the response.

Same attack, different signal.

---

### Conditional responses

The app behaves differently depending on whether your condition is true 
or false. Classic example: "Welcome back" appears on true, disappears 
on false.

**How to extract data:**
Binary search through characters one at a time using SUBSTRING.

```sql
' AND (SELECT SUBSTRING(password,1,1) FROM users WHERE username='administrator') > 'm'--
```

- Welcome back = character is after m, search n-z
- No welcome back = character is m or before, search a-m

Keep halving until you land on the exact character. Repeat for every 
position.

**My insight:** no need to check password length first. Just start 
cracking characters from position 1. When every value starts returning 
true, you've gone past the end of the string — that's how you know 
you're done. Ended up being 20 characters.

Also did it without Intruder, pure manual binary search. Slower but 
you actually understand what's happening at every step. Intruder just 
automates this exact process.

**Labs completed:**
- Blind SQLi with conditional responses — cracked full password manually 
via binary search, logged in as administrator

---

### Conditional errors

The app response looks identical whether your condition is true or false. 
No "welcome back", no visible difference at all. So you manufacture your 
own signal by making the database crash on purpose when the condition is true.

**The trick:** use CASE to trigger a divide-by-zero on true, return 
something harmless on false.

` ` `sql
' AND (SELECT CASE WHEN (condition) THEN 1/0 ELSE 'a' END)='a'--
` ` `

- 500 error = condition true = database crashed = 1/0 executed
- 200 ok = condition false = database happy = returned 'a'

**Why `='a'` at the end:**
AND needs a complete condition to be valid SQL. The CASE expression 
returns either `1/0` or `'a'`, so you complete it with `='a'` to make 
the syntax valid. On the false path `'a'='a'` is fine. On the true path 
the database crashes before it even reaches the comparison.

Binary search is identical to the previous technique, just reading 
500 vs 200 instead of welcome back vs no welcome back.

**Labs completed:**
- Blind SQLi with conditional errors — same binary search approach, 
different signal

---

### Verbose error messages

Easier than both of the above. The database returns a detailed error 
message that actually contains the data you're looking for — it leaks 
it directly into the error output. Turns a blind vulnerability into 
a visible one.

**Why it happens:**
The app doesn't suppress database errors properly, so when you 
trigger the right kind of error the database includes the query result 
inside the error message itself.

**The payload:** use CAST to force a type conversion error that 
carries the data with it.

` ` `sql
' AND 1=CAST((SELECT password FROM users WHERE username='administrator') AS int)--
` ` `

The database tries to cast the password string as an integer, fails, 
and throws an error that includes the actual password value in the 
message. You're not guessing character by character — the full value 
comes back in one shot.

**My insight:** the error message is basically giving you the blueprint 
of the query. The database is telling you exactly what it tried to do 
and what value caused the problem. Once you see that you just read the 
password out of the error.

**Labs completed:**
- Extracting sensitive data via verbose SQL error messages — used CAST 
to trigger a type error, password leaked directly in the error response

---
 
### Time-based blind
 
The hardest blind technique. Unlike conditional responses (welcome back
or not) and conditional errors (500 vs 200), here there is no visible
difference in the response at all — nothing changes no matter what you
do. The only signal you have is whether the page takes longer to respond.
You trigger a time delay when a condition is true, nothing when false.
That's your only way to prove anything about the data inside the database.
 
**The payloads by database:**
 
| Database | Payload |
|---|---|
| PostgreSQL | `'||pg_sleep(10)--` |
| MySQL | `' AND SLEEP(10)--` |
| MSSQL | `'; WAITFOR DELAY '0:0:10'--` |
| Oracle | `'||dbms_pipe.receive_message(('a'),10)--` |
 
**Why PostgreSQL and Oracle use `||` instead of `;` or `AND`:**
Those two databases don't reliably allow stacked queries — meaning you
can't end the original query with `;` and start a new one. `||` keeps
you inside the original query. The database has to evaluate `pg_sleep`
or `dbms_pipe` to build the string, and the delay happens as a side
effect of that evaluation. MSSQL and MySQL allow stacked queries or
inline conditions so you can use `AND` or `;` directly.
 
Even though PortSwigger's cheat sheet says PostgreSQL supports `;`
stacking, whether it actually works depends on the app's database
driver and how the developer wrote the query execution code. When
stacked queries fail, `||` inline concatenation is your fallback for
PostgreSQL and Oracle.
 
**The actual process on an unknown target:**
 
```
Try each DB's time delay payload one by one
    ↓
Page hangs ~10 seconds = DB confirmed + injection confirmed in one shot
    ↓
Nothing works? → Suspect the HTTP layer before suspecting your SQL
    ↓
URL encode your special characters and retry
```
 
**The HTTP layer problem:**
There are two layers between you and the database — HTTP and SQL. When
something doesn't work you have to figure out which layer is the problem.
 
If you're injecting into a cookie, `;` has a specific meaning in HTTP —
it's the separator between cookies. So `TrackingId=xyz'; SELECT...`
gets split by HTTP before it ever reaches the database. The SQL query
only receives the part before the `;`. The fix is URL encoding:
 
- `;` → `%3B`
- space → `+` or `%20`
- `'` → `%27` if it's getting stripped
**My insight:**  The skill is knowing
what to try and in what order, not magically knowing which DB it is
upfront. Time-based blind is considered the most painful SQLi technique
even in real bug bounties because you're flying completely blind and
every character matters.
 
**Labs completed:**
- Blind SQLi with time delays — confirmed PostgreSQL using `||pg_sleep(10)--`
inside a cookie, page delayed 10 seconds confirming injection
- Blind SQLi with time delays and conditional extraction — had to URL
encode `;` as `%3B` and space as `+` because the cookie HTTP layer was
treating `;` as a cookie separator and eating the rest of the payload
 