# Redis Manager

## Overview

The `RedisManager` is a comprehensive Go library for managing Redis operations using the official Go Redis client (`go-redis/v9`). It provides a complete interface for Redis caching with TTL support, key scanning, batch operations, and async execution via worker pools.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Full Redis Support**: Complete Redis operations via `go-redis/v9` client
- **TTL Support**: Set expiration times on cached items
- **Key Scanning**: Pattern-based key scanning (safe, limited to 100)
- **Batch Operations**: Set, get, and delete multiple keys asynchronously
- **Worker Pool**: Async job execution with configurable worker count (default 10)
- **Pool Statistics**: Monitor Redis connection pool health
- **Context Support**: Full `context.Context` support for timeouts
- **Server Info**: Retrieve detailed Redis server information

## Quick Start

```go
package main

import (
    "context"
    "fmt"
    "stackyrd/config"
    "stackyrd/pkg/infrastructure"
    "stackyrd/pkg/logger"
)

func main() {
    // Initialize logger
    log := logger.NewLogger()
    
    // Configure Redis
    cfg := config.RedisConfig{
        Enabled:  true,
        Address:  "localhost:6379",
        Password: "",
        DB:       0,
    }
    
    // Create Redis manager
    manager, err := redis.NewRedisClient(cfg)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Set a value with TTL
    err = manager.Set(ctx, "mykey", "myvalue", 3600*time.Second)
    if err != nil {
        panic(err)
    }
    
    // Get a value
    value, err := manager.Get(ctx, "mykey")
    if err != nil {
        panic(err)
    }
    fmt.Printf("Value: %s\n", value)
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `RedisManager` | Main manager wrapping `go-redis/v9` client with worker pool |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                      RedisManager                             │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────────┐                                  │
│  │  go-redis/v9      │  → Native connection pooling         │
│  │  Client           │  → PoolStats tracking                │
│  └────────────────────┘                                  │
│                                                         │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ AsyncResult │                   │
│  │  (10 workers)│     │   Channel   │                   │
│  └─────────────┘      └──────────────┘                   │
│         ▲                      │                            │
│         │                      ▼                            │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  SubmitJob  │      │ ExecuteAsync │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                         │
│  ┌─────────────────────────────────────┐                  │
│  │       BatchAsync Operations          │                  │
│  │  SetBatchAsync, GetBatchAsync,   │                  │
│  │  DeleteBatchAsync                 │                  │
│  └─────────────────────────────────────┘                  │
└─────────────────────────────────────────────────────────────┘
```

The manager uses `go-redis/v9` which has built-in connection pooling. Additionally, a worker pool with 10 goroutines handles async operations. Batch operations use `BatchAsyncResult` for concurrent execution.

## How It Works

### 1. Initialization Flow

```
NewRedisClient(cfg)
    │
    ├── Check cfg.Enabled
    ├── Create redis.NewClient(&redis.Options{
    │     Addr: cfg.Address,
    │     Password: cfg.Password,
    │     DB: cfg.DB,
    │   })
    ├── Test connection: client.Ping(context.Background())
    ├── Create WorkerPool(10)
    └── Return RedisManager
```

### 2. Set Operation Flow

```
Set(ctx, key, value, ttl)
    │
    └── r.Client.Set(ctx, key, value, ttl).Err()
            │
            └── Returns error or nil
```

### 3. Get Operation Flow

```
Get(ctx, key)
    │
    └── r.Client.Get(ctx, key).Result()
            │
            ├── Returns (string, error)
            └── go-redis handles redis.Nil for key not found
```

### 4. Batch Operation Pattern

```
SetBatchAsync(ctx, kvPairs, ttl)
    │
    ├── Create []AsyncOperation[struct{}]
    ├── For each key-value pair:
    │   └── Append func(ctx) { return struct{}{}, r.Set(ctx, key, value, ttl) }
    └── Return ExecuteBatchAsync(ctx, operations)
            │
            └── Returns *BatchAsyncResult[struct{}]
                    │
                    └── Results via channel: <-result.Ch
```

