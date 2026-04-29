# MySQL Manager

## Overview

The `MySQLManager` is a Go library for managing MySQL database connections. It provides both raw SQL operations via `database/sql` and ORM capabilities via GORM. It uses the `go-sql-driver/mysql` driver with configurable connection pool settings and a worker pool for async operations.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Dual Interface**: Raw SQL (`*sql.DB`) and GORM (`*gorm.DB`)
- **SQL Operations**: Query (returning rows), QueryRow (single row), Exec (no rows)
- **Connection Pool**: Configurable max open connections, max idle connections, connection max lifetime
- **GORM Integration**: Full GORM support for ORM operations
- **Async Operations**: Async versions of Query and Exec via `ExecuteAsync`
- **Worker Pool**: Async job execution support (15 workers)
- **Status Monitoring**: Get connection status and pool statistics
- **Ping Check**: Automatic connection test during initialization

## Quick Start

```go
package main

import (
    "context"
    "database/sql"
    "fmt"
    "stackyrd/pkg/infrastructure"
    "stackyrd/pkg/logger"
    _ "github.com/go-sql-driver/mysql"
)

func main() {
    // Initialize logger
    log := logger.NewLogger()
    
    // Create MySQL manager (configuration via viper)
    manager, err := mysql.NewMySQLManager(log)
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
| `MySQLManager` | Main manager with raw SQL DB, GORM DB, worker pool |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   MySQLManager                              │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  sql.DB     │  → Raw SQL operations                  │
│  │  (pool)     │  → Query, QueryRow, Exec             │
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
│  Host: MySQL host (from config)                            │
│  Port: MySQL port (from config)                            │
│  Database: database name (from config)                      │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewMySQLManager(logger)
    │
    ├── Check viper config: "mysql.enabled"
    ├── Get host: "mysql.host"
    ├── Get port: "mysql.port"
    ├── Get user: "mysql.user"
    ├── Get password: "mysql.password"
    ├── Get dbname: "mysql.dbname"
    ├── Get maxOpenConns: "mysql.max_open_conns"
    ├── Get maxIdleConns: "mysql.max_idle_conns"
    ├── Get connMaxLifetime: "mysql.conn_max_lifetime"
    │
    ├── Build DSN: "user:password@tcp(host:port)/dbname?charset=utf8mb4&parseTime=True&loc=Local"
    │
    ├── Open raw SQL connection: sql.Open("mysql", dsn)
    ├── Ping to verify connection
    ├── Configure connection pool:
    │   ├── SetMaxOpenConns(maxOpenConns)
    │   ├── SetMaxIdleConns(maxIdleConns)
    │   └── SetConnMaxLifetime(connMaxLifetime)
    │
    ├── Initialize GORM with existing SQL connection:
    │   └── gorm.Open(mysql.New(mysql.Config{Conn: sqlDB}), &gorm.Config{})
    │
    ├── Create WorkerPool(15)
    ├── Start worker pool
    └── Return MySQLManager
```

### 2. Query Flow

```
Query(ctx, query, args...)
    │
    └── DB.QueryContext(ctx, query, args...) → *sql.Rows
```

### 3. QueryRow Flow

```
QueryRow(ctx, query, args...)
    │
    └── DB.QueryRowContext(ctx, query, args...) → *sql.Row
```

### 4. Exec Flow

```
Exec(ctx, query, args...)
    │
    └── DB.ExecContext(ctx, query, args...) → sql.Result
```

### 5. Get Status Flow

```
GetStatus()
    │
    ├── If nil or DB == nil: return connected=false
    ├── Ping: DB.Ping()
    ├── Get DB stats: DB.Stats()
    └── Return connected, open_connections, in_use, idle, wait_count, wait_duration_ms
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `mysql.enabled` | bool | false | Enable/disable MySQL manager |
| `mysql.host` | string | "" | MySQL host |
| `mysql.port` | int | 3306 | MySQL port |
| `mysql.user` | string | "" | Username |
| `mysql.password` | string | "" | Password |
| `mysql.dbname` | string | "" | Database name |
| `mysql.max_open_conns` | int | 0 | Max open connections (0 = unlimited) |
| `mysql.max_idle_conns` | int | 0 | Max idle connections |
| `mysql.conn_max_lifetime` | duration | 0 | Max connection lifetime (e.g., "1h") |

### Environment Variables

Configuration is read through Viper, which can load from files or environment variables.

Example programmatic configuration:

```go
viper.Set("mysql.enabled", true)
viper.Set("mysql.host", "localhost")
viper.Set("mysql.port", 3306)
viper.Set("mysql.user", "root")
viper.Set("mysql.password", "secret")
viper.Set("mysql.dbname", "mydb")
viper.Set("mysql.max_open_conns", 10)
viper.Set("mysql.max_idle_conns", 5)
viper.Set("mysql.conn_max_lifetime", "1h")
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
row := manager.QueryRow(ctx, "SELECT name FROM users WHERE id = ?", 1)
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
result, err := manager.Exec(ctx, "INSERT INTO users (name) VALUES (?)", "John")
if err != nil {
    panic(err)
}
lastID, _ := result.LastInsertId()
fmt.Printf("Inserted ID: %d\n", lastID)
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

