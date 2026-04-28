# SSH Tunnel Manager

## Overview

The `SSHTunnelManager` is a Go library for managing SSH tunnels. It allows creating SSH tunnels that forward local ports to remote hosts via an SSH connection, with traffic forwarding, health checks, and async support via a worker pool.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **SSH Tunnel Creation**: Create SSH tunnels with local port forwarding to remote host:port
- **Multiple Tunnels**: Manage multiple tunnels with unique IDs
- **Traffic Forwarding**: Bidirectional TCP forwarding between local and remote
- **Statistics Tracking**: Track bytes sent/received per tunnel
- **Health Check**: HTTP ping to a URL to verify connectivity
- **Background Health Monitoring**: Periodic health checks with configurable interval
- **Context Support**: Cancel tunnels via context
- **Worker Pool**: Async job execution support (4 workers)
- **Status Monitoring**: Get statistics for all active tunnels

## Quick Start

```go
package main

import (
    "context"
    "fmt"
    "stackyrd/pkg/infrastructure"
    "stackyrd/pkg/logger"
    "golang.org/x/crypto/ssh"
)

func main() {
    // Initialize logger
    log := logger.NewLogger()
    
    // Create SSH Tunnel manager (configuration via viper)
    manager, err := ssh_tunnel.NewSSHTunnelManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Create SSH tunnel: local :8080 -> remote localhost:80 via SSH
    auth := ssh.Password("password")
    err = manager.CreateTunnel(ctx, "web-tunnel", 8080, "remote-host", 80, "ssh-user", auth)
    if err != nil {
        panic(err)
    }
    fmt.Println("SSH tunnel created: :8080 -> remote-host:80")
    
    // Keep running...
    select {}
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `SSHTunnelManager` | Main manager with tunnels map, worker pool |
| `Tunnel` | Tunnel with ID, addresses, SSH client, listener, stats |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   SSHTunnelManager                        │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  SSH        │  → SSH client (golang.org/x/crypto/ssh) │
│  │  Tunnels    │  → Local listeners, forwarded connections │
│  └────────────────┘                                      │
│                                                         │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ AsyncResult │                   │
│  │  (4 workers)│      │   Channel   │                   │
│  └─────────────┘      └──────────────┘                   │
│         ▲                      │                            │
│         │                      ▼                            │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  SubmitJob  │      │ ExecuteAsync │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                         │
│  Tunnels: Map of tunnel ID -> Tunnel                  │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewSSHTunnelManager(logger)
    │
    ├── Check viper config: "ssh_tunnel.enabled"
    ├── Initialize tunnels map
    ├── Create WorkerPool(4)
    ├── Start worker pool
    └── Return SSHTunnelManager
```

### 2. Create Tunnel Flow

```
CreateTunnel(ctx, id, localPort, remoteHost, remotePort, user, auth)
    │
    ├── Lock mutex
    ├── Check if ID already exists
    ├── Build ssh.ClientConfig: User, Auth, HostKeyCallback, Timeout
    ├── ssh.Dial("tcp", "remoteHost:remotePort", sshConfig)
    ├── ListenTCP on localPort
    ├── Create Tunnel struct
    ├── Store tunnel in tunnels map
    ├── Start goroutine: handleTunnel(ctx, tunnel)
    └── Return nil
```

### 3. Handle Tunnel Flow

```
handleTunnel(ctx, tunnel)
    │
    └── Loop:
        ├── Select on ctx.Done() → return
        └── default:
            ├── listener.Accept()
            ├── If error: check ctx.Done(), log warning, continue
            └── goroutine: forward(ctx, conn, tunnel)
```

### 4. Forward Flow

```
forward(ctx, localConn, tunnel)
    │
    ├── defer localConn.Close()
    ├── tunnel.sshClient.Dial("tcp", "remoteHost:remotePort")
    ├── If error: log warning, return
    ├── defer remoteConn.Close()
    ├── WaitGroup.Add(2)
    ├── goroutine: io.Copy(remote, local) → update tunnel.BytesSent
    ├── goroutine: io.Copy(local, remote) → update tunnel.BytesRecv
    └── WaitGroup.Wait()
```

### 5. Get Status Flow

```
GetStatus()
    │
    ├── If nil: return connected=false
    ├── Return enabled, tunnels count, pool_active
    └── Note: Does not test SSH connection (unlike others)
```

### 6. Ping URL Flow

```
PingURL(url)
    │
    ├── http.Client with 5s timeout
    ├── client.Get(url)
    ├── Return status code or error
    └── Note: Not used in tunnel forwarding, separate utility
```

### 7. Start Health Check Flow

