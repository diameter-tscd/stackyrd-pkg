# Mikrotik RouterOS Manager

## Overview

The `MikrotikManager` is a Go library for managing MikroTik RouterOS devices. It provides system resource monitoring, network interface management, firewall/NAT rule management, DHCP lease monitoring, wireless client tracking, routing table access, traffic queuing, and proxy server capabilities (SOCKS5 and HTTP). The library uses the `go-routeros` client for direct RouterOS API communication.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **System Monitoring**: Get system resources (CPU, memory, disk, uptime, load)
- **Interface Management**: List network interfaces with statistics (RX/TX bytes, errors, drops)
- **IP Address Management**: View assigned IP addresses and networks
- **Firewall Rules**: List and monitor firewall filter rules with statistics
- **NAT Rules**: Manage NAT rules (src/dst address, port forwarding)
- **DHCP Leases**: Monitor active DHCP server leases
- **Wireless Clients**: Track connected wireless clients (signal strength, rates)
- **Routing Table**: Access routing table entries (destinations, gateways)
- **Traffic Queuing**: Manage simple queues for traffic shaping
- **PPP Connections**: Monitor active PPP connections
- **Proxy Servers**: Run SOCKS5 and HTTP/HTTPS proxy through MikroTik
- **Raw Command Execution**: Execute arbitrary RouterOS commands
- **Reboot**: Remotely reboot the router
- **Worker Pool**: Async job execution support (8 workers)
- **Status Monitoring**: Get connection status and system health

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
    
    // Create Mikrotik manager (configuration via viper)
    manager, err := mikrotik.NewMikrotikManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Get system resources
    sys, err := manager.GetSystemResources(ctx)
    if err != nil {
        panic(err)
    }
    fmt.Printf("RouterOS Version: %s\n", sys.Version)
    fmt.Printf("Board: %s\n", sys.BoardName)
    fmt.Printf("CPU Load: %.2f%%\n", sys.CPULoad)
    fmt.Printf("Free Memory: %.2f%%\n", sys.FreeMemoryPercent)
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `MikrotikManager` | Main manager with RouterOS client, proxy servers, connection tracking |
| `MikrotikSystem` | System info (version, CPU, memory, disk, uptime) |
| `MikrotikInterface` | Network interface with RX/TX stats, errors, drops |
| `MikrotikIPAddress` | IP address assignment (address, network, interface) |
| `MikrotikFirewallRule` | Firewall filter rule (chain, action, addresses, stats) |
| `MikrotikNATRule` | NAT rule (chain, action, to-address, stats) |
| `MikrotikDHCPLease` | DHCP lease (address, MAC, hostname, status) |
| `MikrotikWirelessClient` | Wireless client (MAC, SSID, signal, rates) |
| `MikrotikRoute` | Routing table entry (destination, gateway, distance) |
| `MikrotikQueue` | Traffic queue (target, limits, dropped bytes) |
| `MikrotikPPPActive` | Active PPP connection (service, caller ID, bytes) |
| `SOCKS5Server` | SOCKS5 proxy server implementation |
| `HTTPProxyServer` | HTTP/HTTPS proxy server implementation |
| `ProxyConnection` | Active proxy connection tracking |

### Concurrency Model

```
┌─────────────────────────────────────────────────────┐
│                   MikrotikManager                       │
├─────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  routeros    │  → RouterOS API client                  │
│  │  .Client      │  → TLS or plain TCP connection           │
│  └────────────────┘                                      │
│                                                          │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ AsyncResult │                   │
│  │  (8 workers)│      │   Channel   │                   │
│  └─────────────┘      └──────────────┘                   │
│         ▲                      │                            │
│         │                      ▼                            │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  SubmitJob  │      │ ExecuteAsync │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                          │
│  ┌──────────────────────────────────────┐            │
│  │  Proxy Servers                              │            │
│  │  - SOCKS5Server (port P)                     │            │
│  │  - HTTPProxyServer (port P+1)                 │            │
│  └──────────────────────────────────────┘            │
│                                                          │
│  Host: MikroTik IP/hostname                         │
│  Port: Typically 8728 (API) or 8729 (API-SSL)      │
│  Username/Password: RouterOS credentials              │
└─────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewMikrotikManager(logger)
    │
    ├── Check viper config: "mikrotik.enabled"
    ├── Get host: "mikrotik.host"
    ├── Get port: "mikrotik.port" (default 8728)
    ├── Get username: "mikrotik.username"
    ├── Get password: "mikrotik.password"
    ├── Get useTLS: "mikrotik.tls"
    │
    ├── Connect via routeros.Dial or routeros.DialTLS
    │
    ├── Create WorkerPool(8)
    ├── Start worker pool
    │
    ├── Test connection: testConnection()
    │   └── Run "/system/resource/print"
    │
    ├── If proxy enabled: StartProxyServers()
    │   ├── Start SOCKS5 server on proxyPort
    │   └── Start HTTP proxy server on proxyPort+1
    │
    └── Return MikrotikManager
```

