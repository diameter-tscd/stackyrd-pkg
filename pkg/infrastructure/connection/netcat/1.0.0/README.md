# Netcat Manager

## Overview

The `NetcatManager` is a Go library that provides netcat-like functionality for TCP and UDP networking. It supports connecting to hosts, sending data, listening on ports, port scanning, banner grabbing, file transfers, and TCP proxying, all with async support via a worker pool.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **TCP Connect**: Connect to TCP hosts with timeout
- **UDP Connect**: Connect to UDP hosts
- **TCP Send/Receive**: Send data and receive response over TCP
- **UDP Send/Receive**: Send data and receive response over UDP
- **TCP Listen**: Start TCP listeners with custom handlers
- **UDP Listen**: Start UDP listeners with custom handlers
- **Port Scanning**: Scan single ports or port ranges (with concurrency)
- **Banner Grabbing**: Grab service banners from TCP ports
- **File Transfer**: Transfer data via TCP (send or receive)
- **TCP Proxy**: Create TCP proxy from local port to remote host
- **Listener Management**: Track and stop active listeners
- **Async Operations**: Async port scan, banner grab, send operations via worker pool
- **Worker Pool**: Async job execution support (8 workers)

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
    
    // Create Netcat manager (configuration via viper)
    manager, err := netcat.NewNetcatManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Send TCP data and receive response
    response, err := manager.SendStringTCP(ctx, "example.com", 80, "GET / HTTP/1.0\r\n\r\n")
    if err != nil {
        panic(err)
    }
    fmt.Printf("Response: %s\n", response)
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `NetcatManager` | Main manager with HTTP client, listeners, UDP connections, worker pool |
| `NetcatConnection` | Connection info (ID, local, remote, protocol, opened at) |
| `NetcatListener` | Listener info (ID, address, protocol, started, active) |
| `NetcatScanResult` | Port scan result (host, port, open, latency, service, error) |
| `NetcatTransferResult` | Transfer result (bytes sent/received, duration, source/dest) |
| `NetcatBannerGrabResult` | Banner grab result (host, port, banner, error) |

### Concurrency Model

```
┌──────────────────────────────────────────────────────────────────────┐
│                   NetcatManager                              │
├──────────────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                          │
│  │  TCP/UDP   │  → Direct net.Conn / net.UDPConn          │
│  │  Connections│  → Listeners, scanners, proxy            │
│  └────────────────┘                                          │
│                                                             │
│  ┌─────────────┐      ┌──────────────┐                     │
│  │  WorkerPool │◄─────│ AsyncResult │                     │
│  │  (8 workers)│      │   Channel   │                     │
│  └─────────────┘      └──────────────┘                     │
│         ▲                          │                                │
│         │                          ▼                                │
│  ┌─────────────┐      ┌──────────────┐                    │
│  │  SubmitJob  │      │ ExecuteAsync │                    │
│  └─────────────┘      └──────────────┘                    │
│                                                             │
│  DefaultHost: Default host for operations (viper)               │
│  DefaultPort: Default port for operations (viper)               │
│  Timeout: Connection timeout (default: 10s)                      │
└──────────────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewNetcatManager(logger)
    │
    ├── Check viper config: "netcat.enabled"
    ├── Get defaultHost: "netcat.default_host"
    ├── Get defaultPort: "netcat.default_port"
    ├── Get timeoutSec: "netcat.timeout_sec" (default: 10)
    │
    ├── Create retryablehttp.Client:
    │   ├── RetryMax = 1
    │   ├── RetryWaitMin = 1s
    │   ├── RetryWaitMax = 3s
    │   └── HTTPClient.Timeout = timeout
    │
    ├── Initialize listeners map, udpConns map
    ├── Create WorkerPool(8)
    ├── Start worker pool
    └── Return NetcatManager
```

### 2. TCP Connect Flow

```
ConnectTCP(host, port)
    │
    ├── addr = fmt.Sprintf("%s:%d", host, port)
    ├── net.DialTimeout("tcp", addr, Timeout)
    └── Return net.Conn
```

### 3. UDP Connect Flow

```
ConnectUDP(host, port)
    │
    ├── addr = fmt.Sprintf("%s:%d", host, port)
    ├── net.ResolveUDPAddr("udp", addr)
    ├── net.DialUDP("udp", nil, udpAddr)
    └── Return *net.UDPConn
```

