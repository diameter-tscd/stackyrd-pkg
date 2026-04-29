# CockroachDB Manager

## Overview

The `CockroachDBManager` is a Go library for managing CockroachDB operations. CockroachDB is PostgreSQL wire-compatible, so this manager uses the standard `database/sql` package along with a retryable HTTP client for robust operations.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **SQL Execution**: Execute queries using standard `database/sql`
- **Transaction Support**: Begin and manage transactions
- **Table Listing**: List all tables with schema, owner, and estimated rows
- **Node Listing**: List cluster nodes with status and version info
- **Cluster Settings**: Get all cluster settings as key-value pairs
- **Connection String**: Generate PostgreSQL connection strings
- **Retry Logic**: HTTP client with retry support (3 retries)
- **Worker Pool**: Async job execution support (6 workers)
- **Status Monitoring**: Get connection status and database information

## Quick Start

```go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "stackyrd/pkg/infrastructure"
    "stackyrd/pkg/logger"
    _ "github.com/lib/pq" // PostgreSQL driver
)

func main() {
    // Initialize logger
    log := logger.NewLogger()
    
    // Create CockroachDB manager (configuration via viper)
    manager, err := cockroachdb.NewCockroachDBManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    // Set the database connection (you need to open it first)
    db, err := sql.Open("postgres", manager.ConnectionString())
    if err != nil {
        panic(err)
    }
    manager.SetDB(db)
    
    ctx := context.Background()
    
    // Execute a query
    rows, err := manager.Query(ctx, "SELECT * FROM users")
    if err != nil {
        panic(err)
    }
    defer rows.Close()
    fmt.Println("Query executed successfully")
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `CockroachDBManager` | Main manager with sql.DB, connection config, worker pool |
| `CRDBTable` | Table info (schema, name, owner, estimated rows) |
| `CRDBNode` | Cluster node (ID, address, SQL address, live status, version) |
| `CRDBRange` | Range info (range ID, start/end keys, replicas) |
| `CRDBZoneConfig` | Zone configuration (target, range bytes, GC TTL) |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   CockroachDBManager                       │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  sql.DB    │  → Standard SQL operations               │
│  │            │  → Transactions, queries, exec          │
│  └────────────────┘                                      │
│                                                         │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ AsyncResult │                   │
│  │  (6 workers)│      │   Channel   │                   │
│  └─────────────┘      └──────────────┘                   │
│         ▲                      │                            │
│         │                      ▼                            │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  SubmitJob  │      │ ExecuteAsync │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                         │
│  Host: CockroachDB host (default: localhost)              │
│  Port: CockroachDB port (default: 26257)                  │
│  Database: Database name (default: defaultdb)                │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewCockroachDBManager(logger)
    │
    ├── Check viper config: "cockroachdb.enabled"
    ├── Get host: "cockroachdb.host" (default: "localhost")
    ├── Get port: "cockroachdb.port" (default: 26257)
    ├── Get database: "cockroachdb.database" (default: "defaultdb")
    ├── Get user: "cockroachdb.user"
    ├── Get password: "cockroachdb.password"
    ├── Get sslmode: "cockroachdb.sslmode" (default: "disable")
    │
    ├── Create retryablehttp.Client:
    │   ├── RetryMax = 3
    │   ├── RetryWaitMin = 1s
    │   ├── RetryWaitMax = 10s
    │   └── HTTPClient.Timeout = 30s
    │
    ├── Test connection: testConnection()
    │   └── DB.PingContext()
    │
    ├── Create WorkerPool(6)
    ├── Start worker pool
    └── Return CockroachDBManager
```

### 2. Exec Flow

```
Exec(ctx, query, args...)
    │
    ├── Check DB != nil
    └── DB.ExecContext(ctx, query, args...)
```

### 3. Query Flow

```
Query(ctx, query, args...)
    │
    ├── Check DB != nil
    └── DB.QueryContext(ctx, query, args...)
```

### 4. Begin Transaction Flow

```
Begin(ctx)
    │
    ├── Check DB != nil
    └── DB.BeginTx(ctx, nil)
```

### 5. List Tables Flow

```
ListTables(ctx)
    │
    ├── Check DB != nil
    ├── Query: "SELECT schema_name, table_name, owner FROM [SHOW TABLES]"
    ├── Iterate rows, scan into CRDBTable
    └── Return []CRDBTable
```

### 6. List Nodes Flow

```
ListNodes(ctx)
    │
    ├── Check DB != nil
    ├── Query: "SELECT node_id, address, sql_address, is_live, version, build_tag FROM [SHOW NODES]"
    ├── Iterate rows, scan into CRDBNode
    └── Return []CRDBNode
```

### 7. Get Cluster Settings Flow

```
GetClusterSettings(ctx)
    │
    ├── Check DB != nil
    ├── Query: "SELECT variable, value FROM [SHOW ALL CLUSTER SETTINGS]"
    ├── Iterate rows, build map[string]string
    └── Return map
```

