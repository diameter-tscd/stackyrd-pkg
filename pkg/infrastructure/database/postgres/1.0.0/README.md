# PostgreSQL Manager

## Overview

The `PostgresManager` is a Go library for managing PostgreSQL database connections. It provides raw SQL operations via `database/sql`, ORM capabilities via GORM, and supports multiple connections via `PostgresConnectionManager`. It uses the `github.com/jackc/pgx/v5/stdlib` driver with configurable connection pool settings and a worker pool for async operations.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Single Connection**: `PostgresManager` for a single PostgreSQL database connection
- **Multiple Connections**: `PostgresConnectionManager` for managing multiple named PostgreSQL connections
- **SQL Operations**: Query (returning rows), QueryRow (single row), Exec (no rows), Select, Insert, Update, Delete
- **Raw Query Execution**: Execute raw SQL queries and get results as slice of maps
- **GORM Integration**: Full GORM support for ORM operations
- **Monitoring Helpers**: GetRunningQueries, GetSessionCount, GetDBInfo
- **Connection Pool**: Configurable connection pool settings via GORM/PGX
- **Async Operations**: Async versions of most operations via `ExecuteAsync`
- **Batch Operations**: Batch execution of multiple queries via `ExecuteBatchAsync`
- **Worker Pool**: Async job execution support (15 workers per connection)
- **Status Monitoring**: Get connection status, database stats, session info
- **Transaction Support**: Use GORM transactions or raw SQL transactions

## Quick Start

```go
package main

import (
    "context"
    "fmt"
    "stackyrd/config"
    "stackyrd/pkg/infrastructure"
    "stackyrd/pkg/logger"
    _ "github.com/jackc/pgx/v5/stdlib"
)

func main() {
    // Initialize logger
    log := logger.NewLogger()
    
    // Configuration (typically via viper/config)
    cfg := config.PostgresConfig{
        Enabled:  true,
        Host:     "localhost",
        Port:     5432,
        User:     "postgres",
        Password: "password",
        DBName:   "mydb",
        SSLMode:  "disable",
    }
    
    // Create PostgreSQL manager
    manager, err := postgres.NewPostgresDB(cfg, log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
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
| `PostgresManager` | Single PostgreSQL connection with raw SQL DB, GORM DB, worker pool |
| `PostgresConnectionManager` | Manages multiple named `PostgresManager` connections |
| `PostgresConfig` | Configuration for a single connection (host, port, user, password, dbname, sslmode) |
| `PostgresMultiConfig` | Configuration for multiple connections |
| `PGQuery` | PostgreSQL query info (pid, user, db, state, duration, query) |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   PostgresManager                         │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  sql.DB     │  → Raw SQL operations                  │
│  │  (pool)     │  → Query, QueryRow, Exec, Select      │
│  └────────────────┘                                      │
│                                                         │
│  ┌────────────────┐                                      │
│  │  gorm.DB    │  → ORM operations                      │
│  │             │  → AutoMigrate, Create, etc.          │
│  └────────────────┘                                      │
│                                                         │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ AsyncResult │                   │
│  │  (15 workers)│     │   Channel   │                   │
│  └─────────────┘      └──────────────┘                   │
│         ▲                      │                            │
│         │                      ▼                            │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  SubmitJob  │      │ ExecuteAsync │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                         │
│  Host: PostgreSQL host (from config)                        │
│  Port: PostgreSQL port (default: 5432)                      │
│  Database: database name (from config)                      │
└─────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────┐
│              PostgresConnectionManager                      │
├─────────────────────────────────────────────────────────────┤
│  connections: map[string]*PostgresManager                │
│  mu: sync.RWMutex                                       │
│                                                         │
│  Methods:                                                │
│  - GetConnection(name) → *PostgresManager, bool           │
│  - GetDefaultConnection() → *PostgresManager, bool        │
│  - GetAllConnections() → map[string]*PostgresManager      │
│  - CloseAll() → error                                    │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Single Connection Initialization Flow

```
NewPostgresDB(cfg, logger)
    │
    ├── Check cfg.Enabled
    ├── Build DSN: "host=host port=port user=user password=password dbname=dbname sslmode=sslmode"
    ├── Open raw SQL connection: sql.Open("pgx", dsn)
    ├── Ping to verify connection
    ├── Initialize GORM with existing SQL connection:#
    │   └── gorm.Open(postgres.New(postgres.Config{Conn: sqlDB}), &gorm.Config{})
    │
    ├── Create WorkerPool(15)
    ├── Start worker pool
    └── Return PostgresManager