```
StartHealthCheck(ctx, url, interval)
    │
    └── Goroutine:
        ├── ticker := time.NewTicker(interval)
        └── Loop:
            ├── Select ctx.Done() → return
            └── ticker.C:
                ├── PingURL(url)
                └── Log result (info or warn)
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `ssh_tunnel.enabled` | bool | false | Enable/disable SSH Tunnel manager |

### Environment Variables

The SSH Tunnel manager does not directly use environment variables. Configuration is read through Viper, which can load from:
- Configuration files (JSON, YAML, TOML)
- Environment variables (if Viper is configured for it)

You can enable the manager via:
```go
viper.Set("ssh_tunnel.enabled", true)
```

## Usage Examples

### Creating an SSH Tunnel

```go
ctx := context.Background()

// Create SSH tunnel: local :8080 -> remote localhost:80 via SSH
auth := ssh.Password("password")
err := manager.CreateTunnel(ctx, "web-tunnel", 8080, "remote-host", 80, "ssh-user", auth)
if err != nil {
    panic(err)
}
fmt.Println("SSH tunnel created")
```

### Using Password Authentication

```go
import "golang.org/x/crypto/ssh"

auth := ssh.Password("your-password")
err := manager.CreateTunnel(ctx, "db-tunnel", 5432, "db-host", 5432, "user", auth)
if err != nil {
    panic(err)
}
```

### Using Public Key Authentication

```go
import (
    "golang.org/x/crypto/ssh"
    "io/ioutil"
)

// Load private key
keyData, err := ioutil.ReadFile("/path/to/private/key")
if err != nil {
    panic(err)
}

signer, err := ssh.ParsePrivateKey(keyData)
if err != nil {
    panic(err)
}

auth := ssh.PublicKeys(signer)
err = manager.CreateTunnel(ctx, "secure-tunnel", 2222, "remote-host", 22, "user", auth)
if err != nil {
    panic(err)
}
```

### Getting Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Enabled: %v\n", status["enabled"])
fmt.Printf("Active Tunnels: %v\n", status["tunnels"])

if status["enabled"] == false {
    if errMsg, ok := status["error"].(string); ok {
        fmt.Printf("Error: %s\n", errMsg)
    }
}
```

### Closing All Tunnels

```go
// Close manager (closes all tunnels)
err := manager.Close()
if err != nil {
    fmt.Printf("Error closing: %v\n", err)
}
fmt.Println("All tunnels closed")
```

### Starting Health Check

```go
ctx := context.Background()

// Start health check on a URL every 30 seconds
manager.StartHealthCheck(ctx, "http://remote-host:8080/health", 30*time.Second)

// Health check runs in background goroutine
// It will log results periodically
```

### Ping URL Utility

```go
// Ping a URL to check reachability
statusCode, err := manager.PingURL("http://example.com")
if err != nil {
    fmt.Printf("Ping failed: %v\n", err)
} else {
    fmt.Printf("Ping successful: HTTP %d\n", statusCode)
}
```

## Dependencies

| Dependency | Role |
|------------|------|
| `golang.org/x/crypto/ssh` | SSH client library |
| `github.com/spf13/viper` | Configuration management |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `context`, `fmt`, `io`, `net`, `net/http`, `sync`, `time` |

## Error Handling

The library uses Go error handling patterns:

- **Duplicate ID**: `CreateTunnel` checks if ID already exists
- **SSH connection**: Error from `ssh.Dial`
- **Listen errors**: Handled in accept loop with retry
- **Forward errors**: Logged as warning, connection dropped
- **Nil checks**: Public methods handle nil receiver gracefully in `GetStatus()`

Example error handling:

```go
err := manager.CreateTunnel(ctx, "web-tunnel", 8080, "remote-host", 80, "user", auth)
if err != nil {
    if strings.Contains(err.Error(), "already exists") {
        return fmt.Errorf("tunnel ID already in use: %w", err)
    }
    if strings.Contains(err.Error(), "ssh") {
        return fmt.Errorf("SSH connection failed: %w", err)
    }
    if strings.Contains(err.Error(), "listen") {
        return fmt.Errorf("failed to listen on local port: %w", err)
    }
    return fmt.Errorf("failed to create tunnel: %w", err)
}
```

## Common Pitfalls

### 1. SSH Authentication Failed

**Problem**: `ssh: handshake failed` or `unable to authenticate`

**Solution**:
- Verify username and authentication method (password or key)
- For password: ensure correct password
- For public key: ensure private key path is correct and key is accepted by server

### 2. Remote Host Not Reachable

**Problem**: `dial tcp: connection refused` or `i/o timeout`