### 4. Send TCP Flow

```
SendTCP(ctx, host, port, data)
    │
    ├── ConnectTCP(host, port)
    ├── Set deadline: time.Now().Add(Timeout)
    ├── conn.Write(data)
    ├── Read response into buffer (65536 bytes)
    └── Return []byte
```

### 5. Listen TCP Flow

```
ListenTCP(port, handler)
    │
    ├── net.Listen("tcp", ":port")
    ├── Store listener in listeners map with ID "tcp-{port}"
    ├── Start goroutine:
    │   └── Loop: Accept connections
    │       ├── If handler: go handler(conn)
    │       └── Else: go defaultHandler(conn) (echo)
    └── Return NetcatListener
```

### 6. Port Scan Flow

```
ScanPort(ctx, host, port, protocol)
    │
    ├── If UDP: ConnectUDP, send probe, read with timeout
    ├── If TCP: DialTimeout
    ├── Measure latency
    └── Return NetcatScanResult
```

### 7. Port Range Scan Flow

```
ScanPortRange(ctx, host, startPort, endPort, protocol)
    │
    ├── Use semaphore (100) for concurrency
    ├── For each port:
    │   └── goroutine: ScanPort(ctx, host, port, protocol)
    ├── Wait for all goroutines
    └── Return []NetcatScanResult
```

### 8. Banner Grab Flow

```
BannerGrab(ctx, host, port)
    │
    ├── ConnectTCP(host, port)
    ├── Set deadline
    ├── Send probes: "", "HEAD / HTTP/1.0\r\n\r\n", "\r\n", "\n"
    ├── Read response
    └── Return banner or error
```

### 9. TCP Transfer Flow

```
TransferTCP(ctx, host, port, reader)
    │
    ├── ConnectTCP(host, port)
    ├── Start timer
    ├── io.Copy(conn, reader) → bytesSent
    ├── Record duration
    └── Return NetcatTransferResult
```

### 10. TCP Proxy Flow

```
ProxyTCP(localPort, remoteHost, remotePort)
    │
    ├── ListenTCP(localPort, nil)
    ├── Store listener with ID "proxy-{localPort}-to-{remoteHost}-{remotePort}"
    ├── Start goroutine:
    │   └── Loop: Accept client connections
    │       └── goroutine: 
    │           ├── DialTimeout to remote
    │           ├── Bidirectional copy (io.Copy)
    │           └── Wait for both directions
    └── Return NetcatListener
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `netcat.enabled` | bool | false | Enable/disable Netcat manager |
| `netcat.default_host` | string | "" | Default host for operations |
| `netcat.default_port` | int | 0 | Default port for operations |
| `netcat.timeout_sec` | int | 10 | Connection timeout in seconds |

### Environment Variables

The Netcat manager does not directly use environment variables. Configuration is read through Viper, which can load from:
- Configuration files (JSON, YAML, TOML)
- Environment variables (if Viper is configured for it)

You can set Netcat parameters via:
```go
viper.Set("netcat.enabled", true)
viper.Set("netcat.default_host", "localhost")
viper.Set("netcat.default_port", 8080)
viper.Set("netcat.timeout_sec", 15)
```

## Usage Examples

### TCP Connect and Send

```go
ctx := context.Background()

// Connect to TCP server
conn, err := manager.ConnectTCP("example.com", 80)
if err != nil {
    panic(err)
}
defer conn.Close()

// Send data
_, err = conn.Write([]byte("GET / HTTP/1.0\r\n\r\n"))
if err != nil {
    panic(err)
}

// Read response
buf := make([]byte, 4096)
n, err := conn.Read(buf)
if err != nil {
    panic(err)
}
fmt.Printf("Response: %s\n", string(buf[:n]))
```

### UDP Connect and Send

```go
ctx := context.Background()

// Connect to UDP server
conn, err := manager.ConnectUDP("localhost", 53)
if err != nil {
    panic(err)
}
defer conn.Close()

// Send data
_, err = conn.Write([]byte("DNS query"))
if err != nil {
    panic(err)
}

