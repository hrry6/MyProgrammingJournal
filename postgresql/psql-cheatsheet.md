# psql CLI Cheatsheet

[🏠 Home](../README.md) > [🐘 PostgreSQL](./README.md)

## Quick Reference

```bash
# Connect to database
psql -U <username> -d <database> -h <host> -p <port: default 5432>

# Connect using connection URI
psql postgresql://<username>:<password>@<host>:<port>/<database>

# Execute a single query and exit
psql -U <username> -d <database> -c "SELECT version();"

# Run SQL commands from a file
psql -U <username> -d <database> -f <file.sql>
```

## Essential Meta-Commands

### Database & Connection

| Command | Action |
| :--- | :--- |
| `\l` or `\list` | List all databases |
| `\c <db_name>` | Switch to another database |
| `\conninfo` | Display current connection information |
| `\q` | Quit `psql` |

### Tables & Schema Inspection

| Command | Action |
| :--- | :--- |
| `\dt` | List all tables in current schema |
| `\dt+` | List all tables with size and description |
| `\dt *.*` | List all tables across all schemas |
| `\d <table_name>` | Describe table structure (columns, types, constraints) |
| `\d+ <table_name>` | Describe table with extra details (storage, stats, description) |
| `\dn` | List all schemas |
| `\di` | List all indexes |
| `\dv` | List all views |
| `\df` | List all functions |
| `\ds` | List all sequences |

### Users & Permissions

| Command | Action |
| :--- | :--- |
| `\du` | List all roles and users |
| `\du+` | List all roles with descriptions |
| `\dp` or `\z` | List table, view, and sequence access privileges |

### Query Execution & Output Formatting

| Command | Action |
| :--- | :--- |
| `\x` | Toggle expanded display (great for wide tables) |
| `\timing` | Toggle query execution time measurement |
| `\e` | Open query buffer in default text editor |
| `\ef <func_name>` | Edit function definition in text editor |
| `\i <file.sql>` | Execute SQL commands from an external file |
| `\o <file.txt>` | Send all query results to a file (run `\o` again to stop) |
| `\watch <sec>` | Re-run the previous query every `<sec>` seconds |

### Help & History

| Command | Action |
| :--- | :--- |
| `\?` | List all available `psql` meta-commands |
| `\h` | List all SQL commands |
| `\h <SQL_COMMAND>` | Show syntax help for a specific SQL command (e.g., `\h ALTER TABLE`) |
| `\s` | Display command history |
| `\s <file.txt>` | Save command history to a file |
