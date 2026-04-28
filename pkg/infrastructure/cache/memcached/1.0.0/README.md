# Memcached Manager

## Overview

The `MemcachedManager` is a Go library for managing Memcached operations via the Memcached text protocol. It provides a simple interface for caching operations including get, set, delete, and statistics retrieval with built-in connection retry logic and async job support.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Text Protocol**: Direct TCP communication using Memcached text protocol
- **Connection Retry**: Automatic retries with exponential backoff via `go-retryablehttp`
- **Multiple Servers**: Support for multiple Memcached server instances
- **Statistics**: Retrieve server statistics and slab information
- **Worker Pool**: Async job execution support
- **Context Support**: Full `context.Context` support for timeouts

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
    
    // Create Memcached manager (configuration via viper)
    manager, err := memcached.NewMemcachedManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Set a value
    err = manager.Set(ctx, memcached.MemcachedItem{
        Key:        "mykey",
        Value:       "myvalue",
        Flags:       0,
        Expiration:  3600, // 1 hour
    })
    
    // Get a value
    item, err := manager.Get(ctx, "mykey")
    if err == nil {
        fmt.Printf("Value: %s\n", item.Value)
    }
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `MemcachedManager` | Main manager with retryable HTTP client and worker pool |
| `MemcachedItem` | Represents a cached item with key, value, flags, expiration, and CAS |
| `MemcachedStats` | Server statistics with server address and stats map |
| `MemcachedSlab` | Slab information (ID, chunk size, pages, chunks) |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   MemcachedManager                          │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │ retryablehttp  │  → Retry logic (max 3, 1-5s wait) │
│  │ Client         │                                      │
│  └────────────────┘                                      │
│                                                         │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ SubmitAsyncJob│                   │
│  │  (6 workers)│      └──────────────┘                   │
│  └─────────────┘                                          │
│                                                         │
│  Servers: []string (e.g., "localhost:11211")             │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewMemcachedManager()
    │
    ├── Check viper config: "memcached.enabled"
    ├── Get servers: "memcached.servers" (default: ["localhost:11211"])
    ├── Create retryablehttp.Client (max 3 retries, 1-5s wait)
    ├── Test connection: manager.testConnection() → Stats()
    ├── Create WorkerPool(6)
    └── Return MemcachedManager
```

### 2. Command Execution Flow

```
sendCommand(ctx, server, cmd)
    │
    ├── dialServer(ctx, server) → net.Dialer with timeout
    ├── Write command: "cmd\r\n"
    ├── Read response lines until "END", "OK", or "ERROR"
    └── Return joined response lines
```

### 3. Get Operation Flow

```
Get(ctx, key)
    │
    ├── For each server in Servers:
    │   ├── sendCommand(ctx, server, "get <key>")
    │   ├── If result not empty:
    │   │   ├── Parse: fields[3] = value
    │   │   └── Return MemcachedItem
    │   └── Continue to next server
    └── Return "key not found" error
```

### 4. Set Operation Flow

```
Set(ctx, item)
    │
    ├── Build command: "set <key> <flags> <expiration> <len(value)>\r\n<value>"
    ├── For each server in Servers:
    │   ├── sendCommand(ctx, server, command)
    │   └── If success: return nil
    └── Return "failed to set key" error
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `memcached.enabled` | bool | false | Enable/disable Memcached manager |
| `memcached.servers` | []string | ["localhost:11211"] | List of Memcached server addresses |

### Environment Variables

The Memcached manager does not directly use environment variables. Configuration is read through Viper, which can load from:
- Configuration files (JSON, YAML, TOML)
- Environment variables (if Viper is configured for it)
- Command-line flags

## Usage Examples

### Basic Operations

```go
ctx := context.Background()

// Set a value with TTL
err := manager.Set(ctx, memcached.MemcachedItem{
    Key:        "user:123",
    Value:       `{"name":"John","age":30}`,
    Flags:       0,
    Expiration:  3600, // 1 hour in seconds
})

// Get a value
item, err := manager.Get(ctx, "user:123")
if err == nil {
    fmt.Printf("Key: %s, Value: %s\n", item.Key, item.Value)
}

// Delete a key
err = manager.Delete(ctx, "user:123")

// Flush all data (clear cache)
err = manager.FlushAll(ctx)
```

### Get Server Statistics

```go
// Get stats from all servers
allStats, err := manager.Stats(ctx)
if err != nil {
    fmt.Printf("Error getting stats: %v\n", err)
}

for _, serverStats := range allStats {
    fmt.Printf("Server: %s\n", serverStats.Server)
    for key, value := range serverStats.Stats {
        fmt.Printf("  %s: %s\n", key, value)
    }
}
```

### Working with Different Servers

```go
// Configure multiple servers via viper
viper.Set("memcached.servers", []string{
    "cache1.example.com:11211",
    "cache2.example.com:11211",
    "cache3.example.com:11211",
})

// Manager will try each server for get operations
// Set operations are sent to all servers
```

### Async Job Execution

```go
// Submit custom jobs to the worker pool
manager.SubmitAsyncJob(func() {
    ctx := context.Background()
    manager.Set(ctx, memcached.MemcachedItem{
        Key:   "background:key",
        Value:  "async-value",
    })
})
```

### Get Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Servers: %v\n", status["servers"])