// Read response
buf := make([]byte, 4096)
n, _, err := conn.ReadFromUDP(buf)
if err != nil {
    panic(err)
}
fmt.Printf("Response: %s\n", string(buf[:n]))
```

### TCP Listen

```go
// Start TCP listener on port 1234
listener, err := manager.ListenTCP(1234, func(conn net.Conn) {
    defer conn.Close()
    fmt.Println("Connection from:", conn.RemoteAddr())
    conn.Write([]byte("Hello from server!\n"))
})
if err != nil {
    panic(err)
}
fmt.Printf("Listening on %s\n", listener.Address)

// Keep running...
time.Sleep(1 * time.Minute)

// Stop listener
manager.StopListener("tcp-1234")
```

### UDP Listen

```go
// Start UDP listener on port 1234
listener, err := manager.ListenUDP(1234, func(data []byte, addr *net.UDPAddr, conn *net.UDPConn) {
    fmt.Printf("Received from %s: %s\n", addr.String(), string(data))
    conn.WriteToUDP([]byte("Reply"), addr)
})
if err != nil {
    panic(err)
}
fmt.Printf("Listening on UDP %s\n", listener.Address)

// Keep running...
time.Sleep(1 * time.Minute)

// Stop listener
manager.StopListener("udp-1234")
```

### Port Scan

```go
ctx := context.Background()

// Scan single port
result := manager.ScanPort(ctx, "localhost", 80, "tcp")
fmt.Printf("Port 80: open=%v, latency=%v, error=%s\n", 
    result.Open, result.Latency, result.Error)

// Scan port range
results := manager.ScanPortRange(ctx, "localhost", 80, 90, "tcp")
for _, res := range results {
    if res.Open {
        fmt.Printf("Port %d: OPEN (%s)\n", res.Port, res.Service)
    }
}
```

### Banner Grab

```go
ctx := context.Background()

// Grab banner from SSH port
result := manager.BannerGrab(ctx, "localhost", 22)
if result.Error != "" {
    fmt.Printf("Error: %s\n", result.Error)
} else {
    fmt.Printf("Banner: %s\n", result.Banner)
}
```

### TCP Transfer (Send file)

```go
ctx := context.Background()

// Send file to remote host
file, err := os.Open("myfile.txt")
if err != nil {
    panic(err)
}
defer file.Close()

transferResult, err := manager.TransferTCP(ctx, "backup-server", 9000, file)
if err != nil {
    panic(err)
}
fmt.Printf("Sent %d bytes in %v\n", transferResult.BytesSent, transferResult.Duration)
```

### TCP Receive (Receive file)

```go
// Receive file on local port 9000
file, err := os.Create("received.txt")
if err != nil {
    panic(err)
}
defer file.Close()

transferResult, err := manager.ReceiveTCP(9000, file, 30*time.Second)
if err != nil {
    panic(err)
}
fmt.Printf("Received %d bytes in %v\n", transferResult.BytesReceived, transferResult.Duration)
```

### TCP Proxy

```go
// Create TCP proxy: local port 8080 → remote host:80
listener, err := manager.ProxyTCP(8080, "remote-server", 80)
if err != nil {
    panic(err)
}
fmt.Printf("Proxy started: %s\n", listener.Address)

// Keep running...
time.Sleep(1 * time.Hour)

// Stop proxy
manager.StopListener(listener.ID)
```

### List Listeners

```go
// List active listeners
listeners := manager.ListListeners()
fmt.Printf("Active listeners: %d\n", len(listeners))
for _, l := range listeners {
    fmt.Printf("  %s: %s (%s) - Active: %v\n", 
        l.ID, l.Address, l.Protocol, l.Active)
}
```

### Async Operations

```go
ctx := context.Background()