### 8. Get Status Flow

```
GetStatus()
    │
    ├── If nil: return connected=false
    ├── If DB == nil: return connected=false, error
    ├── PingContext with 5s timeout
    ├── Return connected, host, port, database, pool_active
    └── Note: Tests actual connection with PingContext
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `cockroachdb.enabled` | bool | false | Enable/disable CockroachDB manager |
| `cockroachdb.host` | string | "localhost" | CockroachDB host |
| `cockroachdb.port` | int | 26257 | CockroachDB port |
| `cockroachdb.database` | string | "defaultdb" | Database name |
| `cockroachdb.user` | string | "" | Username |
| `cockroachdb.password` | string | "" | Password |
| `cockroachdb.sslmode` | string | "disable" | SSL mode (disable, require, verify-full) |

### Environment Variables

The CockroachDB manager does not directly use environment variables. Configuration is read through Viper, which can load from:
- Configuration files (JSON, YAML, TOML)
- Environment variables (if Viper is configured for it)

You can set CockroachDB parameters via:
```go
viper.Set("cockroachdb.enabled", true)
viper.Set("cockroachdb.host", "localhost")
viper.Set("cockroachdb.port", 26257)
viper.Set("cockroachdb.database", "mydb")
viper.Set("cockroachdb.user", "root")
viper.Set("cockroachdb.password", "")
viper.Set("cockroachdb.sslmode", "disable")
```

## Usage Examples

### Setting Up the Connection

```go
import (
    "database/sql"
    _ "github.com/lib/pq"
)

// After creating the manager
manager, err := cockroachdb.NewCockroachDBManager(log)
if err != nil {
    panic(err)
}

// Open the database connection using the connection string
db, err := sql.Open("postgres", manager.ConnectionString())
if err != nil {
    panic(err)
}
manager.SetDB(db)
```

### Executing a Query

```go
ctx := context.Background()

// Execute a query without returning rows
result, err := manager.Exec(ctx, "INSERT INTO users (id, name) VALUES ($1, $2)", "user123", "John")
if err != nil {
    panic(err)
}
rowsAffected, _ := result.RowsAffected()
fmt.Printf("Inserted %d rows\n", rowsAffected)
```

### Querying Data

```go
ctx := context.Background()

// Query returning rows
rows, err := manager.Query(ctx, "SELECT id, name FROM users")
if err != nil {
    panic(err)
}
defer rows.Close()

for rows.Next() {
    var id, name string
    if err := rows.Scan(&id, &name); err == nil {
        fmt.Printf("User: %s (%s)\n", id, name)
    }
}
```

### Using Transactions

```go
ctx := context.Background()

// Begin a transaction
tx, err := manager.Begin(ctx)
if err != nil {
    panic(err)
}
defer tx.Rollback()

// Execute queries in transaction
_, err = tx.Exec("INSERT INTO users (id, name) VALUES ($1, $2)", "user123", "John")
if err != nil {
    panic(err)
}

// Commit transaction
if err := tx.Commit(); err != nil {
    panic(err)
}
fmt.Println("Transaction committed")
```

### Listing Tables

```go
ctx := context.Background()