### 5. Async Operation Pattern

```
GetAsync(ctx, key)
    │
    └── ExecuteAsync(ctx, func(ctx) {
            return r.Get(ctx, key)
        })
            │
            └── Returns *AsyncResult[string]
                    │
                    └── Result via channel: <-result.Ch
```

## Configuration

### RedisConfig Struct

The Redis manager uses `config.RedisConfig` for configuration:

| Field | Type | Description |
|-------|------|-------------|
| `Enabled` | bool | Enable/disable Redis manager |
| `Address` | string | Redis server address (e.g., "localhost:6379") |
| `Password` | string | Redis password (empty for no auth) |
| `DB` | int | Redis database number (0-15) |

### Example Configuration

```go
cfg := config.RedisConfig{
    Enabled:  true,
    Address:  "redis.example.com:6379",
    Password: "secret",
    DB:       0,
}
```

## Usage Examples

### Basic Operations

```go
ctx := context.Background()

// Set a value with TTL (1 hour)
err := manager.Set(ctx, "user:123", "John Doe", 3600*time.Second)

// Get a value
value, err := manager.Get(ctx, "user:123")
if err == nil {
    fmt.Printf("Value: %s\n", value)
}

// Get value with helper (assuming string)
strValue, err := manager.GetValue(ctx, "user:123")
if err == nil {
    fmt.Printf("String value: %s\n", strValue)
}

// Delete a key
err = manager.Delete(ctx, "user:123")

// Replace only if exists (XX option)
err = manager.Replace(ctx, "user:123", "Jane Doe", 3600*time.Second)
```

### Key Scanning

```go
// Scan for keys matching a pattern (limited to 100)
keys, err := manager.ScanKeys(ctx, "user:*")
if err != nil {
    panic(err)
}

fmt.Printf("Found %d keys:\n", len(keys))
for _, key := range keys {
    fmt.Printf("  - %s\n", key)
}
```

### Get Server Information

```go
// Get Redis INFO command output
info, err := manager.GetInfo(ctx)
if err != nil {
    panic(err)
}

fmt.Println("Redis Server Info:")
fmt.Println(info)
```

### Async Operations

```go
ctx := context.Background()

// Async set
result := manager.SetAsync(ctx, "async:key", "async-value", 3600*time.Second)
select {
case <-result.Ctx.Done():
    fmt.Println("Operation cancelled")
case <-result.Ch:
    fmt.Println("Async set complete")
}

// Async get
result := manager.GetAsync(ctx, "mykey")
select {
case value := <-result.Ch:
    fmt.Printf("Got value: %s\n", value)
}

// Async delete
result := manager.DeleteAsync(ctx, "oldkey")
select {
case <-result.Ch:
    fmt.Println("Async delete complete")
}

// Async replace
result := manager.ReplaceAsync(ctx, "mykey", "new-value", 3600*time.Second)
select {
case <-result.Ch:
    fmt.Println("Async replace complete")
}

// Async scan keys
result := manager.ScanKeysAsync(ctx, "user:*")
select {
case keys := <-result.Ch:
    fmt.Printf("Found %d keys\n", len(keys))
}

// Async get server info
result := manager.GetInfoAsync(ctx)
select {
case info := <-result.Ch:
    fmt.Println("Server info:", info)
}
```

### Batch Operations

```go
ctx := context.Background()

// Batch set multiple keys
kvPairs := map[string]interface{}{
    "key1": "value1",
    "key2": "value2",
    "key3": "value3",
}
result := manager.SetBatchAsync(ctx, kvPairs, 3600*time.Second)
select {
case <-result.Ch:
    fmt.Println("Batch set complete")
case err := <-result.ErrCh:
    fmt.Printf("Batch error: %v\n", err)
}

// Batch get multiple keys
keys := []string{"key1", "key2", "key3"}
result := manager.GetBatchAsync(ctx, keys)
select {
case results := <-result.Ch:
    for i, val := range results {
        fmt.Printf("Key %s: %s\n", keys[i], val)
    }
}

// Batch delete multiple keys
result := manager.DeleteBatchAsync(ctx, keys)
select {
case <-result.Ch:
    fmt.Println("Batch delete complete")
}
```

