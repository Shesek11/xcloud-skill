---
allowed-tools: Bash(ssh:*), Bash(mysql:*), Bash(mariadb:*), Read, Write, Glob, AskUserQuestion
description: Connect to MariaDB database and execute queries, view schema, export tables, or search data. Supports SSH tunneling for non-public databases.
---

## Context

- Current directory: !`pwd`

## Database Operations

### Step 1: Find Project Context

Locate `.xcloud/context.md`:
1. Check current directory for `.xcloud/context.md`
2. Check parent directories if not found
3. If no context file, inform user to run `/xcloud-init` first

### Step 2: Read Database Configuration

From context file, extract:
- DB Host, Port, Name, Username
- Whether SSH tunnel is needed
- SSH details (if tunnel needed)

### Step 3: Test Connection First

Before any operation, verify the database is reachable:
```bash
mysql -h [host] -P [port] -u [user] --password [db-name] -e "SELECT 1" 2>&1
```

If this fails:
- **Access denied**: wrong credentials, verify with hosting provider
- **Can't connect**: DB may not be publicly accessible, try SSH tunnel
- **Command not found**: mysql client not installed (see Important Notes)

### Step 4: Ask What to Do

Use AskUserQuestion:
- **Run Query** - Execute a specific SQL query
- **Show Schema** - Display all tables and their structure
- **Export Table** - Export a table to CSV/SQL
- **Search Data** - Find specific records
- **Interactive Session** - Open mysql client

### Step 5: Establish Connection

**If SSH Tunnel Required:**

Inform the user to establish an SSH tunnel in a separate terminal:
```bash
# Run in a separate terminal and keep open:
ssh -p [ssh-port] -L 3307:[db-host]:[db-port] [ssh-user]@[ssh-host] -N
```

Then connect via:
```bash
mysql -h 127.0.0.1 -P 3307 -u [db-user] --password [db-name]
```

**If Direct Connection:**
```bash
mysql -h [db-host] -P [db-port] -u [db-user] --password [db-name]
```

### Step 6: Execute Operations

**Run Query:**
Ask for the query, then execute:
```bash
mysql -h [host] -P [port] -u [user] --password [db-name] -e "[query]"
```

**Show Schema:**

First get tables and columns, then save to context:
```bash
mysql -h [host] -P [port] -u [user] --password [db-name] -e "SHOW TABLES; SELECT TABLE_NAME, COLUMN_NAME, DATA_TYPE, IS_NULLABLE, COLUMN_KEY FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_SCHEMA='[db-name]' ORDER BY TABLE_NAME, ORDINAL_POSITION;"
```

Save schema to `.xcloud/context.md` under "Database Schema" section. **Always verify column names from the schema before using them in queries** — do not assume column names from code or previous conversations.

**Export Table:**
Ask which table, then export (note: default output is tab-delimited):
```bash
mysql -h [host] -P [port] -u [user] --password [db-name] --batch --raw -e "SELECT * FROM [table]" > export_[table]_[date].tsv
```

For large tables, use mysqldump instead:
```bash
mysqldump -h [host] -P [port] -u [user] --password [db-name] [table] > dump_[table]_[date].sql
```

**Search Data:**
Ask for table and search criteria, construct appropriate SELECT query.

### Step 7: Update Context

After schema queries, update the "Database Schema" section in `.xcloud/context.md`.

Add to Development Log:
```markdown
### [DATE] [TIME] - Database Query
- Query: [description]
- Tables affected: [tables]
- Result: [outcome]
```

## Important Notes

- Password is prompted by mysql client — never ask for or store it
- **Always run schema first** before writing queries that reference specific column names — do not guess column names from code
- For large exports, use mysqldump instead of SELECT *
- Always be careful with UPDATE/DELETE queries — suggest using transactions
- Read-only queries are safer; warn before write operations
- If using SSH tunnel, remind user to keep the tunnel terminal open
- If `mysql` client is not installed:
  - Windows: `winget install MariaDB.Client` or download from mariadb.org
  - Mac: `brew install mysql-client`
  - Linux: `sudo apt install mariadb-client`
