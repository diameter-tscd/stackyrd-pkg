# Port Forwarder Manager

## Overview

The `PortForwarderManager` is a Go library for managing TCP port forwarding rules. It allows creating and removing port forwards that accept TCP connections on a local port and forward them to a target address, with statistics tracking and async support via a worker pool.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **TCP Port Forwarding**: Create TCP port forwards from local port to target address
- **Multiple Forwards**: Manage multiple port forwarding rules with unique IDs
- **Statistics Tracking**: Track bytes sent/received and connection count per forward
- **Forward Management**: Add and remove port forwards dynamically
- **Context Support**: Cancel forwards via context cancellation
- **Async Operations**: Async add/remove forwards via worker pool
- **Worker Pool**: Async job execution support (8 workers)
- **Status Monitoring**: Get statistics for all active forwards

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
    
    // Create Port Forwarder manager (configuration via viper)
    manager, err := port_forwarder.NewPortForwarderManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Add a port forward: local :8080 -> target localhost:80
    err = manager.AddForward(ctx, "web-proxy", "tcp", 8080, "localhost", 80)
    if err != nil {
        panic(err)
    }
    fmt.Println("Port forward created: :8080 -> localhost:80")
    
    // Keep running...
    select {}
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `PortForwarderManager` | Main manager with forwards map, worker pool |
| `PortForward` | Forward rule with ID, addresses, stats, context |
| `PortForwardStats` | Statistics (active, bytes sent/recv, connections, uptime) |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   PortForwarderManager                    │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  TCP        │  → Listeners and forwarded connections     │
│  │  Forwards   │  → Bidirectional io.Copy                │
│  └────────────────┘                                      │
│                                                         │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ AsyncResult │                   │
│  │  (8 workers)│      │   Channel   │                   │
│  └─────────────┘      └──────────────┘                   │
│         ▲                      │                            │
│         │                      ▼                            │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  SubmitJob  │      │ ExecuteAsync │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                         │
│  Forwards: Map of forward ID -> PortForward              │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewPortForwarderManager(logger)
    │
    ├── Check viper config: "port_forwarder.enabled"
    ├── Initialize forwards map
    ├── Create WorkerPool(8)
    ├── Start worker pool
    └── Return PortForwarderManager
```

### 2. Add Forward Flow

```
AddForward(ctx, id, protocol, listenPort, targetAddr, targetPort)
    │
    ├── Lock mutex
    ├── Check if ID already exists
    ├── Create context with cancel
    ├── ListenTCP on listenPort
    ├── Set forward.Active = true
    ├── Start goroutine: acceptTCPConnections(ctx, forward)
    ├── Store forward in forwards map
    └── Return nil
```

### 3. Accept TCP Connections Flow

```
acceptTCPConnections(ctx, forward)
    │
    └── Loop:
        ├── Select on ctx.Done() → return
        └── default:
            ├── listener.Accept()
            ├── If error: check ctx.Done(), sleep 1s, continue
            ├── forward.Connections++
            └── goroutine: handleTCPConnection(ctx, clientConn, forward)
```

### 4. Handle TCP Connection Flow

```
handleTCPConnection(ctx, clientConn, forward)
    │
    ├── defer clientConn.Close()
    ├── DialTimeout to target (10s)
    ├── If error: log warning, return
    ├── defer targetConn.Close()
    ├── Log new connection
    ├── WaitGroup.Add(2)
    ├── goroutine: io.Copy(target, client) → update BytesSent
    ├── goroutine: io.Copy(client, target) → update BytesRecv
    └── WaitGroup.Wait()
```

### 5. Remove Forward Flow

```
RemoveForward(id)
    │
    ├── Lock mutex
    ├── Get forward from map
    ├── If not exists: return error
    ├── Call forward.CancelFunc()
    ├── Close listener
    ├── Set forward.Active = false
    ├── Delete from map
    └── Return nil
```

### 6. Get Stats Flow

```
GetStats()
    │
    ├── Lock RLock
    ├── For each forward in map:
    │   └── Build PortForwardStats
    │       ├── ID, Active
    │       ├── BytesSent, BytesRecv
    │       ├── Connections
    │       └── Uptime (time.Since(CreatedAt))
    └── Return map of stats
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `port_forwarder.enabled` | bool | false | Enable/disable Port Forwarder manager |

### Environment Variables

The Port Forwarder manager does not directly use environment variables. Configuration is read through Viper, which can load from:
- Configuration files (JSON, YAML, TOML)
- Environment variables (if Viper is configured for it)