if stats, ok := status["stats"].([]memcached.MemcachedStats); ok {
    for _, s := range stats {
        fmt.Printf("Server %s has %d stats\n", s.Server, len(s.Stats))
    }
}
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/hashicorp/go-retryablehttp` | HTTP client with retry logic for connection resilience |
| `github.com/spf13/viper` | Configuration management for servers and enable/disable |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `context`, `net`, `bufio`, `strings`, `time`, `fmt` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: `testConnection()` validates connectivity on startup
- **Server iteration**: Get/Set/Delete iterate through servers, continuing on individual failures
- **Context cancellation**: All operations respect `context.Context` for timeouts
- **Nil checks**: Public methods handle nil receiver gracefully in `GetStatus()`

Example error handling:

```go
item, err := manager.Get(ctx, "mykey")
if err != nil {
    if strings.Contains(err.Error(), "key not found") {
        // Key doesn't exist
        return nil
    }
    return fmt.Errorf("failed to get key: %w", err)
}
```

## Common Pitfalls

### 1. Memcached Not Running

**Problem**: `failed to connect to Memcached: ...`

**Solution**: 
- Ensure Memcached is running: `ps aux | grep memcached`
- Check server address: default is `localhost:11211`
- Start Memcached: `memcached -d -p 11211`

### 2. Connection Timeout

**Problem**: Operations timing out

**Solution**:
```go
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
// Operations will use this timeout
```

### 3. Multiple Servers Confusion

**Problem**: Data inconsistency with multiple servers

**Solution**:
- Get operations try servers sequentially (first hit wins)
- Set/Delete operations are sent to ALL servers
- Consider using a consistent hashing approach for client-side distribution

### 4. Expiration Values

**Problem**: Items expiring unexpectedly

**Solution**:
- Expiration is in seconds
- Values > 30 days (2592000 seconds) are interpreted as Unix timestamps
- Use 0 for no expiration

### 5. Flags Field

**Problem**: Not sure what flags to use

**Solution**:
- Flags is a generic integer field for client use
- Commonly used to store data type info (e.g., 0=string, 1=json, 2=gzip)
- The library doesn't interpret flags, just passes them through

## Advanced Usage

### Batch Operations Pattern

```go
// Set multiple keys (simplified - sends to all servers)
keys := map[string]string{
    "key1": "value1",
    "key2": "value2",
    "key3": "value3",
}

for key, value := range keys {
    key, value := key, value // capture loop variables
    manager.SubmitAsyncJob(func() {
        ctx := context.Background()
        manager.Set(ctx, memcached.MemcachedItem{
            Key:   key,
            Value:  value,
            Flags: 0,
        })
    })
}
```

### Custom Connection Timeout

```go
// The manager uses 5 second timeout by default
// To modify, you would need to adjust the manager's Timeout field
// or use context timeouts for individual operations

ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

_, err := manager.Get(ctx, "mykey")
// This will timeout after 2 seconds
```

### Monitoring with Stats

```go
func printMemcachedStats(manager *memcached.MemcachedManager) {
    stats, err := manager.Stats(context.Background())
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    for _, serverStat := range stats {
        fmt.Printf("\n=== Server: %s ===\n", serverStat.Server)
        s := serverStat.Stats
        
        if bytes, ok := s["bytes"]; ok {
            fmt.Printf("Bytes used: %s\n", bytes)
        }
        if currItems, ok := s["curr_items"]; ok {
            fmt.Printf("Current items: %s\n", currItems)
        }
        if hits, ok := s["get_hits"]; ok {
            fmt.Printf("Get hits: %s\n", hits)
        }
        if misses, ok := s["get_misses"]; ok {
            fmt.Printf("Get misses: %s\n", misses)
        }
    }
}
```

### Health Check Function

```go
func checkMemcachedHealth(manager *memcached.MemcachedManager) bool {
    status := manager.GetStatus()
    
    connected, ok := status["connected"].(bool)
    if !ok || !connected {
        return false
    }
    
    // Try a simple set/get operation
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    testKey := "_health_check_"
    err := manager.Set(ctx, memcached.MemcachedItem{
        Key:   testKey,
        Value:  "ok",
        Flags: 0,
    })
    
    if err != nil {
        return false
    }
    
    _, err = manager.Get(ctx, testKey)
    manager.Delete(ctx, testKey) // Cleanup
    
    return err == nil
}
```

## Internal Algorithms

### Text Protocol Communication

The manager uses Memcached text protocol over TCP:

**Get Command:**
```
Client: get mykey\r\n
Server: VALUE mykey 0 5\r\n
        hello\r\n
        END\r\n
```

**Set Command:**
```
Client: set mykey 0 3600 5\r\n
        hello\r\n
Server: STORED\r\n
```

**Delete Command:**
```
Client: delete mykey\r\n
Server: DELETED\r\n
```

**Flush All:**
```
Client: flush_all\r\n
Server: OK\r\n
```

### Stats Parsing

```
Stats Command:
Client: stats\r\n
Server: STAT pid 1234\r\n
        STAT uptime 3600\r\n
        STAT bytes 1024000\r\n
        END\r\n

Parsed into: map[string]string{
    "pid": "1234",
    "uptime": "3600",
    "bytes": "1024000",
}
```

### Retry Logic

Using `go-retryablehttp`:
- Maximum retries: 3
- Minimum wait: 1 second
- Maximum wait: 5 seconds
- HTTP client timeout: 5 seconds

Note: The current implementation uses `retryablehttp` but communicates via raw TCP. The retry logic may be more relevant if using the client for HTTP-based monitoring.

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.