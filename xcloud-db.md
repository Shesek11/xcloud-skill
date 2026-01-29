---
allowed-tools: Bash(ssh:*), Bash(mysql:*), Bash(mariadb:*), Read, Write, Glob
description: Connect to MariaDB database and execute queries
---

## Context

- Current directory: !`pwd`

## Your Task: Database Operations

You need to connect to the MariaDB database for this project and execute queries.

### Step 1: Find Project Context

Locate the `.xcloud/context.md` file:
1. Check current directory for `.xcloud/context.md`
2. Check parent directories if not found
3. If no context file, inform user to run `/xcloud-init` first

### Step 2: Read Database Configuration

From context file, extract:
- DB Host
- DB Port
- DB Name
- DB Username
- Whether SSH tunnel is needed
- SSH details (if tunnel needed)

### Step 3: Ask What User Wants to Do

Use AskUserQuestion:
- **Run Query** - Execute a specific SQL query
- **Show Schema** - Display all tables and their structure
- **Export Table** - Export a table to CSV/SQL
- **Search Data** - Find specific records
- **Interactive Session** - Open mysql client

### Step 4: Establish Connection

**If SSH Tunnel Required:**

First, inform the user they need to establish an SSH tunnel. Provide the command:
```bash
# Run this in a separate terminal and keep it open:
ssh -p [ssh-port] -L 3307:[db-host]:[db-port] [ssh-user]@[ssh-host] -N
```

Then connect via:
```bash
mysql -h 127.0.0.1 -P 3307 -u [db-user] -p [db-name]
```

**If Direct Connection:**
```bash
mysql -h [db-host] -P [db-port] -u [db-user] -p [db-name]
```

### Step 5: Execute Operations

**For Run Query:**
Ask for the query, then execute:
```bash
mysql -h [host] -P [port] -u [user] -p [db-name] -e "[query]"
```

**For Show Schema:**
```bash
mysql -h [host] -P [port] -u [user] -p [db-name] -e "SHOW TABLES; SELECT TABLE_NAME, COLUMN_NAME, DATA_TYPE, IS_NULLABLE, COLUMN_KEY FROM INFORMATION_SCHEMA.COLUMNS WHERE TABLE_SCHEMA='[db-name]' ORDER BY TABLE_NAME, ORDINAL_POSITION;"
```

Save schema to `.xcloud/context.md` under "Database Schema" section.

**For Export Table:**
Ask which table, then:
```bash
mysql -h [host] -P [port] -u [user] -p [db-name] -e "SELECT * FROM [table]" > export_[table]_[date].csv
```

**For Search Data:**
Ask for table and search criteria, construct appropriate SELECT query.

### Step 6: Update Context

After schema queries, update the "Database Schema" section in `.xcloud/context.md`:
```markdown
## Database Schema
Last updated: [DATE]

### Tables
| Table | Description |
|-------|-------------|
| users | User accounts |
| ... | ... |

### Key Tables Detail
#### [table_name]
- id (int, PK)
- name (varchar)
- ...
```

Add to Development Log:
```markdown
### [DATE] [TIME] - Database Query
- Query: [description]
- Tables affected: [tables]
- Result: [outcome]
```

## Important Notes

- Password is prompted by mysql client - never ask for or store it
- For large exports, consider using mysqldump
- Always be careful with UPDATE/DELETE queries - suggest using transactions
- Read-only queries are safer; warn before write operations
- If using SSH tunnel, remind user to keep the tunnel terminal open