You can enable the manager via:
```go
viper.Set("port_forwarder.enabled", true)
```

## Usage Examples

### Creating a Port Forward

```go
ctx := context.Background()

// Forward local port 8080 to localhost:80
err := manager.AddForward(ctx, "web-proxy", "tcp", 8080, "localhost", 80)
if err != nil {
    panic(err)
}
fmt.Println("Port forward created: :8080 -> localhost:80")
```

### Removing a Port Forward

```go
// Remove the port forward
err := manager.RemoveForward("web-proxy")
if err != nil {
    panic(err)
}
fmt.Println("Port forward removed")
```

### Getting a Specific Forward

```go
forward, exists := manager.GetForward("web-proxy")
if !exists {
    fmt.Println("Forward not found")
} else {
    fmt.Printf("Forward %s: %s -> %s:%d\n", 
        forward.ID, forward.ListenAddr, forward.TargetAddr, forward.TargetPort)
    fmt.Printf("  Active: %v, Connections: %d\n", forward.Active, forward.Connections)
    fmt.Printf("  Bytes Sent: %d, Bytes Recv: %d\n", forward.BytesSent, forward.BytesRecv)
}
```

### Getting All Forwards

```go
forwards := manager.GetAllForwards()
fmt.Printf("Active forwards: %d\n", len(forwards))
for _, f := range forwards {
    fmt.Printf("  %s: %s -> %s:%d (Active: %v)\n", 
        f.ID, f.ListenAddr, f.TargetAddr, f.TargetPort, f.Active)
}
```

### Getting Statistics

```go
stats := manager.GetStats()
fmt.Println("Port Forward Statistics:")
for id, s := range stats {
    fmt.Printf("  %s:\n", id)
    fmt.Printf("    Active: %v\n", s.Active)
    fmt.Printf("    Bytes Sent: %d\n", s.BytesSent)
    fmt.Printf("    Bytes Recv: %d\n", s.BytesRecv)
    fmt.Printf("    Connections: %d\n", s.Connections)
    fmt.Printf("    Uptime: %s\n", s.Uptime)
}
```

### Async Operations

```go
ctx := context.Background()

// Async add forward
result := manager.AddForwardAsync(ctx, "web-proxy", "tcp", 8080, "localhost", 80)
select {
case <-result.Ch:
    fmt.Println("Forward added asynchronously!")
case err := <-result.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async remove forward
result2 := manager.RemoveForwardAsync(ctx, "web-proxy")
select {
case <-result2.Ch:
    fmt.Println("Forward removed asynchronously!")
case err := <-result2.ErrCh:
    fmt.Printf("Error: %v\n", err)
}
```

### Get Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Enabled: %v\n", status["enabled"])
fmt.Printf("Active Forwards: %v\n", status["active_forwards"])
fmt.Printf("Active Count: %v\n", status["active_count"])

if status["enabled"] == false {
    if errMsg, ok := status["error"].(string); ok {
        fmt.Printf("Error: %s\n", errMsg)
    }
}
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/spf13/viper` | Configuration management |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `context`, `fmt`, `io`, `net`, `sync`, `time` |

## Error Handling

The library uses Go error handling patterns:

- **Duplicate ID**: `AddForward` checks if ID already exists
- **Not found**: `RemoveForward` returns error if ID not found
- **Listen errors**: Handled in accept loop with retry (1s sleep)
- **Target connection**: Logged as warning, connection dropped
- **Nil checks**: Public methods handle nil receiver gracefully in `GetStatus()`

Example error handling:

```go
err := manager.AddForward(ctx, "web-proxy", "tcp", 8080, "localhost", 80)
if err != nil {
    if strings.Contains(err.Error(), "already exists") {
        return fmt.Errorf("forward ID already in use: %w", err)
    }
    if strings.Contains(err.Error(), "listen") {
        return fmt.Errorf("failed to listen on port: %w", err)
    }
    return fmt.Errorf("failed to add forward: %w", err)
}
```

## Common Pitfalls

### 1. Port Already in Use

**Problem**: `bind: address already in use` error when adding forward

**Solution**: 
- Choose a different local port
- Check if another process is using the port (`lsof -i :port`)
- Wait for socket to release (TIME_WAIT)

### 2. Target Not Reachable

**Problem**: Connections fail with `connection refused` to target

**Solution**:
- Verify target address and port are correct
- Check that target service is running
- Verify network connectivity (ping, telnet)

### 3. Forward ID Already Exists

**Problem**: `port forward with id 'xxx' already exists` error

**Solution**:
- Use unique IDs for each forward
- Check existing forwards: `manager.GetAllForwards()`
- Remove existing forward first: `manager.RemoveForward(id)`

### 4. Context Cancellation

**Problem**: Forward not stopping when expected

**Solution**:
- The forward uses its own context that is cancelled on `RemoveForward`
- Ensure you call `RemoveForward(id)` to stop the forward
- The listener is closed and goroutines exit via context cancellation

### 5. Statistics Not Updating

**Problem**: Bytes sent/recv showing 0

**Solution**:
- Statistics are updated during active connections
- Check that connections are being established
- Use `GetStats()` to see current statistics

### 6. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewPortForwarderManager()` succeeded
- Verify `port_forwarder.enabled` is true

