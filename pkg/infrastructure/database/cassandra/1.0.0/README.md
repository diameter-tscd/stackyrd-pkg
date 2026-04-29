# Cassandra Manager

## Overview

The `CassandraManager` is a Go library for managing Apache Cassandra database interactions. It provides query execution with async support via a worker pool, using the gocql driver.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Query Execution**: Execute CQL queries that return multiple rows
- **Single Row Query**: Execute queries that return at most one row
- **Exec Operations**: Execute queries without returning rows (INSERT, UPDATE, DELETE)
- **Connection Management**: Automatic session creation with configurable hosts, port, keyspace
- **Authentication**: Support for username/password authentication
- **Consistency Levels**: Configurable consistency levels for queries
- **Async Operations**: Async query, query row, and exec via worker pool
- **Worker Pool**: Async job execution support (12 workers)
- **Status Monitoring**: Get connection status and cluster information

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
    // Initialize logger
    log := logger.NewLogger()
    
    // Create Cassandra manager (configuration via viper)
    manager, err := cassandra.NewCassandraManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Execute a query
    rows, err := manager.Query(ctx, "SELECT * FROM users WHERE id = ?", "user123")
    if err != nil {
        panic(err)
    }
    fmt.Printf("Found %d rows\n", len(rows))
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `CassandraManager` | Main manager with gocql session, cluster config, worker pool |
| `CassandraRow` | Map representing a query result row |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   CassandraManager                         │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  gocql     │  → Session, ClusterConfig              │
│  │  Session    │  → Query, Exec, consistency           │
│  └────────────────┘                                      │
│                                                         │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ AsyncResult │                   │
│  │  (12 workers)│     │   Channel   │                   │
│  └─────────────┘      └──────────────┘                   │
│         ▲                      │                            │
│         │                      ▼                            │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  SubmitJob  │      │ ExecuteAsync │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                         │
│  Hosts: Cassandra node addresses                        │
│  Keyspace: Default keyspace for queries                  │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewCassandraManager(logger)
    │
    ├── Check viper config: "cassandra.enabled"
    ├── Get hosts: "cassandra.hosts" (slice)
    ├── Get port: "cassandra.port"
    ├── Get keyspace: "cassandra.keyspace"
    ├── Get username: "cassandra.username"
    ├── Get password: "cassandra.password"
    ├── Get consistency: "cassandra.consistency"
    │
    ├── Create gocql.ClusterConfig with hosts
    ├── Set Port, Keyspace, Consistency, Timeout, ConnectTimeout
    ├── If username/password: Set PasswordAuthenticator
    │
    ├── Create session: cluster.CreateSession()
    ├── Test connection: testConnection()
    │   └── Query: "SELECT count(*) FROM system.local"
    │
    ├── Create WorkerPool(12)
    ├── Start worker pool
    └── Return CassandraManager
```

### 2. Query Flow

```
Query(ctx, query, args...)
    │
    ├── session.Query(query, args...).WithContext(ctx).Iter()
    ├── Iterate with iter.MapScan(row)
    ├── Collect rows into []CassandraRow
    ├── iter.Close()
    └── Return rows, error
```

### 3. QueryRow Flow

```
QueryRow(ctx, query, args...)
    │
    ├── Create CassandraRow map
    ├── session.Query(query, args...).WithContext(ctx).MapScan(row)
    └── Return row, error
```

### 4. Exec Flow

```
Exec(ctx, query, args...)
    │
    └── session.Query(query, args...).WithContext(ctx).Exec()
```

### 5. Get Status Flow

```
GetStatus()
    │
    ├── If nil or session nil: return connected=false
    ├── Test connection: testConnection()
    ├── Return connected, hosts, keyspace, consistency, pool_active
    └── Note: Tests actual connection with system query
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `cassandra.enabled` | bool | false | Enable/disable Cassandra manager |
| `cassandra.hosts` | []string | [] | Cassandra node addresses |
| `cassandra.port` | int | 9042 | Cassandra port |
| `cassandra.keyspace` | string | "" | Default keyspace |
| `cassandra.username` | string | "" | Username for authentication |
| `cassandra.password` | string | "" | Password for authentication |
| `cassandra.consistency` | string | "QUORUM" | Consistency level |

