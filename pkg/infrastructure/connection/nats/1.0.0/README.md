# NATS Manager

## Overview

The `NATSManager` is a Go library for managing NATS operations via the monitoring HTTP API. It provides access to server info (varz), connections (connz), subscriptions (subsz), routes (routez), and leaf nodes (leafz) with async support via a worker pool.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Server Info**: Get server information via `/varz` endpoint
- **Connections**: List connections via `/connz` endpoint
- **Subscriptions**: Get subscription statistics via `/subsz` endpoint
- **Routes**: List routes via `/routez` endpoint
- **Leaf Nodes**: Get leaf nodes via `/leafz` endpoint
- **HTTP Client**: Uses retryablehttp with retry logic (max 3, 1-10s wait)
- **Async Operations**: Async API requests via worker pool
- **Worker Pool**: Async job execution support (6 workers)
- **Connection Test**: Validates connectivity via `/varz` endpoint

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
    
    // Create NATS manager (configuration via viper)
    manager, err := nats.NewNATSManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Get server info
    varz, err := manager.GetVarz(ctx)
    if err != nil {
        panic(err)
    }
    fmt.Printf("Server ID: %s\n", varz.ServerID)
    fmt.Printf("Version: %s\n", varz.Version)
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `NATSManager` | Main manager with retryablehttp client, base URL, worker pool |
| `NATSVarz` | Server info (server_id, version, go, host, port, etc.) |
| `NATSConnz` | Connections info (num_connections, total, connections list) |
| `NATSConnInfo` | Single connection info (cid, ip, port, start, etc.) |
| `NATSSubsz` | Subscription statistics (num_subscriptions, cache hit rate, etc.) |
| `NATSRoutez` | Routes info (num_routes, routes list) |
| `NATSRoute` | Single route info (rid, remote_id, ip, etc.) |
| `NATSLeafz` | Leaf nodes info (num_leafnodes, leafnodes list) |
| `NATSLeaf` | Single leaf node info (account, ip, port, etc.) |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   NATSManager                            │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │ retryablehttp│  → HTTP client with retry logic        │
│  │ Client       │  → Timeout: 30s                       │
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
│  BaseURL: NATS monitoring URL (default: http://localhost:8222) │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewNATSManager(logger)
    │
    ├── Check viper config: "nats.enabled"
    ├── Get baseURL: "nats.monitoring_url" (default: http://localhost:8222)
    │
    ├── Create retryablehttp.Client:
    │   ├── RetryMax = 3
    │   ├── RetryWaitMin = 1s
    │   ├── RetryWaitMax = 10s
    │   └── HTTPClient.Timeout = 30s
    │
    ├── Test connection: GetVarz(ctx)
    ├── Create WorkerPool(6)
    ├── Start worker pool
    └── Return NATSManager
```

### 2. API Request Flow

```
apiRequest(ctx, method, path, body)
    │
    ├── Build URL: BaseURL + path
    ├── If body: JSON marshal, create reader
    ├── Create retryablehttp.NewRequestWithContext
    ├── Set Content-Type: application/json
    ├── Execute client.Do(req)
    ├── Check status code (200-299)
    ├── Read body: io.ReadAll(resp.Body)
    └── Return []byte or error
```

### 3. Get Server Info Flow

```
GetVarz(ctx)
    │
    ├── apiRequest(ctx, "GET", "/varz", nil)
    ├── JSON unmarshal to NATSVarz
    └── Return NATSVarz
```

### 4. Get Connections Flow

```
GetConnz(ctx)
    │
    ├── apiRequest(ctx, "GET", "/connz", nil)
    ├── JSON unmarshal to NATSConnz
    └── Return NATSConnz
```

### 5. Get Subscriptions Flow

```
GetSubsz(ctx)
    │
    ├── apiRequest(ctx, "GET", "/subsz", nil)
    ├── JSON unmarshal to NATSSubsz
    └── Return NATSSubsz
```

### 6. Get Routes Flow

```
GetRoutez(ctx)
    │
    ├── apiRequest(ctx, "GET", "/routez", nil)
    ├── JSON unmarshal to NATSRoutez
    └── Return NATSRoutez
```

### 7. Get Leaf Nodes Flow

```
GetLeafz(ctx)
    │
    ├── apiRequest(ctx, "GET", "/leafz", nil)
    ├── JSON unmarshal to NATSLeafz
    └── Return NATSLeafz
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `nats.enabled` | bool | false | Enable/disable NATS manager |
| `nats.monitoring_url` | string | "http://localhost:8222" | NATS monitoring URL |

### Environment Variables

The NATS manager does not directly use environment variables. Configuration is read through Viper, which can load from:
- Configuration files (JSON, YAML, TOML)
- Environment variables (if Viper is configured for it)

You can set NATS parameters via:
```go
viper.Set("nats.enabled", true)
viper.Set("nats.monitoring_url", "http://nats-server:8222")
```

## Usage Examples

### Getting Server Info

```go
ctx := context.Background()

varz, err := manager.GetVarz(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Server ID: %s\n", varz.ServerID)
fmt.Printf("Version: %s\n", varz.Version)
fmt.Printf("Go Version: %s\n", varz.GoVersion)
fmt.Printf("Host: %s:%d\n", varz.Host, varz.Port)
fmt.Printf("Max Connections: %d\n", varz.MaxConns)
fmt.Printf("Max Payload: %d\n", varz.MaxPayload)
fmt.Printf("Max Pending: %d\n", varz.MaxPending)
fmt.Printf("TLS Verify: %v\n", varz.TLSVerify)
```

### Getting Connections

```go
ctx := context.Background()

connz, err := manager.GetConnz(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Total Connections: %d\n", connz.Total)
fmt.Printf("Number of Connections: %d\n", connz.NumConns)

for _, conn := range connz.Conns {
    fmt.Printf("Connection %d: %s:%d (Name: %s)\n", 
        conn.CID, conn.IP, conn.Port, conn.Name)
    fmt.Printf("  Start: %s, Last Activity: %s\n", conn.Start, conn.LastActivity)
    fmt.Printf("  Uptime: %s, Idle: %s\n", conn.Uptime, conn.Idle)
    fmt.Printf("  Pending Bytes: %d\n", conn.PendingBytes)
    fmt.Printf("  In Msgs: %d, Out Msgs: %d\n", conn.InMsgs, conn.OutMsgs)
    fmt.Printf("  In Bytes: %d, Out Bytes: %d\n", conn.InBytes, conn.OutBytes)
    fmt.Printf("  Subscriptions: %d\n", conn.Subscriptions)
    if conn.Lang != "" {
        fmt.Printf("  Lang: %s, Version: %s\n", conn.Lang, conn.Version)
    }
}
```

### Getting Subscriptions

```go
ctx := context.Background()

subsz, err := manager.GetSubsz(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Subscriptions: %d\n", subsz.NumSubs)
fmt.Printf("Cache: %d\n", subsz.NumCache)
fmt.Printf("Inserts: %d, Removes: %d\n", subsz.NumInserts, subsz.NumRemoves)
fmt.Printf("Matches: %d\n", subsz.NumMatches)
fmt.Printf("Cache Hit Rate: %.2f%%\n", subsz.CacheHitRate*100)
fmt.Printf("Max RSS: %d\n", subsz.MaxRSS)
```

### Getting Routes

```go
ctx := context.Background()

routez, err := manager.GetRoutez(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Routes: %d\n", routez.NumRoutes)

for _, route := range routez.Routes {
    fmt.Printf("Route %s: Remote ID: %s\n", route.RID, route.RemoteID)
    fmt.Printf("  Address: %s:%d\n", route.IP, route.Port)
    fmt.Printf("  In Msgs: %d, Out Msgs: %d\n", route.InMsgs, route.OutMsgs)
    fmt.Printf("  In Bytes: %d, Out Bytes: %d\n", route.InBytes, route.OutBytes)
    fmt.Printf("  Subscriptions: %d\n", route.Subscriptions)
}
```

### Getting Leaf Nodes

```go
ctx := context.Background()

leafz, err := manager.GetLeafz(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Leaf Nodes: %d\n", leafz.NumLeafs)

for _, leaf := range leafz.Leafs {
    fmt.Printf("Leaf: Account: %s\n", leaf.Account)
    fmt.Printf("  Address: %s:%d\n", leaf.IP, leaf.Port)
    fmt.Printf("  RTT: %s\n", leaf.RTT)
    fmt.Printf("  In Msgs: %d, Out Msgs: %d\n", leaf.InMsgs, leaf.OutMsgs)
}
```

### Get Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Server ID: %s\n", status["server_id"])
fmt.Printf("Version: %s\n", status["version"])
fmt.Printf("URL: %s\n", status["url"])

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
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `bytes`, `context`, `encoding/json`, `fmt`, `io`, `time` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: `testConnection()` validates via `GetVarz()`
- **API errors**: HTTP status codes checked and errors returned with response body
- **Context cancellation**: All operations respect `context.Context`
- **Nil checks**: Public methods handle nil receiver gracefully in `GetStatus()`

Example error handling:

```go
varz, err := manager.GetVarz(ctx)
if err != nil {
    if strings.Contains(err.Error(), "connection refused") {
        return fmt.Errorf("NATS monitor not reachable: %w", err)
    }
    if strings.Contains(err.Error(), "timeout") {
        return fmt.Errorf("NATS monitor timeout: %w", err)
    }
    return fmt.Errorf("failed to get varz: %w", err)
}
```

## Common Pitfalls

### 1. NATS Monitor Not Reachable

**Problem**: `connection refused` or `no route to host` errors

**Solution**: 
- Verify monitoring URL: `viper.Set("nats.monitoring_url", "http://localhost:8222")`
- Check NATS server has monitoring enabled (default port 8222)
- Verify firewall allows access to monitoring port

### 2. Monitoring Not Enabled

**Problem**: `404 Not Found` or `connection refused`

**Solution**:
- Enable NATS monitoring when starting server: `nats-server -m 8222`
- Check that the monitoring endpoint is accessible via curl

### 3. Timeout Issues

**Problem**: Operations timing out

**Solution**:
```go
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
```

The HTTP client timeout is set to 30s, but you can adjust via context.

### 4. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewNATSManager()` succeeded
- Verify `nats.enabled` is true

### 5. JSON Unmarshal Errors

**Problem**: Failed to decode response

**Solution**:
- Check that NATS monitoring API returns expected JSON format
- Verify the response structure matches the defined structs

## Advanced Usage

### Monitoring Server Health

```go
func monitorNATS(manager *nats.NATSManager) {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            ctx := context.Background()
            varz, err := manager.GetVarz(ctx)
            if err != nil {
                fmt.Printf("Error getting varz: %v\n", err)
                continue
            }
            fmt.Printf("NATS Server: %s, Version: %s\n", varz.ServerID, varz.Version)
            fmt.Printf("  Max Conns: %d, Max Payload: %d\n", varz.MaxConns, varz.MaxPayload)
        }
    }
}
```

### Connection Monitoring

```go
func monitorConnections(manager *nats.NATSManager) {
    ctx := context.Background()
    
    connz, err := manager.GetConnz(ctx)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    fmt.Printf("Total Connections: %d\n", connz.Total)
    for _, conn := range connz.Conns {
        fmt.Printf("  %s:%d - Msgs In: %d, Out: %d\n", 
            conn.IP, conn.Port, conn.InMsgs, conn.OutMsgs)
    }
}
```

### Subscription Analysis

```go
func analyzeSubscriptions(manager *nats.NATSManager) {
    ctx := context.Background()
    
    subsz, err := manager.GetSubsz(ctx)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    fmt.Printf("Subscription Statistics:\n")
    fmt.Printf("  Total: %d\n", subsz.NumSubs)
    fmt.Printf("  Cache Hit Rate: %.2f%%\n", subsz.CacheHitRate*100)
    fmt.Printf("  Inserts: %d, Removes: %d\n", subsz.NumInserts, subsz.NumRemoves)
    fmt.Printf("  Matches: %d\n", subsz.NumMatches)
}
```

### Route Monitoring

```go
func monitorRoutes(manager *nats.NATSManager) {
    ctx := context.Background()
    
    routez, err := manager.GetRoutez(ctx)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    fmt.Printf("Routes: %d\n", routez.NumRoutes)
    for _, route := range routez.Routes {
        fmt.Printf("  Route to %s: In: %d msgs, Out: %d msgs\n", 
            route.RemoteID, route.InMsgs, route.OutMsgs)
    }
}
```

## Internal Algorithms

### API Request with Retry

```
apiRequest(ctx, method, path, body):
    │
    ├── url = BaseURL + path
    ├── If body: JSON marshal → reader
    ├── retryablehttp.NewRequestWithContext(ctx, method, url, reader)
    ├── Set Content-Type: application/json
    ├── client.Do(req)
    ├── Check status code 200-299
    ├── io.ReadAll(resp.Body)
    └── Return []byte or error
```

### Connection Test

```
testConnection():
    │
    ├── GetVarz(ctx) with timeout 10s
    └── Return error if fails
```

### Worker Pool Integration

```
Async Operations:
    │
    ├── ExecuteAsync(ctx, func)
    │   └── Returns *AsyncResult[T]
    │
    ├── Result channels:
    │   ├── Ch: Receives result value
    │   └── ErrCh: Receives error
    │
    └── Usage: select on Ch and ErrCh
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.