tables, err := manager.ListTables(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Found %d tables:\n", len(tables))
for _, t := range tables {
    fmt.Printf("  %s.%s (owner: %s)\n", t.Schema, t.Name, t.Owner)
}
```

### Listing Nodes

```go
ctx := context.Background()

nodes, err := manager.ListNodes(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Cluster has %d nodes:\n", len(nodes))
for _, n := range nodes {
    fmt.Printf("  Node %d: %s (live: %v, version: %s)\n", n.ID, n.Address, n.Live, n.Version)
}
```

### Getting Cluster Settings

```go
ctx := context.Background()

settings, err := manager.GetClusterSettings(ctx)
if err != nil {
    panic(err)
}

fmt.Println("Cluster settings:")
for k, v := range settings {
    fmt.Printf("  %s = %s\n", k, v)
}
```

### Getting Connection String

```go
// Generate a PostgreSQL connection string
connStr := manager.ConnectionString()
fmt.Printf("Connection string: %s\n", connStr)
// Output: postgresql://user:password@host:port/database?sslmode=disable
```

### Getting Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Host: %s\n", status["host"])
fmt.Printf("Port: %v\n", status["port"])
fmt.Printf("Database: %s\n", status["database"])

if status["connected"] == false {
    if errMsg, ok := status["error"].(string); ok {
        fmt.Printf("Error: %s\n", errMsg)
    }
}
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/hashicorp/go-retryablehttp` | HTTP client with retry logic |
| `github.com/spf13/viper` | Configuration management |
| `github.com/lib/pq` | PostgreSQL driver (import as blank) |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `context`, `database/sql`, `fmt`, `time` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: Returned from `DB.PingContext()`
- **Query failures**: Returned from `DB.QueryContext()`, `DB.ExecContext()`
- **Nil checks**: Public methods check `DB == nil` and return error
- **Nil receiver**: `GetStatus()` handles nil receiver gracefully

Example error handling:

```go
rows, err := manager.Query(ctx, query)
if err != nil {
    if strings.Contains(err.Error(), "connection refused") {
        return fmt.Errorf("cockroachdb not reachable: %w", err)
    }
    if strings.Contains(err.Error(), "authentication failed") {
        return fmt.Errorf("authentication failed: %w", err)
    }
    if strings.Contains(err.Error(), "does not exist") {
        return fmt.Errorf("table/view not found: %w", err)
    }
    return fmt.Errorf("query failed: %w", err)
}
```

## Common Pitfalls

### 1. Database Not Initialized

**Problem**: `cockroachdb DB not initialized` errors

**Solution**:
- Call `manager.SetDB(db)` after opening the database connection
- The manager does not automatically open the connection
- You need to import a PostgreSQL driver (e.g., `github.com/lib/pq`)

### 2. CockroachDB Not Reachable

**Problem**: `connection refused` or `no route to host` errors

**Solution**: 
- Verify CockroachDB is running
- Check host/port in `cockroachdb.host` and `cockroachdb.port`
- Default port is 26257
- Check firewall settings

### 3. Authentication Failed

**Problem**: `authentication failed` or `password authentication failed` errors

**Solution**:
- Verify username and password
- Check if authentication is required
- For insecure mode, you may not need credentials

### 4. SSL Mode Issues

**Problem**: SSL-related connection errors

**Solution**:
- Set `cockroachdb.sslmode` appropriately:
  - `disable`: No SSL (development)
  - `require`: SSL required
  - `verify-full`: SSL with full verification
- For development, `disable` is typically used

### 5. Missing PostgreSQL Driver

**Problem**: `sql: unknown driver "postgres"` error

**Solution**:
- Import a PostgreSQL driver:
  ```go
  import _ "github.com/lib/pq"
  // or
  import _ "github.com/jackc/pgx/v4/stdlib"
  ```
- The driver must be imported before using `sql.Open()`

### 6. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewCockroachDBManager()` succeeded
- Verify `cockroachdb.enabled` is true

## Advanced Usage

### Using CockroachDB-Specific SQL

```go
func useCRDBFeatures(manager *cockroachdb.CockroachDBManager) error {
    ctx := context.Background()
    
    // Use CockroachDB-specific syntax
    // SHOW syntax for introspection
    rows, err := manager.Query(ctx, "[SHOW DATABASES]")
    if err != nil {
        return err
    }
    defer rows.Close()
    
    fmt.Println("Databases:")
    for rows.Next() {
        var dbName string
        if err := rows.Scan(&dbName); err == nil {
            fmt.Printf("  %s\n", dbName)
        }
    }
    return nil
}
```

### Health Check

```go
func healthCheck(manager *cockroachdb.CockroachDBManager) error {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    status := manager.GetStatus()
    if status["connected"] == false {
        return fmt.Errorf("not connected to CockroachDB")
    }
    
    // Additional check: list nodes
    nodes, err := manager.ListNodes(ctx)
    if err != nil {
        return err
    }
    
    liveNodes := 0
    for _, n := range nodes {
        if n.Live {
            liveNodes++
        }
    }
    fmt.Printf("Cluster: %d/%d nodes live\n", liveNodes, len(nodes))
    return nil
}
```

### Batch Operations

```go
func batchInsert(manager *cockroachdb.CockroachDBManager, users []User) error {
    ctx := context.Background()
    
    tx, err := manager.Begin(ctx)
    if err != nil {
        return err
    }
    defer tx.Rollback()
    
    for _, u := range users {
        _, err := tx.Exec("INSERT INTO users (id, name, email) VALUES ($1, $2, $3)",
            u.ID, u.Name, u.Email)
        if err != nil {
            return err
        }
    }
    
    return tx.Commit()
}
```

### Connection String Customization

```go
func customConnection(manager *cockroachdb.CockroachDBManager) {
    // Get the base connection string
    connStr := manager.ConnectionString()
    fmt.Printf("Base: %s\n", connStr)
    
    // You can modify it further if needed
    // The ConnectionString() method returns:
    // postgresql://user:password@host:port/database?sslmode=mode
}
```

## Internal Algorithms

### Connection Test

```
testConnection():
    │
    ├── Check DB != nil
    └── DB.PingContext() with 5s timeout
```

### Table Listing

```
ListTables():
    │
    ├── Query: "SELECT schema_name, table_name, owner FROM [SHOW TABLES]"
    ├── Iterate rows
    │   └── Scan into CRDBTable
    └── Return []CRDBTable
```

### Node Listing

```
ListNodes():
    │
    ├── Query: "SELECT node_id, address, sql_address, is_live, version, build_tag FROM [SHOW NODES]"
    ├── Iterate rows
    │   └── Scan into CRDBNode
    └── Return []CRDBNode
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.