### Environment Variables

The Cassandra manager does not directly use environment variables. Configuration is read through Viper, which can load from:
- Configuration files (JSON, YAML, TOML)
- Environment variables (if Viper is configured for it)

You can set Cassandra parameters via:
```go
viper.Set("cassandra.enabled", true)
viper.Set("cassandra.hosts", []string{"127.0.0.1"})
viper.Set("cassandra.port", 9042)
viper.Set("cassandra.keyspace", "mykeyspace")
viper.Set("cassandra.username", "cassandra")
viper.Set("cassandra.password", "cassandra")
viper.Set("cassandra.consistency", "QUORUM")
```

## Usage Examples

### Executing a Query

```go
ctx := context.Background()

// Query multiple rows
rows, err := manager.Query(ctx, "SELECT * FROM users")
if err != nil {
    panic(err)
}
fmt.Printf("Found %d rows\n", len(rows))
for _, row := range rows {
    fmt.Printf("Row: %v\n", row)
}
```

### Querying a Single Row

```go
ctx := context.Background()

// Query single row
row, err := manager.QueryRow(ctx, "SELECT * FROM users WHERE id = ?", "user123")
if err != nil {
    panic(err)
}
fmt.Printf("User: %v\n", row)
```

### Executing an Update

```go
ctx := context.Background()

// Insert data
err := manager.Exec(ctx, "INSERT INTO users (id, name) VALUES (?, ?)", "user123", "John")
if err != nil {
    panic(err)
}
fmt.Println("User inserted")
```

### Async Query

```go
ctx := context.Background()

// Async query
result := manager.QueryAsync(ctx, "SELECT * FROM users")
select {
case rows := <-result.Ch:
    fmt.Printf("Found %d rows\n", len(rows))
case err := <-result.ErrCh:
    fmt.Printf("Error: %v\n", err)
}
```

### Async QueryRow

```go
ctx := context.Background()

// Async query single row
result := manager.QueryRowAsync(ctx, "SELECT * FROM users WHERE id = ?", "user123")
select {
case row := <-result.Ch:
    fmt.Printf("User: %v\n", row)
case err := <-result.ErrCh:
    fmt.Printf("Error: %v\n", err)
}
```

### Async Exec

```go
ctx := context.Background()

// Async exec
result := manager.ExecAsync(ctx, "INSERT INTO users (id, name) VALUES (?, ?)", "user123", "John")
select {
case <-result.Ch:
    fmt.Println("User inserted")
case err := <-result.ErrCh:
    fmt.Printf("Error: %v\n", err)
}
```

### Get Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Hosts: %v\n", status["hosts"])
fmt.Printf("Keyspace: %s\n", status["keyspace"])
fmt.Printf("Consistency: %s\n", status["consistency"])

