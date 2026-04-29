# Redis Manager

## Overview

The `RedisManager` is a Go library for managing Redis operations. It provides basic key-value operations (Set, Get, Delete, Replace) and monitoring capabilities using the `github.com/redis/go-redis/v9` client with a worker pool for async operations.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Key-Value Operations**: Set (with TTL), Get, Delete, Replace (SetXX)
- **Monitoring**: GetStatus (ping, pool stats), GetInfo (server info), ScanKeys (pattern matching), GetValue (get specific key)
- **Async Operations**: Async versions of Set, Get, Delete, Replace, GetInfo, ScanKeys, GetValue via `ExecuteAsync`
- **Batch Operations**: Batch Set, Get, Delete via `ExecuteBatchAsync`
- **Worker Pool**: Async job execution support (10 workers)
- **Status Monitoring**: Get connection status, ping response, pool statistics
- **Connection Pool**: Uses Redis client's built-in connection pool

## Quick Start

```go
package main

import (
    "context"
    "fmt"
    "stackyrd/config"
    "stackyrd/pkg/infrastructure"
    "stackyrd/pkg/logger"
    "time"
)

func main() {
    // Initialize logger
    log := logger.NewLogger()
    
    // Configuration (typically via viper/config)
    cfg := config.RedisConfig{
        Enabled:  true,
        Address:  "localhost:6379",
        Password: "secret",
        DB:       0,
    }
    
    // Create Redis manager
    manager, err := redis.NewRedisClient(cfg, log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Set a key
    err = manager.Set(ctx, "mykey", "myvalue", 5*time.Minute)
    if err != nil {
        panic(err)
    }
    fmt.Println("Key set")
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `RedisManager` | Main manager with Redis client and worker pool |
| `RedisConfig` | Configuration (address, password, db, enabled) |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   RedisManager                            │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  redis.Client│  → Redis operations                   │
│  │             │  → Set, Get, Del, SetXX            │
│  └────────────────┘                                      │
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
│  Address: host:port (default: localhost:6379)             │
│  DB: Redis database number (default: 0)                    │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewRedisClient(cfg, logger)
    │
    ├── Check cfg.Enabled
    ├── Create redis.Client with options:
    │   ├── Addr: cfg.Address
    │   ├── Password: cfg.Password
    │   └── DB: cfg.DB
    │
    ├── Test connection: client.Ping(context.Background())
    ├── If error: return error
    │
    ├── Create WorkerPool(10)
    ├── Start worker pool
    └── Return RedisManager
```

### 2. Set Operation Flow

```
Set(ctx, key, value, ttl)
    │
    └── Client.Set(ctx, key, value, ttl).Err()
```

### 3. Get Operation Flow

```
Get(ctx, key)
    │
    └── Client.Get(ctx, key).Result() → (string, error)
```

### 4. Delete Operation Flow

```
Delete(ctx, key)
    │
    └── Client.Del(ctx, key).Err()
```

### 5. Replace Operation Flow

```
Replace(ctx, key, value, ttl)
    │
    └── Client.SetXX(ctx, key, value, ttl).Err()
```

### 6. GetStatus Flow

```
GetStatus()
    │
    ├── If nil or Client == nil: return connected=false
    ├── Ping: Client.Ping(context.Background()).Result()
    ├── Get pool stats: Client.PoolStats()
    └── Return connected, ping, addr, db, pool_hits, pool_misses, pool_timeouts, pool_total_conns, pool_idle_conns
```

### 7. GetInfo Flow

```
GetInfo(ctx)
    │
    └── Client.Info(ctx).Result() → (string, error)
```

### 8. ScanKeys Flow

```
ScanKeys(ctx, pattern)
    │
    ├── Create slice for keys
    ├── Create iterator: Client.Scan(ctx, 0, pattern, 100)
    ├── Iterate and collect keys
    └── Return keys, error
```

## Configuration

### Configuration Struct

```go
type RedisConfig struct {
    Enabled  bool   `mapstructure:"enabled"`
    Address  string `mapstructure:"address"`
    Password string `mapstructure:"password"`
    DB       int    `mapstructure:"db"`
}
```

### Viper Configuration Keys

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `redis.enabled` | bool | false | Enable/disable Redis manager |
| `redis.address` | string | "localhost:6379" | Redis server address |
| `redis.password` | string | "" | Password for Redis |
| `redis.db` | int | 0 | Redis database number |