## Advanced Usage

### Multiple Port Forwards

```go
func setupMultipleForwards(manager *port_forwarder.PortForwarderManager) {
    ctx := context.Background()
    
    // Forward web traffic
    manager.AddForward(ctx, "web", "tcp", 8080, "localhost", 80)
    // Forward SSH
    manager.AddForward(ctx, "ssh", "tcp", 2222, "localhost", 22)
    // Forward database
    manager.AddForward(ctx, "db", "tcp", 5432, "db-server", 5432)
    
    fmt.Println("All forwards created")
}
```

### Monitoring Forwards

```go
func monitorForwards(manager *port_forwarder.PortForwarderManager) {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            stats := manager.GetStats()
            fmt.Printf("\n=== Port Forward Statistics ===\n")
            for id, s := range stats {
                fmt.Printf("  %s: Active=%v, Conn=%d, Sent=%d, Recv=%d, Uptime=%s\n",
                    id, s.Active, s.Connections, s.BytesSent, s.BytesRecv, s.Uptime)
            }
        }
    }
}
```

### Dynamic Forward Management

```go
func dynamicForward(manager *port_forwarder.PortForwarderManager, id string, listenPort int, targetAddr string, targetPort int) {
    ctx := context.Background()
    
    // Add forward
    err := manager.AddForward(ctx, id, "tcp", listenPort, targetAddr, targetPort)
    if err != nil {
        fmt.Printf("Failed to add forward: %v\n", err)
        return
    }
    fmt.Printf("Forward %s created\n", id)
    
    // Simulate some time
    time.Sleep(5 * time.Minute)
    
    // Remove forward
    err = manager.RemoveForward(id)
    if err != nil {
        fmt.Printf("Failed to remove forward: %v\n", err)
        return
    }
    fmt.Printf("Forward %s removed\n", id)
}
```

### Connection Counting

```go
func trackConnections(manager *port_forwarder.PortForwarderManager) {
    forwards := manager.GetAllForwards()
    totalConns := int64(0)
    
    for _, f := range forwards {
        totalConns += f.Connections
        fmt.Printf("Forward %s: %d connections\n", f.ID, f.Connections)
    }
    
    fmt.Printf("Total connections across all forwards: %d\n", totalConns)
}
```

## Internal Algorithms

### TCP Connection Handling

```
handleTCPConnection():
    │
    ├── DialTimeout to target (10s)
    ├── WaitGroup.Add(2)
    ├── goroutine: io.Copy(target, client)
    │   └── Update forward.BytesSent += n
    ├── goroutine: io.Copy(client, target)
    │   └── Update forward.BytesRecv += n
    └── WaitGroup.Wait()
```

### Accept Loop with Context

```
acceptTCPConnections():
    │
    └── Loop:
        ├── Select:
        │   ├── ctx.Done() → return
        │   └── default: continue
        ├── listener.Accept()
        ├── If error:
        │   ├── Check ctx.Done() → return
        │   ├── Log warning
        │   ├── Sleep 1s
        │   └── continue
        ├── forward.Connections++
        └── goroutine: handleTCPConnection(ctx, clientConn, forward)
```

### Statistics Collection

```
GetStats():
    │
    ├── RLock
    ├── For each forward:
    │   ├── Build PortForwardStats
    │   ├── Calculate uptime: time.Since(CreatedAt)
    │   └── Store in map
    └── Return stats map
```

### Async Operation Pattern

```
AddForwardAsync(ctx, id, protocol, listenPort, targetAddr, targetPort):
    │
    └── ExecuteAsync(ctx, func):
        └── return AddForward(ctx, id, protocol, listenPort, targetAddr, targetPort)
            └── Returns (struct{}, error)
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.
