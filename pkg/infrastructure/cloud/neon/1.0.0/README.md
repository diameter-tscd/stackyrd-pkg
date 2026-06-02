# Neon Manager

## Overview

The `NeonManager` is a comprehensive Go library for interacting with [Neon](https://neon.tech) — the serverless PostgreSQL platform. It provides database connectivity via `pgx/v5/stdlib` (database/sql compatible), optional GORM ORM support, a full asynchronous wrapper layer with batch operations, worker pool concurrency, and a complete HTTP-based Neon Management API v2 client — covering projects, branches, databases, compute endpoints, roles, and operations — all as a self-contained plugin that requires zero changes to the central configuration structs.

**Import Path:** `stackyrd/pkg/infrastructure`

**Libraries:** `github.com/jackc/pgx/v5/stdlib` (PostgreSQL driver), `gorm.io/gorm` + `gorm.io/driver/postgres` (ORM), standard `net/http` (Management API)

## Features

- **Serverless PostgreSQL**: Connect to any Neon database with full `database/sql` compatibility
- **PgBouncer Pooling Support**: Configurable pooled vs. direct connection mode
- **Rich SQL API**: `Query`, `Select`, `QueryRow`, `Exec`, `Insert`, `Update`, `Delete`, `ExecuteRawQuery`, `Prepare`
- **GORM ORM**: Full GORM support with async `Create`, `Find`, `First`, `Update`, `Delete`
- **Neon Management API v2**: Complete project, branch, database, endpoint, role, and operation management
- **Branch Management**: List, get, create, delete, set primary — all with async wrappers
- **Compute Endpoint Lifecycle**: Start, suspend, restart, create, update, delete endpoints
- **Role Management**: List, get, create, delete, reset passwords
- **Database Introspection**: `GetDBInfo` (version, size, timezone, sessions), `GetRunningQueries`, `GetTableSizes`, `GetIndexUsage`
- **Multi-Branch Connection Manager**: `NeonConnectionManager` for connecting to multiple branches simultaneously
- **Connection URI Generator**: Build Neon-compatible connection strings from components
- **Complete Async Support**: Every operation has an `*Async` counterpart returning `*AsyncResult[T]`
- **Batch Execution**: `ExecuteBatchAsync` for concurrent query execution
- **Worker Pool**: 10-worker pool for async jobs + `SubmitAsyncJob`
- **Status & Health**: TTL-cached `GetStatus()` with connection pool and API availability stats
- **Plugin Architecture**: 100% self-contained — configuration read exclusively via Viper, registered via `init()`
- **Graceful Disable**: Returns `nil, nil` when `neon.enabled=false`

## Quick Start

```go
package main

import (
	"context"
	"fmt"
	"stackyrd/pkg/infrastructure"
	"stackyrd/pkg/logger"
)

func main() {
	log := logger.NewLogger()

	manager, err := infrastructure.NewNeonManager(log)
	if err != nil {
		panic(err)
	}
	if manager == nil {
		fmt.Println("Neon disabled in config")
		return
	}
	defer manager.Close()

	ctx := context.Background()

	// Create table and insert
	manager.Exec(ctx, `CREATE TABLE IF NOT EXISTS users (id SERIAL PRIMARY KEY, name TEXT, email TEXT UNIQUE)`)

	id, _ := manager.Insert(ctx, `INSERT INTO users (name, email) VALUES ($1,$2)`, "Alice", "alice@example.com")
	fmt.Printf("Inserted ID: %d\n", id)

	// Raw query
	rows, _ := manager.ExecuteRawQuery(ctx, "SELECT * FROM users")
	fmt.Printf("Users: %+v\n", rows)

	// Management API — list branches
	if manager.Management != nil {
		branches, _ := manager.Management.ListBranches(ctx)
		for _, b := range branches {
			fmt.Printf("Branch: %s (%s, primary=%v)\n", b.Name, b.ID, b.Primary)
		}
	}
}
```

## Architecture

### Core Structs

| Struct                      | Description                                      |
|----------------------------|--------------------------------------------------|
| `NeonManager`              | Main manager wrapping `*sql.DB`, GORM, pool, mgmt API |
| `NeonManagementClient`     | Neon HTTP Management API v2 client               |
| `NeonBranch`               | A Neon project branch (with state, size, parent) |
| `NeonDatabase`             | A database within a branch                       |
| `NeonEndpoint`             | A compute endpoint (with host, pooler, state)    |
| `NeonRole`                 | A database role (name, protected, password)      |
| `NeonProject`              | Neon project metadata (region, pg version)      |
| `NeonOperation`            | An async project operation (action, status)      |
| `NeonConnectionDetails`    | Parsed connection URI with pooled alternative    |
| `NeonBranchConnection`     | An active DB connection to a specific branch     |
| `NeonConnectionManager`    | Multi-branch connection manager                  |
| `PGQuery`                  | A running PostgreSQL query with wait/block info  |
| `neonConfig` (local)       | Internal configuration shape                     |

### Concurrency Model

```mermaid
flowchart TB
    subgraph NeonManager
        direction TB
        DB["*sql.DB<br/>(pgx/v5/stdlib)"]
        ORM["GORM<br/>(gorm.io/gorm)"]
        Pool["WorkerPool<br/>(10 workers)"]
        Cache["Status Cache<br/>(2s TTL + mutex)"]
        Mgmt["NeonManagementClient<br/>(HTTP)"]
    end

    subgraph AsyncLayer
        AsyncResult["AsyncResult[T]"]
        BatchResult["BatchAsyncResult"]
    end

    DB -->|Query / Select / Exec / Insert<br/>Update / Delete / RawQuery / Prepare| SyncOps["Sync Operations"]
    ORM -->|Create / Find / First / Update / Delete| SyncOps
    Mgmt -->|Branches / DBs / Endpoints / Roles<br/>Operations / Project| SyncOps

    SyncOps -->|GetRunningQueries / GetTableSizes<br/>GetIndexUsage / GetDBInfo / GetSessionCount| Monitor["Monitoring"]

    Pool --> AsyncResult
    Pool --> BatchResult
    Pool -->|SubmitAsyncJob| CustomJobs["Custom Jobs"]

    SyncOps --> Cache
    AsyncResult --> SyncOps
    BatchResult --> SyncOps

    subgraph ConnectionManager["NeonConnectionManager (multi-branch)"]
        Branch1["Branch: production"]
        Branch2["Branch: staging"]
        Branch3["Branch: feature-x"]
    end
```

### State Diagram

```mermaid
stateDiagram-v2
    [*] --> ConfigLoading: NewNeonManager(l)
    ConfigLoading --> Disabled: neon.enabled=false
    ConfigLoading --> Connecting: neon.enabled=true

    Connecting --> Ping: sql.Open("pgx", dsn)
    Ping --> GORMInit: Ping OK
    GORMInit --> PoolInit: GORM ready
    PoolInit --> MgmtInit: WorkerPool started

    MgmtInit --> Ready: api_key & project_id configured
    MgmtInit --> Ready: no management API

    Disabled --> [*]

    state Ready {
        [*] --> Connected
        Connected --> WarmingUp: cold start
        WarmingUp --> Connected: compute active
        Connected --> Suspended: auto-suspend (5m idle)
        Suspended --> WarmingUp: query triggers wake
    }

    Ready --> [*]: Close()
```

## How It Works

### 1. Initialization Flow

```
NewNeonManager(l)
    │
    ├── viper.UnmarshalKey("neon", &neonConfig)
    ├── !cfg.Enabled → return nil, nil
    ├── Build DSN from connection_string or decomposed fields
    ├── sql.Open("pgx", dsn)
    ├── Ping with 10s timeout
    ├── SetMaxOpenConns, SetMaxIdleConns, SetConnMaxLifetime(30m)
    ├── GORM init via postgres.New(Config{Conn: sqlDB})
    ├── NewWorkerPool(10).Start()
    ├── Parse NeonConnectionDetails from host/user/password/database/port
    ├── If cfg.APIKey && cfg.ProjectID:
    │     └── newNeonManagementClient(apiKey, projectID, hc)
    └── Return NeonManager{DB, ORM, Pool, Management, connectionURI}
```

### 2. Management API Flow

```
Management.ListBranches(ctx)
    │
    ├── GET .../api/v2/projects/{project_id}/branches
    ├── Authorization: Bearer {api_key}
    ├── Parse response {branches: [...]}
    └── return []NeonBranch

Management.ResetRolePassword(ctx, branchID, roleName)
    │
    ├── POST .../branches/{branch_id}/roles/{role_name}/reset_password
    ├── Parse response {role: {..., password: "new-pw"}}
    └── return *NeonRole (with new password)
```

### 3. Async Wrapper Flow

```
InsertAsync(ctx, sql, args)
    │
    └── ExecuteAsync(ctx, func() (int64, error) {
            return Insert(ctx, sql, args...)
        })
    └── Returns *AsyncResult[int64] with Done channel
```

### 4. Batch Async Flow

```
ExecuteBatchAsync(ctx, queries, args)
    │
    ├── length validation → error if mismatch
    ├── build []AsyncOperation[sql.Result]
    └── ExecuteBatchAsync(ctx, ops, 10)
```

### 5. Status Caching Flow

```
GetStatus()
    │
    ├── statusMu + check 2s TTL cache → return cached
    ├── actual DB.Ping()
    ├── collect sql.DB.Stats()
    ├── management_api configured flag
    ├── store in cache + update expiry
    └── return fresh stats
```

## Configuration

### Viper Configuration Options (plugin-style — no central struct)

| Key                             | Type   | Default | Description                                      |
|--------------------------------|--------|---------|--------------------------------------------------|
| `neon.enabled`                 | bool   | false   | Enable/disable the Neon plugin                   |
| `neon.connection_string`       | string | ""      | Full PostgreSQL DSN                              |
| `neon.pooled`                  | bool   | false   | Use PgBouncer pooled connection                  |
| `neon.host`                    | string | ""      | Database host (used when connection_string empty) |
| `neon.port`                    | int    | 5432    | Database port                                    |
| `neon.user`                    | string | ""      | Database user                                    |
| `neon.password`                | string | ""      | Database password                                |
| `neon.database`                | string | ""      | Database name                                    |
| `neon.api_key`                 | string | ""      | Neon Management API key                          |
| `neon.project_id`              | string | ""      | Neon project ID                                  |
| `neon.max_open_conns`          | int    | 10      | Max open connections                             |
| `neon.max_idle_conns`          | int    | 5       | Max idle connections                             |
| `neon.branches`                | array  | []      | Multi-branch connection configs (for NeonConnectionManager) |

### Multi-Branch Config Example

When using `NeonConnectionManager`, define multiple branches:

```yaml
neon:
  enabled: true
  api_key: "your-api-key"
  project_id: "proj-xxx"
  branches:
    - name: production
      connection_string: "postgres://user:pass@ep-prod.us-east-2.aws.neon.tech/neondb?sslmode=require"
      pooled: true
      max_open_conns: 20
      max_idle_conns: 5
    - name: staging
      connection_string: "postgres://user:pass@ep-staging.us-east-2.aws.neon.tech/neondb?sslmode=require"
      pooled: true
      max_open_conns: 5
      max_idle_conns: 2
```

**Environment variable mapping** (automatic via Viper):
- `NEON_ENABLED=true`
- `NEON_CONNECTION_STRING=postgres://...`
- `NEON_API_KEY=your-api-key`
- `NEON_PROJECT_ID=proj-xxx`
- `NEON_BRANCHES_0_NAME=production`
- `NEON_BRANCHES_0_CONNECTION_STRING=postgres://...`

### Example YAML — Full Feature

```yaml
neon:
  enabled: true
  connection_string: "postgres://user:pass@ep-green-water-123456.us-east-2.aws.neon.tech/neondb?sslmode=require"
  pooled: false
  api_key: "your-api-key"
  project_id: "proj-xxx"
  max_open_conns: 10
  max_idle_conns: 5
```

## Usage Examples

### Basic CRUD

```go
ctx := context.Background()

manager.Exec(ctx, `CREATE TABLE IF NOT EXISTS products (id SERIAL PRIMARY KEY, name TEXT, price NUMERIC)`)
manager.Insert(ctx, `INSERT INTO products (name, price) VALUES ($1, $2)`, "Laptop", 1299.99)
manager.Update(ctx, `UPDATE products SET price=$1 WHERE name=$2`, 1199.99, "Laptop")
manager.Delete(ctx, `DELETE FROM products WHERE id=$1`, 1)

rows, _ := manager.Query(ctx, `SELECT * FROM products WHERE price > $1`, 100)
defer rows.Close()
```

### Select (alias for Query)

```go
rows, _ := manager.Select(ctx, "SELECT id, name FROM users WHERE active = $1", true)
```

### Prepared Statements

```go
ctx := context.Background()

stmt, _ := manager.Prepare(ctx, "INSERT INTO logs (level, message) VALUES ($1, $2)")
defer stmt.Close()

stmt.ExecContext(ctx, "info", "Server started")
stmt.ExecContext(ctx, "warn", "Disk space low")
```

### Raw Query (map results)

```go
results, _ := manager.ExecuteRawQuery(ctx, "SELECT id, name, price FROM products LIMIT 100")
for _, row := range results {
    fmt.Printf("%v: %v ($%v)\n", row["id"], row["name"], row["price"])
}
```

### Database Introspection

```go
info, _ := manager.GetDBInfo(ctx)
fmt.Printf("Version: %v\n", info["version"])
fmt.Printf("Database: %v\n", info["db_name"])
fmt.Printf("Size: %v\n", info["size"])
fmt.Printf("Timezone: %v\n", info["timezone"])
fmt.Printf("Active Sessions: %v\n", info["active_sessions"])
```

### Running Queries

```go
queries, _ := manager.GetRunningQueries(ctx)
for _, q := range queries {
    fmt.Printf("PID=%d User=%s Duration=%s Query=%s\n", q.PID, q.User, q.Duration, q.Query)
}
```

### Table Sizes

```go
tables, _ := manager.GetTableSizes(ctx)
for _, t := range tables {
    fmt.Printf("%s.%s: %s (%v rows)\n", t["schema"], t["table_name"], t["total_size"], t["estimated_rows"])
}
```

### Index Usage

```go
stats, _ := manager.GetIndexUsage(ctx)
for _, s := range stats {
    fmt.Printf("%s.%s [%s]: %d scans\n", s["schema"], s["table_name"], s["index_name"], s["scans"])
}
```

### Async Operations

```go
result := manager.InsertAsync(ctx, "INSERT INTO users (name) VALUES ($1)", "Bob")
id, err := result.WaitWithTimeout(5 * time.Second)
```

### Batch Async

```go
queries := []string{
    "INSERT INTO events (type) VALUES ($1)",
    "UPDATE stats SET count = count + 1",
}
args := [][]interface{}{{"login"}, {}}
batch := manager.ExecuteBatchAsync(ctx, queries, args)
values, errors := batch.WaitAll()
```

### GORM Operations

```go
type User struct {
    ID    uint   `gorm:"primaryKey"`
    Name  string
    Email string
}

manager.ORM.AutoMigrate(&User{})
manager.ORM.Create(&User{Name: "Alice", Email: "alice@example.com"})
manager.ORM.Where("name = ?", "Alice").First(&user)
manager.ORM.Model(&user).Update("Email", "alice@newdomain.com")
```

### GORM Async

```go
// Create
manager.GORMCreateAsync(ctx, &User{Name: "Bob", Email: "bob@example.com"})

// Find all
var users []User
manager.GORMFindAsync(ctx, &users)

// First with condition
var user User
manager.GORMFirstAsync(ctx, &user, "email = ?", "bob@example.com")

// Update
manager.GORMUpdateAsync(ctx, &User{}, map[string]interface{}{"email": "bob@new.org"}, "id = ?", 1)

// Delete
manager.GORMDeleteAsync(ctx, &User{}, "id = ?", 1)
```

### Neon Management API — Branches

```go
ctx := context.Background()
m := manager.Management

// List
branches, _ := m.ListBranches(ctx)

// Get by ID
branch, _ := m.GetBranch(ctx, "br-xxx")

// Create from primary
branch, _ = m.CreateBranch(ctx, "feature-payments", "")

// Set as primary
m.SetPrimaryBranch(ctx, branch.ID)

// Delete
m.DeleteBranch(ctx, "br-xxx")
```

### Neon Management API — Databases

```go
// List databases in a branch
databases, _ := m.ListDatabases(ctx, "br-xxx")

// Get specific database
db, _ := m.GetDatabase(ctx, "br-xxx", "mydb")

// Create new database
db, _ = m.CreateDatabase(ctx, "br-xxx", "analytics", "alex")

// Delete
m.DeleteDatabase(ctx, "br-xxx", "analytics")
```

### Neon Management API — Endpoints

```go
// List endpoints
endpoints, _ := m.ListEndpoints(ctx)

// Get by ID
ep, _ := m.GetEndpoint(ctx, "ep-xxx")

// Start suspended endpoint
ep, _ = m.StartEndpoint(ctx, "ep-xxx")

// Suspend running endpoint
ep, _ = m.SuspendEndpoint(ctx, "ep-xxx")

// Restart
ep, _ = m.RestartEndpoint(ctx, "ep-xxx")

// Create new endpoint
ep, _ = m.CreateEndpoint(ctx, "br-xxx", "read_only", 300)

// Update settings (e.g., suspend timeout)
ep, _ = m.UpdateEndpoint(ctx, "ep-xxx", map[string]interface{}{
    "suspend_timeout_seconds": 600,
})

// Delete
m.DeleteEndpoint(ctx, "ep-xxx")
```

### Neon Management API — Roles

```go
// List roles
roles, _ := m.ListRoles(ctx, "br-xxx")

// Get role
role, _ := m.GetRole(ctx, "br-xxx", "alex")

// Create role
role, _ = m.CreateRole(ctx, "br-xxx", "analyst")

// Reset password (returns new password)
role, _ = m.ResetRolePassword(ctx, "br-xxx", "analyst")
fmt.Printf("New password: %s\n", role.Password)

// Delete role
m.DeleteRole(ctx, "br-xxx", "analyst")
```

### Neon Management API — Project & Operations

```go
// Project info
project, _ := m.GetProject(ctx)
fmt.Printf("Project: %s (PG %d, region: %s)\n", project.Name, project.PgVersion, project.RegionID)

// List operations
ops, _ := m.ListOperations(ctx)
for _, op := range ops {
    fmt.Printf("Operation: %s - %s (%s)\n", op.Action, op.Status, op.CreatedAt)
}

// Get specific operation
op, _ := m.GetOperation(ctx, "op-xxx")
```

### Connection URI Generator

```go
details := infrastructure.GetConnectionURI(
    "ep-green-water-123456.us-east-2.aws.neon.tech",
    5432,
    "neondb",
    "alex",
    "AbC123dEf",
    false,
)
fmt.Println(details.URI)
// postgres://alex:AbC123dEf@ep-green-water-123456.us-east-2.aws.neon.tech:5432/neondb?sslmode=require

// Also accessible from the manager when built from individual fields
if uri := manager.ConnectionURI(); uri != nil {
    fmt.Printf("Pooled: %s\n", uri.PooledURI)
}
```

### Multi-Branch Connection Manager

```go
// Connect to multiple branches via Viper config (neon.branches)
cm, err := infrastructure.NewNeonConnectionManager(manager.Management, log)
if err != nil {
    panic(err)
}
defer cm.CloseAll()

// Use a specific branch
conn, err := cm.GetConnection("production")
if err != nil {
    panic(err)
}

rows, _ := conn.DB.QueryContext(ctx, "SELECT count(*) FROM users")
// Or use GORM on the branch
var count int64
conn.ORM.Model(&User{}).Count(&count)

// List available connections
for _, name := range cm.ListConnections() {
    fmt.Printf("Connected to: %s\n", name)
}

// Status of all branches
status := cm.GetStatus()
```

### Submit Custom Async Job

```go
manager.SubmitAsyncJob(func() {
    manager.Exec(context.Background(), "VACUUM ANALYZE")
})
```

## Error Handling

All methods follow standard Go error patterns. `GetStatus()` returns `connected=false` on nil receiver or failed ping. Errors are prefixed for easy debugging:

- `neon: failed to open connection: ...`
- `neon: failed to connect: ...`
- `neon: database connection is nil`
- `neon: neither connection_string nor host configured`
- `neon: queries and args length mismatch`
- `neon api: status 401: ...` — invalid API key
- `neon api: status 404: ...` — resource not found
- `neon: management API not configured`
- `neon: branch "prod" not connected` — NeonConnectionManager lookup failure
- `neon: branch "prod": ping: ...` — branch connection failure

## Common Pitfalls

### 1. Connection String vs Individual Fields
Use `connection_string` for simplicity (paste from Neon Console). When both are set, `connection_string` takes precedence. If using individual fields, `host` is required.

### 2. SSL Requirement
Neon requires SSL. The plugin appends `sslmode=require` when building from individual fields. For custom `connection_string`, ensure `sslmode=require` is present.

### 3. Pooled Connections
Neon provides pooled (via PgBouncer) and direct connections. Use the `-pooler` host suffix for PgBouncer. Set `pooled: true` for accurate status reporting.

### 4. Free Tier Limits
Neon's free tier limits concurrent connections (typically 10-20). Set `max_open_conns` conservatively (e.g., 5) on free tier. Monitor via `GetStatus()`.

### 5. Auto-Suspend Cold Starts
Neon compute endpoints auto-suspend after 5 minutes of inactivity. First query after suspension triggers a 1-5s cold start. Use `StartEndpoint` or `RestartEndpoint` proactively via the Management API.

### 6. Branch Creation Latency
Creating branches from large databases can take time. `CreateBranch` is synchronous via the API — use `CreateBranchAsync` to avoid blocking.

### 7. Management API Key Scope
API keys must have sufficient permissions for the operations you call (e.g., `endpoint:start` requires `endpoints_allow` scope). Insufficient permissions return HTTP 403.

### 8. Branch Connection Strings
When using `NeonConnectionManager`, each branch needs its own `connection_string`. Use the Neon Console to get connection strings for each branch.

## Advanced Usage

### Transactions

```go
tx, _ := manager.DB.BeginTx(ctx, nil)
defer tx.Rollback()
tx.ExecContext(ctx, "UPDATE products SET price=$1 WHERE id=$2", 99.99, 1)
tx.ExecContext(ctx, "INSERT INTO price_changes (product_id, old_price) VALUES ($1, $2)", 1, 1299.99)
tx.Commit()
```

### Prepared Statement with Async

```go
// Create prepared statement asynchronously
result := manager.PrepareAsync(ctx, "INSERT INTO audit (user_id, action) VALUES ($1, $2)")
stmt, err := result.Wait()
if err != nil {
    panic(err)
}
defer stmt.Close()

// Use in goroutines
stmt.ExecContext(ctx, 42, "login")
stmt.ExecContext(ctx, 42, "view_page")
```

### Branch-Based Workflow

```go
// 1. Create branch for a feature
branch, _ := manager.Management.CreateBranch(ctx, "feat-42", "")
endpoints, _ := manager.Management.ListEndpoints(ctx)

// 2. Find the endpoint for the new branch
var ep *NeonEndpoint
for _, e := range endpoints {
    if e.BranchID == branch.ID {
        ep = &e
        break
    }
}

// 3. Build connection URI from endpoint details
details := infrastructure.GetConnectionURI(
    ep.Host, ep.Port, "neondb", "alex", "password",
    false,
)
fmt.Println("Branch connection URI:", details.URI)
```

### Monitoring Dashboard

```go
// Gather all monitoring data
ctx := context.Background()

info, _ := manager.GetDBInfo(ctx)
queries, _ := manager.GetRunningQueries(ctx)
tables, _ := manager.GetTableSizes(ctx)
indexStats, _ := manager.GetIndexUsage(ctx)
sessions, _ := manager.GetSessionCount(ctx)
status := manager.GetStatus()

fmt.Printf("Database: %s | Sessions: %d | Size: %s\n",
    info["db_name"], sessions, info["size"])
fmt.Printf("Open conns: %v | In use: %v | Idle: %v\n",
    status["open_connections"], status["in_use"], status["idle"])

if len(queries) > 0 {
    fmt.Println("Running queries:")
    for _, q := range queries {
        fmt.Printf("  [%s] %s (%s)\n", q.User, q.Query, q.Duration)
    }
}
```

## Internal Algorithms

(See "How It Works" section above for the numbered flows.)

## Dependencies

| Dependency                              | Role                                      |
|-----------------------------------------|-------------------------------------------|
| `github.com/jackc/pgx/v5`              | PostgreSQL driver (stdlib-compatible)     |
| `gorm.io/gorm`                         | ORM framework                             |
| `gorm.io/driver/postgres`              | GORM PostgreSQL driver                    |
| `github.com/spf13/viper`               | Configuration (plugin style)              |
| `stackyrd/pkg/logger`                  | Structured logging                        |
| `stackyrd/pkg/infrastructure` (internal) | WorkerPool, AsyncResult, registry         |
| Standard library                       | `database/sql`, `context`, `net/http`, `sync`, `time`, `fmt` |

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.