```

### 2. Multiple Connections Initialization Flow

```
NewPostgresConnectionManager(cfg)
    │
    ├── Check cfg.Enabled
    ├── Create PostgresConnectionManager with empty connections map
    ├── For each connCfg in cfg.Connections:#    │   ├── Skip if !connCfg.Enabled
    │   ├── Convert to PostgresConfig
    │   ├── Call NewPostgresDB(connCfg, logger)
    │   ├── If error: log and continue
    │   └── Store in connections[connCfg.Name]
    │
    └── Return PostgresConnectionManager
```

### 3. SQL Operations Flow

```
Query(ctx, query, args...)
    │
    └── DB.QueryContext(ctx, query, args...) → *sql.Rows

QueryRow(ctx, query, args...)
    │
    └── DB.QueryRowContext(ctx, query, args...) → *sql.Row

Exec(ctx, query, args...)
    │
    └── DB.ExecContext(ctx, query, args...) → sql.Result

Select(ctx, query, args...)
    │
    └── Query(ctx, query, args...) → *sql.Rows

Insert(ctx, query, args...)
    │
    ├── Exec(ctx, query, args...)
    └── Return result.RowsAffected()

Update(ctx, query, args...)
    │
    ├── Exec(ctx, query, args...)
    └── Return result.RowsAffected()

Delete(ctx, query, args...)
    │
    ├── Exec(ctx, query, args...)
    └── Return result.RowsAffected()
```

### 4. Raw Query Execution Flow

```
ExecuteRawQuery(ctx, query)
    │
    ├── DB.QueryContext(ctx, query)
    ├── Get columns
    ├── For each row: scan into []interface{}
    ├── Build rowMap: column → value (convert []byte to string)
    └── Return []map[string]interface{}
```

### 5. Monitoring Flows

```
GetRunningQueries(ctx)
    │
    ├── Query pg_stat_activity for active queries
    ├── Scan pid, usename, datname, state, duration, query
    └── Return []PGQuery

GetSessionCount(ctx)
    │
    ├── Query: SELECT count(*) FROM pg_stat_activity
    └── Return count

GetDBInfo(ctx)
    │
    ├── Get version: SELECT version()
    ├── Get size: SELECT pg_size_pretty(pg_database_size(current_database()))
    ├── Get db name: SELECT current_database()
    ├── Get current user: SELECT current_user
    ├── Get SSL status: SELECT COALESCE((SELECT 'enable' FROM pg_stat_ssl WHERE pid = pg_backend_pid() AND ssl = true), 'disable')
    └── Return map with version, size, db_name, user, ssl_mode
```

### 6. Get Status Flow

```
GetStatus()
    │
    ├── If nil or DB == nil: return connected=false
    ├── Ping: DB.Ping()
    ├── Get DB stats: DB.Stats()
    └── Return connected, open_connections, in_use, idle, wait_count, wait_duration_ms
```

### 7. Connection Manager Status Flow

```
PostgresConnectionManager.GetStatus()
    │
    ├── For each connection in connections:#    │   └── Call conn.GetStatus()
    └── Return map of statuses
```

## Configuration

### Configuration Structs

```go
type PostgresConfig struct {
    Enabled  bool   `mapstructure:"enabled"`
    Host     string `mapstructure:"host"`
    Port     int    `mapstructure:"port"`
    User     string `mapstructure:"user"`
    Password string `mapstructure:"password"`
    DBName   string `mapstructure:"dbname"`
    SSLMode  string `mapstructure:"sslmode"`
}