### 2. Get System Resources Flow

```
GetSystemResources(ctx)
    │
    ├── Run "/system/resource/print"
    ├── Parse reply.Re[0].Map fields
    ├── Convert totals and free memory, HDD, CPU load
    ├── Calculate free memory percent
    └── Return *MikrotikSystem
```

### 3. Get Interfaces Flow

```
GetInterfaces(ctx)
    │
    ├── Run "/interface/print" with "stats"
    ├── Iterate reply.Re
    ├── Parse each interface's stats (RX/TX bytes, packets, errors, drops)
    └── Return []MikrotikInterface
```

### 4. Proxy Server Flow

```
StartProxyServers()
    │
    ├── Listen on TCP :proxyPort (SOCKS5)
    ├── Listen on TCP :proxyPort+1 (HTTP)
    │
    ├── SOCKS5: handleConnection()
    │   ├── SOCKS5 handshake
    │   ├── Parse CONNECT request
    │   ├── Dial through MikroTik (manager.Dial)
    │   └── Bidirectional copy
    │
    └── HTTP: ServeHTTP()
        ├── If CONNECT: tunnel via manager.Dial
        └── Else: use http.Transport with manager.Dial
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `mikrotik.enabled` | bool | false | Enable/disable Mikrotik manager |
| `mikrotik.host` | string | "" | MikroTik router IP/hostname |
| `mikrotik.port` | int | 8728 | RouterOS API port (8728 plain, 8729 TLS) |
| `mikrotik.username` | string | "" | RouterOS username |
| `mikrotik.password` | string | "" | RouterOS password |
| `mikrotik.tls` | bool | false | Use TLS connection |
| `mikrotik.proxy_enabled` | bool | false | Enable SOCKS5/HTTP proxy servers |
| `mikrotik.proxy_port` | int | 1080 | Starting port for proxy servers |
| `mikrotik.proxy_auth` | bool | false | Enable proxy authentication |
| `mikrotik.proxy_username` | string | "" | Proxy auth username |
| `mikrotik.proxy_password` | string | "" | Proxy auth password |

## Usage Examples

### Get System Resources

```go
ctx := context.Background()

