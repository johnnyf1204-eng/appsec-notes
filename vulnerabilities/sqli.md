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