type PostgresMultiConfig struct {
    Enabled     bool             `mapstructure:"enabled"`
    Connections []PostgresConfig `mapstructure:"connections"`
}
```

### Viper Configuration Keys

For single connection:
- `postgres.enabled` (bool)
- `postgres.host` (string)
- `postgres.port` (int, default 5432)
- `postgres.user` (string)
- `postgres.password` (string)
- `postgres.dbname` (string)
- `postgres.sslmode` (string, e.g., "disable", "require")

For multiple connections:
- `postgres-multi.enabled` (bool)
- `postgres-multi.connections` (list of maps with keys: `name`, `enabled`, `host`, `port`, `user`, `password`, `dbname`, `sslmode`)

### Environment Variables

Configuration is read through Viper, which can load from files or environment variables.

Example programmatic configuration:

```go
cfg := config.PostgresConfig{
    Enabled:  true,
    Host:     "localhost",
    Port:     5432,
    User:     "postgres",
    Password: "password",
    DBName:   "mydb",
    SSLMode:  "disable",
}
```

## Usage Examples

### Using Raw SQL

```go
ctx := context.Background()

// Query returning multiple rows
rows, err := manager.Query(ctx, "SELECT id, name FROM users")
if err != nil {
    panic(err)
}
defer rows.Close()

for rows.Next() {
    var id int
    var name string
    if err := rows.Scan(&id, &name); err == nil {
        fmt.Printf("User: %d - %s\n", id, name)
    }
}
```

### Using QueryRow

```go
ctx := context.Background()

// Query single row
row := manager.QueryRow(ctx, "SELECT name FROM users WHERE id = $1", 1)
var name string
if err := row.Scan(&name); err != nil {
    panic(err)
}
fmt.Printf("User name: %s\n", name)
```

### Using Exec

```go
ctx := context.Background()

// Execute insert/update/delete
result, err := manager.Exec(ctx, "INSERT INTO users (name) VALUES ($1)", "John")
if err != nil {
    panic(err)
}
rowsAffected, _ := result.RowsAffected()
fmt.Printf("Affected %d rows\n", rowsAffected)
```

### Using Insert/Update/Delete

```go
ctx := context.Background()

// Insert
count, err := manager.Insert(ctx, "INSERT INTO users (name) VALUES ($1)", "Alice")
if err != nil {
    panic(err)
}
fmt.Printf("Inserted %d rows\n", count)

// Update
count, err = manager.Update(ctx, "UPDATE users SET age = $1 WHERE name = $2", 30, "Alice")
if err != nil {
    panic(err)
}
fmt.Printf("Updated %d rows\n", count)

// Delete
count, err = manager.Delete(ctx, "DELETE FROM users WHERE name = $1", "Alice")
if err != nil {
    panic(err)
}
fmt.Printf("Deleted %d rows\n", count)
```

### Using GORM

```go
// Use the GORM DB for ORM operations
db := manager.ORM

// Auto-migrate schema
db.AutoMigrate(&User{})

// Create a record
db.Create(&User{Name: "John"})

// Query
var user User
db.First(&user, "name = ?", "John")
fmt.Printf("User: %v\n", user)
```

### Raw Query Execution

```go
ctx := context.Background()

results, err := manager.ExecuteRawQuery(ctx, "SELECT * FROM users WHERE active = true")
if err != nil {
    panic(err)
}

fmt.Printf("Found %d rows\n", len(results))
for _, row := range results {
    fmt.Printf("Row: %v\n", row)
}
```

### Monitoring Queries

```go
ctx := context.Background()