### Get Status and Pool Statistics

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Address: %s\n", status["addr"])
fmt.Printf("Database: %v\n", status["db"])
fmt.Printf("Ping: %s\n", status["ping"])

// Pool statistics
fmt.Printf("Pool hits: %v\n", status["pool_hits"])
fmt.Printf("Pool misses: %v\n", status["pool_misses"])
fmt.Printf("Pool timeouts: %v\n", status["pool_timeouts"])
fmt.Printf("Total connections: %v\n", status["pool_total_conns"])
fmt.Printf("Idle connections: %v\n", status["pool_idle_conns"])
```

### Async Job Execution

```go
// Submit custom jobs to the worker pool
manager.SubmitAsyncJob(func() {
    ctx := context.Background()
    manager.Set(ctx, "background:key", "background-value", 3600*time.Second)
})
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/redis/go-redis/v9` | Official Redis client for Go with connection pooling |
| `stackyrd/config` | Internal configuration package with `RedisConfig` struct |
| `stackyrd/pkg/logger` | Structured logging (though not directly used in current implementation) |
| Standard library | `context`, `time`, `fmt` |

## Error Handling

The library uses Go error handling patterns via `go-redis`:

- **Connection failures**: `NewRedisClient()` validates connectivity with `Ping()`
- **Key not found**: Returns `redis.Nil` error (handle with `errors.Is(err, redis.Nil)`)
- **Context cancellation**: All operations respect `context.Context` for timeouts
- **Nil checks**: Public methods handle nil receiver gracefully in `GetStatus()`

Example error handling:

```go
value, err := manager.Get(ctx, "mykey")
if err != nil {
    if errors.Is(err, redis.Nil) {
        // Key doesn't exist
        return nil
    }
    return fmt.Errorf("failed to get key: %w", err)
}
```

## Common Pitfalls

### 1. Redis Not Running

**Problem**: `failed to connect to redis: ping: dial tcp... connection refused`

**Solution**: 
- Ensure Redis is running: `redis-cli ping` should return `PONG`
- Check server address: default is `localhost:6379`
- Start Redis: `redis-server` or `systemctl start redis`

### 2. Authentication Failed

**Problem**: `NOAUTH Authentication required`

**Solution**:
- Set correct password in `RedisConfig.Password`
- Or disable auth in `redis.conf`: `requirepass ""`

### 3. Wrong Database

**Problem**: Operations affecting wrong data

**Solution**:
- Verify `RedisConfig.DB` (0-15 for default Redis config)
- Check database in `redis-cli`: `SELECT <db>`

### 4. Context Timeout

**Problem**: Operations timing out

**Solution**:
```go
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
// Operations will use this timeout
```

### 5. Pool Stats Interpretation

**Problem**: Confusion about pool statistics

**Solution**:
- `Hits`: Times a connection was found in the pool (good)
- `Misses`: Times no connection available, had to create new (may indicate pool too small)
- `Timeouts`: Times waited too long for connection (bad - increase pool size or check for leaks)
- `TotalConns`: Total connections ever created
- `IdleConns`: Currently idle connections in pool

### 6. Batch Operations Channel Order

**Problem**: Results in wrong order

**Solution**:
- `GetBatchAsync` returns results in the same order as input keys
- Use index to correlate: `results[i]` corresponds to `keys[i]`

## Advanced Usage

### Health Check Function

```go
func checkRedisHealth(manager *redis.RedisManager) bool {
    status := manager.GetStatus()
    
    connected, ok := status["connected"].(bool)
    if !ok || !connected {
        return false
    }
    
    // Check pool health
    misses, _ := status["pool_misses"].(int64)
    timeouts, _ := status["pool_timeouts"].(int64)
    
    if timeouts > 0 {
        fmt.Printf("Warning: %d pool timeouts detected\n", timeouts)
    }
    
    return true
}
```

### Monitoring Pool Health

```go
func monitorPoolHealth(manager *redis.RedisManager) {
    status := manager.GetStatus()
    
    fmt.Println("=== Redis Pool Health ===")
    fmt.Printf("Hits: %v\n", status["pool_hits"])
    fmt.Printf("Misses: %v\n", status["pool_misses"])
    fmt.Printf("Timeouts: %v\n", status["pool_timeouts"])
    fmt.Printf("Total Connections: %v\n", status["pool_total_conns"])
    fmt.Printf("Idle Connections: %v\n", status["pool_idle_conns"])
    
    // Calculate hit rate
    hits, _ := status["pool_hits"].(int64)
    misses, _ := status["pool_misses"].(int64)
    total := hits + misses
    if total > 0 {
        hitRate := float64(hits) / float64(total) * 100
        fmt.Printf("Hit Rate: %.2f%%\n", hitRate)
    }
}
```

### Cached User Session Example

```go
type UserSession struct {
    UserID    string
    Username  string
    ExpiresAt time.Time
}