### Environment Variables

Configuration is read through Viper, which can load from files or environment variables.

Example programmatic configuration:

```go
cfg := config.RedisConfig{
    Enabled:  true,
    Address:  "localhost:6379",
    Password: "secret",
    DB:       0,
}
```

## Usage Examples

### Setting a Key

```go
ctx := context.Background()

err := manager.Set(ctx, "username", "john_doe", 10*time.Minute)
if err != nil {
    panic(err)
}
fmt.Println("Key set with TTL")
```

### Getting a Key

```go
ctx := context.Background()

value, err := manager.Get(ctx, "username")
if err != nil {
    panic(err)
}
fmt.Printf("Value: %s\n", value)
```

### Deleting a Key

```go
ctx := context.Background()

err := manager.Delete(ctx, "username")
if err != nil {
    panic(err)
}
fmt.Println("Key deleted")
```

### Replace (Set if exists)

```go
ctx := context.Background()

err := manager.Replace(ctx, "username", "new_value", 5*time.Minute)
if err != nil {
    // Returns error if key doesn't exist
    fmt.Println("Key not replaced (may not exist)")
} else {
    fmt.Println("Key replaced")
}
```

### Getting Server Info

```go
ctx := context.Background()

info, err := manager.GetInfo(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Redis Info:\n%s\n", info)
```

### Scanning Keys

```go
ctx := context.Background()

keys, err := manager.ScanKeys(ctx, "user:*")
if err != nil {
    panic(err)
}
fmt.Printf("Found %d keys:\n", len(keys))
for _, key := range keys {
    fmt.Printf("  - %s\n", key)
}
```

### Getting a Value for Monitoring

```go
ctx := context.Background()

value, err := manager.GetValue(ctx, "some_key")
if err != nil {
    fmt.Printf("Key not found: %v\n", err)
} else {
    fmt.Printf("Value: %s\n", value)
}
```

### Async Operations

```go
ctx := context.Background()

// Async set
asyncResult := manager.SetAsync(ctx, "async_key", "async_value", time.Hour)
select {
case <-asyncResult.Ch:#    fmt.Println("Async set completed")
case err := <-asyncResult.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async get
asyncResult2 := manager.GetAsync(ctx, "async_key")
select {
case value := <-asyncResult2.Ch:#    fmt.Printf("Async got: %s\n", value)
case err := <-asyncResult2.ErrCh:#    fmt.Printf("Error: %v\n", err)
}
```

### Batch Operations

```go
ctx := context.Background()

// Batch set
kvPairs := map[string]interface{}{
    "key1": "value1",
    "key2": "value2",
    "key3": "value3",
}
batchResult := manager.SetBatchAsync(ctx, kvPairs, 10*time.Minute)
// Wait for all operations
for i, opResult := range batchResult.Results {
    if opResult.Error != nil {
        fmt.Printf("Set %d failed: %v\n", i, opResult.Error)
    } else {
        fmt.Printf("Set %d succeeded\n", i)
    }
}

// Batch get
keys := []string{"key1", "key2", "key3"}
batchResult2 := manager.GetBatchAsync(ctx, keys)
for i, opResult := range batchResult2.Results {
    if opResult.Error != nil {
        fmt.Printf("Get %d failed: %v\n", i, opResult.Error)
    } else {
        fmt.Printf("Get %d: %s\n", i, opResult.Result)
    }
}

// Batch delete
batchResult3 := manager.DeleteBatchAsync(ctx, keys)
```