// Async scan port
result := manager.ScanPortAsync(ctx, "localhost", 80, "tcp")
select {
case scanRes := <-result.Ch:
    fmt.Printf("Port scan: open=%v\n", scanRes.Open)
case err := <-result.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async scan port range
result2 := manager.ScanPortRangeAsync(ctx, "localhost", 1, 1024, "tcp")
select {
case scanResults := <-result2.Ch:
    fmt.Printf("Found %d open ports\n", countOpen(scanResults))
case err := <-result2.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async banner grab
result3 := manager.BannerGrabAsync(ctx, "localhost", 22)
select {
case bannerRes := <-result3.Ch:
    fmt.Printf("Banner: %s\n", bannerRes.Banner)
case err := <-result3.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async send TCP
result4 := manager.SendTCPAsync(ctx, "example.com", 80, []byte("GET / HTTP/1.0\r\n\r\n"))
select {
case resp := <-result4.Ch:
    fmt.Printf("Response: %s\n", string(resp))
case err := <-result4.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async send string TCP
result5 := manager.SendStringTCPAsync(ctx, "example.com", 80, "GET / HTTP/1.0\r\n\r\n")
select {
case resp := <-result5.Ch:
    fmt.Printf("Response: %s\n", resp)
case err := <-result5.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async transfer TCP
file, _ := os.Open("data.txt")
result6 := manager.TransferTCPAsync(ctx, "backup", 9000, file)
select {
case transferRes := <-result6.Ch:
    fmt.Printf("Transferred %d bytes\n", transferRes.BytesSent)
case err := <-result6.ErrCh:
    fmt.Printf("Error: %v\n", err)
}
```

### Get Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Active Listeners: %v\n", status["active_listeners"])
fmt.Printf("Default Timeout: %.0f sec\n", status["default_timeout_sec"])
fmt.Printf("Default Host: %s\n", status["default_host"])
fmt.Printf("Default Port: %v\n", status["default_port"])

if status["connected"] == false {
    if errMsg, ok := status["error"].(string); ok {
        fmt.Printf("Error: %s\n", errMsg)
    }
}
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/hashicorp/go-retryablehttp` | HTTP client with retry logic (used for some operations) |
| `github.com/spf13/viper` | Configuration management |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `bufio`, `context`, `fmt`, `io`, `net`, `sync`, `time` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: Returned from `ConnectTCP`/`ConnectUDP`
- **Timeout errors**: Handled via `SetDeadline`/`DialTimeout`
- **Listener errors**: Returned from `ListenTCP`/`ListenUDP`
- **Scan errors**: Captured in `NetcatScanResult.Error`
- **Nil checks**: Public methods handle nil receiver gracefully in `GetStatus()`

Example error handling:

```go
conn, err := manager.ConnectTCP(host, port)
if err != nil {
    if strings.Contains(err.Error(), "connection refused") {
        return fmt.Errorf("port closed: %w", err)
    }
    if strings.Contains(err.Error(), "timeout") {
        return fmt.Errorf("connection timeout: %w", err)
    }
    return fmt.Errorf("connection failed: %w", err)
}
```

## Common Pitfalls

### 1. Port Already in Use

**Problem**: `bind: address already in use` error when listening

**Solution**: 
- Choose a different port
- Check if another process is using the port (`lsof -i :port`)
- Wait for socket to release (TIME_WAIT)

### 2. Connection Refused

**Problem**: `connection refused` when connecting

**Solution**:
- Verify the host is reachable (ping)
- Check if the service is running on the target port
- Verify firewall settings

### 3. Timeout Issues

**Problem**: Operations timing out

**Solution**:
```go
// Adjust timeout via viper
viper.Set("netcat.timeout_sec", 30)
```

Or use context with TCP connect (though `ConnectTCP` uses `net.DialTimeout` with manager's Timeout).

### 4. Too Many Open Files

**Problem**: `too many open files` during port scan

**Solution**:
- The scan uses semaphore limited to 100 concurrent goroutines
- Increase system limits: `ulimit -n 4096`
- Reduce scan range

### 5. Listener Not Stopping

**Problem**: Listener continues after `StopListener`

**Solution**:
- Ensure you call `StopListener` with correct ID
- Listener IDs are: `tcp-{port}` or `udp-{port}` or `proxy-{localPort}-to-{remoteHost}-{remotePort}`
- Check `ListListeners()` to see active listeners

### 6. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewNetcatManager()` succeeded
- Verify `netcat.enabled` is true

## Advanced Usage

### Port Scan with Service Detection

```go
func scanWithServiceDetection(manager *netcat.NetcatManager, host string, start, end int) {
    ctx := context.Background()
    results := manager.ScanPortRange(ctx, host, start, end, "tcp")
    
    fmt.Printf("Scan results for %s:\n", host)
    for _, res := range results {
        if res.Open {
            service := detectService(res.Port)
            fmt.Printf("  Port %d: OPEN (Service: %s, Latency: %v)\n", 
                res.Port, service, res.Latency)
        }
    }
}

func detectService(port int) string {
    services := map[int]string{
        22:   "SSH",
        23:   "Telnet",
        25:   "SMTP",
        53:   "DNS",
        80:   "HTTP",
        443:  "HTTPS",
        3306: "MySQL",
        5432: "PostgreSQL",
        6379: "Redis",
        8080: "HTTP-Alt",
    }
    if s, ok := services[port]; ok {
        return s
    }
    return "Unknown"
}
```

### Simple Chat Server

```go
func chatServer(manager *netcat.NetcatManager) {
    listener, err := manager.ListenTCP(4000, func(conn net.Conn) {
        defer conn.Close()
        fmt.Fprintf(conn, "Welcome to chat! Your address: %s\n", conn.RemoteAddr())
        
        scanner := bufio.NewScanner(conn)
        for scanner.Scan() {
            msg := scanner.Text()
            fmt.Fprintf(conn, "You said: %s\n", msg)
            if msg == "quit" {
                return
            }
        }
    })
    if err != nil {
        panic(err)
    }
    fmt.Printf("Chat server on %s\n", listener.Address)
    
    // Keep running
    select {}
}
```

### TCP Proxy with Logging

```go
func loggingProxy(manager *netcat.NetcatManager, localPort int, remoteHost string, remotePort int) {
    id := fmt.Sprintf("proxy-%d-to-%s-%d", localPort, remoteHost, remotePort)
    
    listener, err := manager.ProxyTCP(localPort, remoteHost, remotePort)
    if err != nil {
        panic(err)
    }
    
    fmt.Printf("Proxy started: %s → %s:%d\n", listener.Address, remoteHost, remotePort)
    
    // Log proxy activity (simplified)
    ticker := time.NewTicker(1 * time.Minute)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            listeners := manager.ListListeners()
            for _, l := range listeners {
                if l.ID == id {
                    fmt.Printf("Proxy %s still active\n", l.ID)
                }
            }
        }
    }
}
```

### Batch Port Scanning Multiple Hosts

```go
func scanMultipleHosts(manager *netcat.NetcatManager, hosts []string, port int) {
    ctx := context.Background()
    
    for _, host := range hosts {
        result := manager.ScanPort(ctx, host, port, "tcp")
        if result.Error != "" {
            fmt.Printf("%s:%d - Error: %s\n", host, port, result.Error)
        } else {
            status := "CLOSED"
            if result.Open {
                status = "OPEN"
            }
            fmt.Printf("%s:%d - %s (latency: %v)\n", host, port, status, result.Latency)
        }
    }
}
```

## Internal Algorithms

### Port Scanning with Concurrency

```
ScanPortRange():
    │
    ├── semaphore = make(chan struct{}, 100)
    ├── resultChan = make(chan NetcatScanResult)
    │
    ├── For each port:
    │   ├── wg.Add(1)
    │   ├── semaphore <- struct{}{} (acquire)
    │   └── goroutine:
    │       ├── defer wg.Done()
    │       ├── defer release semaphore
    │       └── resultChan <- ScanPort(ctx, host, port, protocol)
    │
    ├── goroutine: wait for all, close channel
    └── Collect results from resultChan
```

### TCP Proxy Bidirectional Copy

```
ProxyTCP():
    │
    └── For each client connection:
        ├── DialTimeout to remote
        ├── goroutine: io.Copy(remote, client) → done channel
        ├── goroutine: io.Copy(client, remote) → done channel
        └── <-done (wait for either direction to finish)
```

### Banner Grab Probing

```
BannerGrab():
    │
    ├── ConnectTCP(host, port)
    ├── probes = ["", "HEAD / HTTP/1.0\r\n\r\n", "\r\n", "\n"]
    ├── For each probe:
    │   ├── If probe not empty: conn.Write(probe)
    │   ├── Read response (4096 bytes)
    │   └── If read successful: return banner
    └── Return "no banner received"
```

### Async Operation Pattern

```
ScanPortAsync(ctx, host, port, protocol):
    │
    └── ExecuteAsync(ctx, func):
        └── return ScanPort(ctx, host, port, protocol)
            └── Returns (*NetcatScanResult, error)
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.