func cacheUserSession(manager *redis.RedisManager, session UserSession) error {
    ctx := context.Background()
    
    // Serialize session (simplified - use JSON in production)
    value := fmt.Sprintf("%s:%s", session.UserID, session.Username)
    
    // Calculate TTL
    ttl := time.Until(session.ExpiresAt)
    if ttl <= 0 {
        return fmt.Errorf("session already expired")
    }
    
    return manager.Set(ctx, "session:"+session.UserID, value, ttl)
}

func getCachedSession(manager *redis.RedisManager, userID string) (*UserSession, error) {
    ctx := context.Background()
    
    value, err := manager.Get(ctx, "session:"+userID)
    if err != nil {
        return nil, err
    }
    
    // Parse session (simplified)
    parts := strings.Split(value, ":")
    if len(parts) != 2 {
        return nil, fmt.Errorf("invalid session format")
    }
    
    return &UserSession{
        UserID:   parts[0],
        Username: parts[1],
    }, nil
}
```

### Batch Cache Warming

```go
func warmCache(manager *redis.RedisManager, data map[string]interface{}) error {
    ctx := context.Background()
    
    // Batch set with 1 hour TTL
    result := manager.SetBatchAsync(ctx, data, 3600*time.Second)
    
    // Wait for completion
    select {
    case <-result.Ch:
        fmt.Printf("Cache warmed with %d items\n", len(data))
        return nil
    case err := <-result.ErrCh:
        return fmt.Errorf("cache warm failed: %w", err)
    case <-time.After(30 * time.Second):
        return fmt.Errorf("cache warm timed out")
    }
}
```

### Custom Worker Pool Size

The default worker pool has 10 workers. To modify, you would need to adjust the `NewRedisClient` function or extend the struct to accept a custom pool size.

## Internal Algorithms

### go-redis Connection Pooling

The `go-redis/v9` client manages its own connection pool internally:

```
Pool Configuration (defaults):
  - MaxRetries: 0 (no retries by default)
  - MinIdleConns: 0
  - PoolSize: 10 * runtime.NumCPU()
  - PoolTimeout: 4 seconds
  - IdleTimeout: 5 minutes
```

### Async Execution Pattern

```
ExecuteAsync(ctx, operation)
    │
    └── Submit to WorkerPool
            │
            └── Worker executes operation in goroutine
                    │
                    └── Send result to channel (result.Ch)
```

### Batch Async Execution

```
ExecuteBatchAsync(ctx, operations)
    │
    ├── Create *BatchAsyncResult[T]
    ├── For each operation:
    │   └── Submit to WorkerPool
    │           │
    │           └── Execute operation
    │                   │
    │                   └── Send result to result.Ch[i]
    │                   └── Send error to result.ErrCh[i]
    └── Return *BatchAsyncResult[T]
```

### Status Collection

```
GetStatus()
    │
    ├── Ping server (context.Background(), 5s timeout)
    ├── Collect pool stats:
    │   ├── Hits, Misses, Timeouts
    │   ├── TotalConns, IdleConns
    └── Return map[string]interface{}
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.