**Solution**:
- Verify remote host address and SSH port (usually 22)
- Check network connectivity (ping, telnet to SSH port)
- Ensure SSH server is running on remote host

### 3. Local Port Already in Use

**Problem**: `bind: address already in use` when creating tunnel

**Solution**:
- Choose a different local port
- Check if another process is using the port (`lsof -i :port`)
- Wait for socket to release (TIME_WAIT)

### 4. Tunnel ID Already Exists

**Problem**: `tunnel with id xxx already exists` error

**Solution**:
- Use unique IDs for each tunnel
- Check existing tunnels (not directly exposed, but can be tracked)
- Close existing tunnel before recreating (manager.Close() closes all)

### 5. Context Cancellation

**Problem**: Tunnel not stopping when expected

**Solution**:
- The tunnel uses the context passed to `CreateTunnel`
- When context is cancelled, the tunnel's accept loop will exit
- However, active forwarded connections may not be immediately terminated

### 6. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewSSHTunnelManager()` succeeded
- Verify `ssh_tunnel.enabled` is true

## Advanced Usage

### Multiple SSH Tunnels

```go
func setupMultipleTunnels(manager *ssh_tunnel.SSHTunnelManager) {
    ctx := context.Background()
    auth := ssh.Password("password")
    
    // Web tunnel
    manager.CreateTunnel(ctx, "web", 8080, "web-host", 80, "user", auth)
    // SSH tunnel
    manager.CreateTunnel(ctx, "ssh", 2222, "ssh-host", 22, "user", auth)
    // Database tunnel
    manager.CreateTunnel(ctx, "db", 5432, "db-host", 5432, "user", auth)
    
    fmt.Println("All tunnels created")
}
```

### Health Check with Custom Action

```go
func healthCheckWithAction(manager *ssh_tunnel.SSHTunnelManager, url string) {
    ctx := context.Background()
    
    // Start health check with callback-like logging
    go manager.StartHealthCheck(ctx, url, 1*time.Minute)
    
    // In a real application, you might take action on failure
    // The health check only logs; you could extend it.
}
```

### Tunnel Statistics (Simplified)

```go
func printTunnelStats(manager *ssh_tunnel.SSHTunnelManager) {
    // Note: The current implementation does not expose tunnel stats publicly.
    // But the Tunnel struct tracks BytesSent, BytesRecv.
    // You could extend the manager to provide a GetTunnelStats(id) method.
    fmt.Println("Tunnel statistics are tracked internally.")
}
```

### SSH Tunnel with Key and Passphrase

```go
import (
    "golang.org/x/crypto/ssh"
    "golang.org/x/crypto/ssh/knownhosts"
    "io/ioutil"
    "os"
)

func createTunnelWithKeyAndPassphrase(manager *ssh_tunnel.SSHTunnelManager) {
    ctx := context.Background()
    
    // Load private key with passphrase
    keyData, err := ioutil.ReadFile("/path/to/private/key")
    if err != nil {
        panic(err)
    }
    
    // If key has passphrase, use ssh.ParsePrivateKeyWithPassphrase
    // For simplicity, assuming no passphrase:
    signer, err := ssh.ParsePrivateKey(keyData)
    if err != nil {
        panic(err)
    }
    
    auth := ssh.PublicKeys(signer)
    
    // Optionally set up host key verification
    hostKeyCallback, err := knownhosts.New("/path/to/known_hosts")
    if err != nil {
        panic(err)
    }
    
    // Then create tunnel with appropriate ssh.ClientConfig
    // Note: The current API only accepts ssh.AuthMethod, not full config.
    // You might need to extend CreateTunnel to accept ssh.ClientConfig.
}
```

## Internal Algorithms

### SSH Tunnel Forwarding

```
forward():
    │
    ├── sshClient.Dial("tcp", remoteAddr)
    ├── WaitGroup.Add(2)
    ├── goroutine: io.Copy(remote, local) → update BytesSent
    ├── goroutine: io.Copy(local, remote) → update BytesRecv
    └── WaitGroup.Wait()
```

### Accept Loop with Context

```
handleTunnel():
    │
    └── Loop:
        ├── Select:
        │   ├── ctx.Done() → return
        │   └── default: continue
        ├── listener.Accept()
        ├── If error:
        │   ├── Check ctx.Done() → return
        │   ├── Log warning
        │   └── continue
        └── goroutine: forward(ctx, conn, tunnel)
```

### Health Check Loop

```
StartHealthCheck():
    │
    └── Goroutine:
        ├── ticker := time.NewTicker(interval)
        └── Loop:
            ├── Select ctx.Done() → return
            └── ticker.C:
                ├── PingURL(url)
                └── Log result
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.