if status["connected"] == false {
    if errMsg, ok := status["error"].(string); ok {
        fmt.Printf("Error: %s\n", errMsg)
    }
}
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/gocql/gocql` | Cassandra driver for Go |
| `github.com/spf13/viper` | Configuration management |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `context`, `fmt`, `time` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: Returned from `cluster.CreateSession()`
- **Query failures**: Returned from `Query()`, `QueryRow()`, `Exec()`
- **Test connection**: Uses `system.local` query
- **Nil checks**: Public methods handle nil receiver gracefully in `GetStatus()`

Example error handling:

```go
rows, err := manager.Query(ctx, query, args...)
if err != nil {
    if strings.Contains(err.Error(), "connection refused") {
        return fmt.Errorf("cassandra not reachable: %w", err)
    }
    if strings.Contains(err.Error(), "unauthorized") {
        return fmt.Errorf("authentication failed: %w", err)
    }
    if strings.Contains(err.Error(), "keyspace") {
        return fmt.Errorf("keyspace not found: %w", err)
    }
    return fmt.Errorf("query failed: %w", err)
}
```

## Common Pitfalls

### 1. Cassandra Not Reachable

**Problem**: `connection refused` or `no route to host` errors

**Solution**: 
- Verify Cassandra is running
- Check host addresses in `cassandra.hosts`
- Verify port (default: 9042)
- Check firewall settings

### 2. Authentication Failed

**Problem**: `unauthorized` or `invalid credentials` errors

**Solution**:
- Verify username and password
- Check if authentication is required
- Ensure user has access to the keyspace

### 3. Keyspace Not Found

**Problem**: `keyspace not found` errors

**Solution**:
- Verify keyspace name in `cassandra.keyspace`
- Create keyspace if it doesn't exist
- Check case sensitivity (CQL is case-sensitive for identifiers)

### 4. Consistency Level Issues

**Problem**: `consistency level` errors

**Solution**:
- Use valid consistency levels: `ANY`, `ONE`, `TWO`, `THREE`, `QUORUM`, `ALL`, `LOCAL_QUORUM`, `LOCAL_ONE`, `LOCAL_SERIAL`, `SERIAL`
- Check `cassandra.consistency` configuration

### 5. Timeout Issues

**Problem**: Operations timing out

**Solution**:
- The default timeout is 10s for queries, 5s for connect
- Adjust based on your network and cluster performance
- Note: Timeout is set in ClusterConfig, not configurable via viper in current implementation

### 6. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewCassandraManager()` succeeded
- Verify `cassandra.enabled` is true

## Advanced Usage

### Batch Query Processing

```go
func batchQuery(manager *cassandra.CassandraManager, queries []string) error {
    ctx := context.Background()
    
    for i, query := range queries {
        rows, err := manager.Query(ctx, query)
        if err != nil {
            return fmt.Errorf("query %d failed: %w", i, err)
        }
        fmt.Printf("Query %d: %d rows\n", i, len(rows))
    }
    return nil
}
```

### Using Query Results

```go
func processUsers(manager *cassandra.CassandraManager) error {
    ctx := context.Background()
    
    rows, err := manager.Query(ctx, "SELECT id, name, email FROM users")
    if err != nil {
        return err
    }
    
    for _, row := range rows {
        id, _ := row["id"].(string)
        name, _ := row["name"].(string)
        email, _ := row["email"].(string)
        fmt.Printf("User: %s (%s) - %s\n", id, name, email)
    }
    return nil
}
```

### Prepared Statements Pattern

```go
func insertUser(manager *cassandra.CassandraManager, id, name, email string) error {
    ctx := context.Background()
    
    // Note: gocql automatically prepares statements
    err := manager.Exec(ctx, 
        "INSERT INTO users (id, name, email) VALUES (?, ?, ?)",
        id, name, email)
    return err
}
```

### Health Check Query

```go
func healthCheck(manager *cassandra.CassandraManager) error {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    _, err := manager.QueryRow(ctx, "SELECT now() FROM system.local")
    return err
}
```

## Internal Algorithms

### Query Execution

```
Query():
    │
    ├── session.Query(query, args...).WithContext(ctx).Iter()
    ├── Loop: iter.MapScan(row)
    │   └── Append to rows
    ├── iter.Close()
    └── Return rows
```

### Connection Test

```
testConnection():
    │
    └── session.Query("SELECT count(*) FROM system.local").Scan(&count)
```

### Async Operation Pattern

```
QueryAsync(ctx, query, args...):
    │
    └── ExecuteAsync(ctx, func):
        └── return Query(ctx, query, args...)
            └── Returns ([]CassandraRow, error)
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.