// Get running queries
queries, err := manager.GetRunningQueries(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Running queries: %d\n", len(queries))
for _, q := range queries {
    fmt.Printf("  PID %d: %s (duration: %s)\n", q.Pid, q.Query, q.Duration)
}

// Get session count
count, err := manager.GetSessionCount(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Sessions: %d\n", count)

// Get database info
info, err := manager.GetDBInfo(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("DB Info: %v\n", info)
```

### Using Multiple Connections

```go
// Assuming you have a PostgresConnectionManager
connManager, err := postgres.NewPostgresConnectionManager(cfg, log)
if err != nil {
    panic(err)
}

// Get a specific connection
conn, exists := connManager.GetConnection("primary")
if !exists {
    panic("connection not found")
}

// Use the connection
rows, err := conn.Query(ctx, "SELECT 1")
```

### Async Operations

```go
ctx := context.Background()

// Async query
asyncResult := manager.QueryAsync(ctx, "SELECT * FROM users")
select {
case rows := <-asyncResult.Ch:#    defer rows.Close()
    fmt.Println("Query completed")
case err := <-asyncResult.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async exec
asyncResult2 := manager.ExecAsync(ctx, "INSERT INTO users (name) VALUES ($1)", "Alice")
select {
case result := <-asyncResult2.Ch:#    fmt.Println("Exec completed")
case err := <-asyncResult2.ErrCh:#    fmt.Printf("Error: %v\n", err)
}
```

### GORM Async Operations

```go
ctx := context.Background()

// Async GORM create
err := manager.GORMCreateAsync(ctx, &User{Name: "Bob"}).ErrCh
if err != nil {
    fmt.Printf("Error: %v\n", err)
}

// Async GORM find
var users []User
err = manager.GORMFindAsync(ctx, &users).ErrCh
if err != nil {
    fmt.Printf("Error: %v\n", err)
}
```

### Batch Operations

```go
ctx := context.Background()

queries := []string{
    "INSERT INTO users (name) VALUES ('User1')",
    "INSERT INTO users (name) VALUES ('User2')",
    "INSERT INTO users (name) VALUES ('User3')",
}
args := [][]interface{}{nil, nil, nil}

batchResult := manager.ExecuteBatchAsync(ctx, queries, args)
// Wait for all operations
for i, opResult := range batchResult.Results {
    if opResult.Error != nil {
        fmt.Printf("Query %d failed: %v\n", i, opResult.Error)
    } else {
        fmt.Printf("Query %d succeeded\n", i)
    }
}
```

### Getting Status

```go
// For single connection
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])

if status["connected"] == true {
    fmt.Printf("Open connections: %v\n", status["open_connections"])
    fmt.Printf("In use: %v\n", status["in_use"])
    fmt.Printf("Idle: %v\n", status["idle"])
}

// For connection manager
connStatus := connManager.GetStatus()
for name, stat := range connStatus {
    fmt.Printf("Connection %s: %v\n", name, stat)
}
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/jackc/pgx/v5/stdlib` | PostgreSQL driver for Go |
| `gorm.io/driver/postgres` | GORM PostgreSQL driver |
| `gorm.io/gorm` | ORM library |
| `github.com/spf13/viper` | Configuration management |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `context`, `database/sql`, `fmt`, `sync`, `time` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: Returned from `sql.Open()` or `db.Ping()`
- **Query failures**: Returned from `DB.QueryContext()`, `DB.QueryRowContext()`, `DB.ExecContext()`
- **Nil checks**: `GetStatus()` handles nil receiver gracefully

Example error handling:

```go
rows, err := manager.Query(ctx, query)
if err != nil {
    if strings.Contains(err.Error(), "connection refused") {
        return fmt.Errorf("postgres not reachable: %w", err)
    }
    if strings.Contains(err.Error(), "password authentication failed") {
        return fmt.Errorf("authentication failed: %w", err)
    }
    if strings.Contains(err.Error(), "database") && strings.Contains(err.Error(), "does not exist") {
        return fmt.Errorf("database not found: %w", err)
    }
    return fmt.Errorf("query failed: %w", err)
}
```

## Common Pitfalls

### 1. PostgreSQL Not Reachable

**Problem**: `connection refused` errors

**Solution**: 
- Verify PostgreSQL is running
- Check host/port in configuration
- Default port is 5432
- Check firewall settings

### 2. Authentication Failed

**Problem**: `password authentication failed` errors

**Solution**:
- Verify username and password
- Check if user has access to the database
- Ensure user has proper grants

### 3. Database Not Found

**Problem**: `database does not exist` errors

**Solution**:
- Check database name in configuration
- Create database if it doesn't exist
- Verify case sensitivity (PostgreSQL database names are case-sensitive)

### 4. Connection Pool Exhaustion

**Problem**: `too many clients` errors

**Solution**:
- Adjust connection pool settings via GORM/pgx
- Ensure connections are properly closed (use `defer rows.Close()`)

### 5. GORM Issues

**Problem**: GORM operations failing

**Solution**:
- The GORM instance is initialized with the same SQL connection
- Use `manager.ORM` for GORM operations
- Check GORM documentation for proper usage

### 6. SSL Mode Issues

**Problem**: SSL-related connection errors

**Solution**:
- Set `sslmode` appropriately:#    - `disable`: No SSL (development)
  - `require`: SSL required
  - `verify-full`: SSL with full verification
- For development, `disable` is typically used

### 7. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewPostgresDB()` succeeded
- Verify `postgres.enabled` is true

## Advanced Usage

### Using Transactions with Raw SQL

```go
ctx := context.Background()

// Begin transaction
tx, err := manager.DB.BeginTx(ctx, nil)
if err != nil {
    panic(err)
}
defer tx.Rollback()

// Execute queries in transaction
_, err = tx.Exec("INSERT INTO users (name) VALUES ($1)", "Bob")
if err != nil {
    panic(err)
}

// Commit transaction
if err := tx.Commit(); err != nil {
    panic(err)
}
fmt.Println("Transaction committed")
```

### Using GORM Transactions

```go
db := manager.ORM

// Transaction
db.Transaction(func(tx *gorm.DB) error {
    // Create user
    if err := tx.Create(&User{Name: "Alice"}).Error; err != nil {
        return err
    }
    // Update something else
    if err := tx.Model(&Account{}).Where("user_id = ?", 1).Update("balance", gorm.Expr("balance - ?", 100)).Error; err != nil {
        return err
    }
    return nil
})
```

### Health Check

```go
func healthCheck(manager *postgres.PostgresManager) error {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    status := manager.GetStatus()
    if status["connected"] == false {
        return fmt.Errorf("not connected to PostgreSQL")
    }
    
    // Additional check: simple query
    row := manager.QueryRow(ctx, "SELECT 1")
    var result int
    if err := row.Scan(&result); err != nil {
        return err
    }
    fmt.Println("PostgreSQL connection healthy")
    return nil
}
```

### Complex Raw Query with Results

```go
func complexQuery(manager *postgres.PostgresManager) error {
    ctx := context.Background()
    
    // Query with JOIN
    query := `
        SELECT u.name, p.title 
        FROM users u 
        JOIN posts p ON u.id = p.user_id 
        WHERE u.active = true 
        ORDER BY p.created_at DESC
    `
    
    results, err := manager.ExecuteRawQuery(ctx, query)
    if err != nil {
        return err
    }
    
    fmt.Printf("Found %d results\n", len(results))
    for _, row := range results {
        fmt.Printf("User: %v, Post: %v\n", row["name"], row["title"])
    }
    return nil
}
```

## Internal Algorithms

### Connection Test

```
Initialization:
    │
    ├── sql.Open("pgx", dsn)
    └── sqlDB.Ping()
```

### Raw Query Execution

```
ExecuteRawQuery():
    │
    ├── DB.QueryContext(ctx, query)
    ├── Get columns
    ├── For each row:#    │   ├── Scan into []interface{}
    │   ├── Build rowMap
    │   └── Convert []byte to string
    └── Return []map[string]interface{}
```

### Status Collection

```
GetStatus():
    │
    ├── DB.Ping()
    ├── DB.Stats() → sql.DBStats
    └── Return map with connected, open_connections, in_use, idle, wait_count, wait_duration_ms
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.