### Getting Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
if status["connected"] == true {
    fmt.Printf("Ping: %v\n", status["ping"])
    fmt.Printf("Address: %v\n", status["addr"])
    fmt.Printf("DB: %v\n", status["db"])
    fmt.Printf("Pool Hits: %v\n", status["pool_hits"])
    fmt.Printf("Pool Misses: %v\n", status["pool_misses"])
}
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/redis/go-redis/v9` | Redis client for Go |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `context`, `fmt`, `time` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: Returned from `client.Ping()`
- **Operation failures**: Returned from Redis client methods
- **Nil checks**: `GetStatus()` handles nil receiver gracefully

Example error handling:

```go
err := manager.Set(ctx, key, value, ttl)
if err != nil {
    if strings.Contains(err.Error(), "connection refused") {
        return fmt.Errorf("redis not reachable: %w", err)
    }
    if strings.Contains(err.Error(), "NOAUTH") {
        return fmt.Errorf("authentication failed: %w", err)
    }
    if strings.Contains(err.Error(), "WRONGTYPE") {
        return fmt.Errorf("wrong type for key: %w", err)
    }
    return fmt.Errorf("set failed: %w", err)
}
```

## Common Pitfalls

### 1. Redis Not Reachable

**Problem**: `connection refused` errors

**Solution**: 
- Verify Redis is running
- Check address in configuration
- Default port is 6379
- Check firewall settings

### 2. Authentication Failed

**Problem**: `NOAUTH` or `ERR invalid password` errors

**Solution**:
- Verify password in `redis.password`
- Check if Redis requires authentication
- Ensure password is correct

### 3. Wrong Type

**Problem**: `WRONGTYPE` errors when operating on a key

**Solution**:
- Ensure key contains expected type (string, list, hash, etc.)
- Use `type` command to check key type
- Delete and recreate if necessary

### 4. Pool Exhaustion

**Problem**: High number of pool timeouts

**Solution**:
- Redis client has built-in connection pool
- Adjust pool size via `redis.Options.PoolSize` (not exposed in current config)
- Monitor pool stats via `GetStatus()`

### 5. TTL Issues

**Problem**: Keys expiring too soon or not expiring

**Solution**:
- Ensure TTL is set correctly (use `time.Duration`)
- TTL=0 means no expiration
- Check Redis `TTL` command to verify

### 6. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewRedisClient()` succeeded
- Verify `redis.enabled` is true

## Advanced Usage

### Using Redis for Caching

```go
func cacheExample(manager *redis.RedisManager) error {
    ctx := context.Background()
    
    // Try to get from cache
    cached, err := manager.Get(ctx, "cache:user:1")
    if err == nil {
        fmt.Printf("Cache hit: %s\n", cached)
        return nil
    }
    
    // Simulate expensive operation
    value := "expensive_result"
    // Set to cache with 5 minute TTL
    if err := manager.Set(ctx, "cache:user:1", value, 5*time.Minute); err != nil {
        return err
    }
    fmt.Println("Cached result")
    return nil
}
```

### Health Check

```go
func healthCheck(manager *redis.RedisManager) error {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    status := manager.GetStatus()
    if status["connected"] == false {
        return fmt.Errorf("not connected to Redis")
    }
    
    // Additional check: ping
    pong, err := manager.Client.Ping(ctx).Result()
    if err != nil {
        return err
    }
    fmt.Printf("Redis PONG: %s\n", pong)
    return nil
}
```

### Using Scan for Iteration

```go
func scanAllKeys(manager *redis.RedisManager, pattern string) ([]string, error) {
    ctx := context.Background()
    var allKeys []string
    
    iter := manager.Client.Scan(ctx, 0, pattern, 100).Iterator()
    for iter.Next(ctx) {
        allKeys = append(allKeys, iter.Val())
    }
    
    if err := iter.Err(); err != nil {
        return nil, err
    }
    
    return allKeys, nil
}
```

### Monitoring Pool Stats

```go
func monitorPool(manager *redis.RedisManager) {
    status := manager.GetStatus()
    
    fmt.Printf("Pool Stats:\n")
    fmt.Printf("  Hits: %v\n", status["pool_hits"])
    fmt.Printf("  Misses: %v\n", status["pool_misses"])
    fmt.Printf("  Timeouts: %v\n", status["pool_timeouts"])
    fmt.Printf("  Total Connections: %v\n", status["pool_total_conns"])
    fmt.Printf("  Idle Connections: %v\n", status["pool_idle_conns"])
}
```

## Internal Algorithms

### Connection Test

```
Initialization:
    │
    └── client.Ping(context.Background())
```

### Set Operation

```
Set():
    │
    └── Client.Set(ctx, key, value, ttl) → check Err()
```

### Get Operation

```
Get():
    │
    └── Client.Get(ctx, key) → Result()
```

### Status Collection

```
GetStatus():
    │
    ├── Client.Ping(context.Background()).Result()
    ├── Client.PoolStats() → redis.PoolStats
    └── Return map with connected, ping, addr, db, pool stats
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.