sys, err := manager.GetSystemResources(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Version: %s\n", sys.Version)
fmt.Printf("Board: %s\n", sys.BoardName)
fmt.Printf("CPU: %s (%d cores @ %d MHz)\n", sys.CPU, sys.CPUCount, sys.CPUFrequency)
fmt.Printf("Memory: %d MB total, %d MB free (%.2f%%)\n", 
    sys.TotalMemory/1024/1024, sys.FreeMemory/1024/1024, sys.FreeMemoryPercent)
fmt.Printf("Uptime: %s\n", sys.Uptime)
```

### List Network Interfaces

```go
ctx := context.Background()

interfaces, err := manager.GetInterfaces(ctx)
if err != nil {
    panic(err)
}

for _, iface := range interfaces {
    fmt.Printf("  %s: %s (Running: %v, Enabled: %v)\n",
        iface.Name, iface.Type, iface.Running, iface.Enabled)
    fmt.Printf("    MAC: %s\n", iface.MacAddress)
    fmt.Printf("    RX: %d bytes, TX: %d bytes\n", iface.RxByte, iface.TxByte)
    fmt.Printf("    Errors - RX: %d, TX: %d\n", iface.RxError, iface.TxError)
    fmt.Printf("    Drops - RX: %d, TX: %d\n", iface.RxDrop, iface.TxDrop)
}
```

### Get IP Addresses

```go
ctx := context.Background()

addresses, err := manager.GetIPAddresses(ctx)
if err != nil {
    panic(err)
}

for _, addr := range addresses {
    fmt.Printf("  %s on %s (Network: %s, Disabled: %v, Dynamic: %v)\n",
        addr.Address, addr.Interface, addr.Network, addr.Disabled, addr.Dynamic)
}
```

### Get Firewall Rules

```go
ctx := context.Background()

rules, err := manager.GetFirewallRules(ctx)
if err != nil {
    panic(err)
}

for _, rule := range rules {
    fmt.Printf("  Chain: %s, Action: %s\n", rule.Chain, rule.Action)
    fmt.Printf("    Src: %s, Dst: %s, Protocol: %s\n", rule.SrcAddr, rule.DstAddr, rule.Protocol)
    fmt.Printf("    Bytes: %d, Packets: %d\n", rule.Bytes, rule.Packets)
}
```

### Get NAT Rules

```go
ctx := context.Background()

rules, err := manager.GetNATRules(ctx)
if err != nil {
    panic(err)
}

for _, rule := range rules {
    fmt.Printf("  Chain: %s, Action: %s\n", rule.Chain, rule.Action)
    fmt.Printf("    Src: %s, Dst: %s -> To: %s:%s\n",
        rule.SrcAddr, rule.DstAddr, rule.ToAddress, rule.ToPorts)
}
```

### Get DHCP Leases

```go
ctx := context.Background()

leases, err := manager.GetDHCPLeases(ctx)
if err != nil {
    panic(err)
}

for _, lease := range leases {
    fmt.Printf("  %s: %s (MAC: %s, Hostname: %s)\n",
        lease.Address, lease.Status, lease.MacAddress, lease.Hostname)
    fmt.Printf("    Expires: %s, Blocked: %v\n", lease.ExpiresAfter, lease.Blocked)
}
```

### Get Wireless Clients

```go
ctx := context.Background()

clients, err := manager.GetWirelessClients(ctx)
if err != nil {
    panic(err)
}

for _, client := range clients {
    fmt.Printf("  %s on %s (SSID: %s)\n", client.MacAddress, client.Interface, client.SSID)
    fmt.Printf("    Signal: %d dBm, TX: %s, RX: %s\n", client.Signal, client.TxRate, client.RxRate)
    fmt.Printf("    Bytes: %d, Uptime: %s\n", client.Bytes, client.Uptime)
}
```

### Get Routing Table

```go
ctx := context.Background()

routes, err := manager.GetRoutes(ctx)
if err != nil {
    panic(err)
}

for _, route := range routes {
    fmt.Printf("  %s via %s (Distance: %d, Active: %v)\n",
        route.Destination, route.Gateway, route.Distance, route.Active)
}
```

### Get Queues

```go
ctx := context.Background()

queues, err := manager.GetQueues(ctx)
if err != nil {
    panic(err)
}

for _, queue := range queues {
    fmt.Printf("  %s: Target %s, Max Limit: %s\n", queue.Name, queue.Target, queue.MaxLimit)
    fmt.Printf("    Bytes: %d, Dropped: %d\n", queue.Bytes, queue.Dropped)
}
```

### Get Active PPP Connections

```go
ctx := context.Background()

connections, err := manager.GetPPPActive(ctx)
if err != nil {
    panic(err)
}

for _, conn := range connections {
    fmt.Printf("  %s: %s (Service: %s, Caller: %s)\n",
        conn.Name, conn.Address, conn.Service, conn.CallerID)
    fmt.Printf("    Bytes In: %d, Out: %d\n", conn.BytesIn, conn.BytesOut)
    fmt.Printf("    Uptime: %s\n", conn.Uptime)
}
```

### Reboot Router

```go
ctx := context.Background()

err := manager.Reboot(ctx)
if err != nil {
    panic(err)
}
fmt.Println("Router reboot initiated")
```

### Execute Raw Command

```go
ctx := context.Background()

reply, err := manager.ExecuteCommand(ctx, "/system/identity/print")
if err != nil {
    panic(err)
}
for _, re := range reply.Re {
    fmt.Printf("Identity: %s\n", re.Map["name"])
}
```

### Using Proxy Servers

```go
// After enabling proxy in config and starting manager:
// SOCKS5 proxy available at :proxyPort
// HTTP proxy available at :proxyPort+1

// Example: configure browser to use HTTP proxy at http://router-ip:proxyPort+1
```

### Async Operations

```go
ctx := context.Background()

// Async get system resources
sysChan := manager.GetSystemResourcesAsync(ctx)
sys, err := sysChan.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Version: %s\n", sys.Version)

// Async get interfaces
ifacesChan := manager.GetInterfacesAsync(ctx)
interfaces, err := ifacesChan.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Interfaces: %d\n", len(interfaces))
```

### Getting Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Host: %s, Port: %v\n", status["host"], status["port"])
fmt.Printf("Version: %s\n", status["version"])
fmt.Printf("CPU Load: %.2f%%\n", status["cpu_load"])
fmt.Printf("Free Memory: %.2f%%\n", status["free_memory_percent"])
fmt.Printf("Proxy Enabled: %v\n", status["proxy_enabled"])
fmt.Printf("Proxy Port: %v\n", status["proxy_port"])

if status["connected"] == false {
    if errMsg, ok := status["error"].(string); ok {
        fmt.Printf("Error: %s\n", errMsg)
    }
}
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/go-routeros/routeros` | RouterOS API client |
| `github.com/spf13/viper` | Configuration management |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `crypto/tls`, `net`, `net/http`, `context`, `encoding/json`, `strconv`, `sync`, `time`, `io` |

## Error Handling

The library uses Go error handling patterns:

- **Authentication failures**: Invalid username/password
- **Connection errors**: Router not reachable
- **API errors**: RouterOS command failures
- **Nil checks**: Public methods check manager state

Example error handling:

```go
sys, err := manager.GetSystemResources(ctx)
if err != nil {
    if strings.Contains(err.Error(), "connection refused") {
        return fmt.Errorf("router not reachable: %w", err)
    }
    if strings.Contains(err.Error(), "invalid user") {
        return fmt.Errorf("invalid credentials: %w", err)
    }
    return fmt.Errorf("API request failed: %w", err)
}
```

## Common Pitfalls

### 1. Manager Not Enabled

**Problem**: `mikrotik.enabled` is false

**Solution**:
- Set `mikrotik.enabled = true` in configuration
- Check viper configuration is loaded

### 2. Connection Failed

**Problem**: Cannot connect to MikroTik

**Solution**:
- Verify `mikrotik.host` is correct
- Verify `mikrotik.port` (default 8728 for plain, 8729 for TLS)
- Ensure network can reach the router
- Check firewall settings on router and client
- For TLS: set `mikrotik.tls = true`

### 3. Invalid Credentials

**Problem**: Authentication failed

**Solution**:
- Verify `mikrotik.username` and `mikrotik.password`
- Check user has API access (in RouterOS: `/user/set [find name=username] group=full`)

### 4. Proxy Not Working

**Problem**: Proxy servers not starting

**Solution**:
- Enable proxy: `mikrotik.proxy_enabled = true`
- Check `mikrotik.proxy_port` is not in use
- Check proxy auth settings if enabled

### 5. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewMikrotikManager()` succeeded
- Verify `mikrotik.enabled` is true

## Advanced Usage

### Health Check

```go
func healthCheck(manager *mikrotik.MikrotikManager) error {
    ctx := context.Background()
    sys, err := manager.GetSystemResources(ctx)
    if err != nil {
        return fmt.Errorf("not connected to MikroTik: %w", err)
    }
    fmt.Printf("MikroTik is healthy (Version: %s)\n", sys.Version)
    return nil
}
```

### Batch Interface Stats

```go
func batchGetInterfaceStats(manager *mikrotik.MikrotikManager) {
    ctx := context.Background()
    interfaces, err := manager.GetInterfaces(ctx)
    if err != nil {
        panic(err)
    }
    totalRx := int64(0)
    totalTx := int64(0)
    for _, iface := range interfaces {
        totalRx += iface.RxByte
        totalTx += iface.TxByte
    }
    fmt.Printf("Total RX: %d bytes, TX: %d bytes\n", totalRx, totalTx)
}
```

## Internal Algorithms

### RouterOS API Communication

```
GetSystemResources():
    │
    ├── Client.Run("/system/resource/print")
    ├── Parse reply.Re[0].Map
    ├── Convert string values to appropriate types
    └── Return MikrotikSystem struct
```

### Proxy Connection Handling

```
SOCKS5 handleConnection():
    │
    ├── Read SOCKS5 handshake
    ├── Send method selection
    ├── Read CONNECT request
    ├── Parse destination address (IPv4, domain, or IPv6)
    ├── Dial through MikroTik manager.Dial()
    ├── Send success response
    └── Bidirectional io.Copy
```

### Connection Test

```
testConnection():
    │
    ├── Run "/system/resource/print"
    ├── Check for error
    └── Return error if failed
```

## API Endpoints (RouterOS API)

The library uses RouterOS API via TCP connection. Common commands used:

| Command | Description |
|---------|-------------|
| `/system/resource/print` | Get system resources |
| `/interface/print` | List interfaces with stats |
| `/ip/address/print` | List IP addresses |
| `/ip/firewall/filter/print` | List firewall rules |
| `/ip/firewall/nat/print` | List NAT rules |
| `/ip/dhcp-server/lease/print` | List DHCP leases |
| `/interface/wireless/registration-table/print` | List wireless clients |
| `/ip/route/print` | List routing table |
| `/queue/simple/print` | List simple queues |
| `/ppp/active/print` | List active PPP connections |
| `/system/reboot` | Reboot router |

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.