### Async Operations

```go
ctx := context.Background()

// Async query
asyncResult := manager.QueryAsync(ctx, "SELECT * FROM users")
select {
case rows := <-asyncResult.Ch:
    defer rows.Close()
    fmt.Println("Query completed")
case err := <-asyncResult.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async exec
asyncResult2 := manager.ExecAsync(ctx, "INSERT INTO users (name) VALUES (?)", "Alice")
select {
case result := <-asyncResult2.Ch:
    fmt.Println("Insert completed")
case err := <-asyncResult2.ErrCh:
    fmt.Printf("Error: %v\n", err)
}
```

### Getting Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])

if status["connected"] == true {
    fmt.Printf("Open connections: %v\n", status["open_connections"])
    fmt.Printf("In use: %v\n", status["in_use"])
    fmt.Printf("Idle: %v\n", status["idle"])
}
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/go-sql-driver/mysql` | MySQL driver for Go |
| `gorm.io/driver/mysql` | GORM MySQL driver |
| `gorm.io/gorm` | ORM library |
| `github.com/spf13/viper` | Configuration management |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `context`, `database/sql`, `fmt`, `time` |

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
        return fmt.Errorf("mysql not reachable: %w", err)
    }
    if strings.Contains(err.Error(), "access denied") {
        return fmt.Errorf("authentication failed: %w", err)
    }
    if strings.Contains(err.Error(), "unknown database") {
        return fmt.Errorf("database not found: %w", err)
    }
    return fmt.Errorf("query failed: %w", err)
}
```

## Common Pitfalls

### 1. MySQL Not Reachable

**Problem**: `connection refused` errors

**Solution**: 
- Verify MySQL is running
- Check host/port in configuration
- Default port is 3306
- Check firewall settings

### 2. Authentication Failed

**Problem**: `access denied` errors

**Solution**:
- Verify username and password
- Check if user has access to the database
- Ensure user has proper grants

### 3. Database Not Found

**Problem**: `unknown database` errors

**Solution**:
- Check database name in configuration
- Create database if it doesn't exist
- Verify case sensitivity (MySQL database names are case-sensitive on some systems)

### 4. Connection Pool Exhaustion

**Problem**: `too many connections` errors

**Solution**:
- Adjust `mysql.max_open_conns`
- Adjust `mysql.max_idle_conns`
- Ensure connections are properly closed (use `defer rows.Close()`)

### 5. GORM Issues

**Problem**: GORM operations failing

**Solution**:
- The GORM instance is initialized with the same SQL connection
- Use `manager.ORM` for GORM operations
- Check GORM documentation for proper usage

### 6. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewMySQLManager()` succeeded
- Verify `mysql.enabled` is true

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
_, err = tx.Exec("INSERT INTO users (name) VALUES (?)", "Bob")
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
func healthCheck(manager *mysql.MySQLManager) error {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    status := manager.GetStatus()
    if status["connected"] == false {
        return fmt.Errorf("not connected to MySQL")
    }
    
    // Additional check: simple query
    row := manager.QueryRow(ctx, "SELECT 1")
    var result int
    if err := row.Scan(&result); err != nil {
        return err
    }
    fmt.Println("MySQL connection healthy")
    return nil
}
```

### Batch Operations

```go
func batchInsert(manager *mysql.MySQLManager) error {
    ctx := context.Background()
    
    // Prepare statement
    stmt, err := manager.DB.PrepareContext(ctx, "INSERT INTO users (name) VALUES (?)")
    if err != nil {
        return err
    }
    defer stmt.Close()
    
    // Batch insert
    users := []string{"User1", "User2", "User3"}
    for _, u := range users {
        if _, err := stmt.ExecContext(ctx, u); err != nil {
            return err
        }
    }
    return nil
}
```

## Internal Algorithms

### Connection Test

```
Initialization:
    │
    ├── sql.Open("mysql", dsn)
    └── sqlDB.Ping()
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