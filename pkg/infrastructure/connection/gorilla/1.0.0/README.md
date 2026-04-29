# Gorilla WebSocket Manager#

## Overview#

The `GorillaWebsocketManager` is a Go library for managing WebSocket client connections using the Gorilla WebSocket package. It provides automatic reconnection, ping/pong keep-alive, async operations, and message handling with a worker pool.#

**Import Path:** `stackyrd/pkg/infrastructure/connection/gorilla`#

## Features#

- **WebSocket Client**: Dial and maintain WebSocket connections with automatic reconnection#
- **Auto-Reconnect**: Configurable maximum reconnect attempts with delay backoff#
- **Keep-Alive**: Ping/pong handlers with configurable intervals and timeouts#
- **Message Sending**: Send text or binary messages, including JSON marshaling#
- **Message Handling**: Register custom handler for incoming messages (text/binary)#
- **Async Operations**: Async connect, send message, send JSON via worker pool#
- **Worker Pool**: Async job execution support (6 workers)#
- **Connection Management**: Connect, disconnect, and monitor connection status#
- **Context Support**: Graceful shutdown via context cancellation#

## Quick Start#

```go#
package main#

import (#
    "context"
    "fmt"
    "stackyrd/pkg/infrastructure/connection/gorilla"
    "stackyrd/pkg/logger"
)#

func main() {#
    // Initialize logger#
    log := logger.NewLogger()#
    
    // Create Gorilla WebSocket manager (configuration via viper)#
    manager, err := gorilla.NewGorillaWebsocketManager(log)#
    if err != nil {#
        panic(err)#
    }#
    defer manager.Close()#
    
    ctx := context.Background()#
    
    // Connect to WebSocket server#
    err = manager.Connect()#
    if err != nil {#
        panic(err)#
    }#
    
    // Send a message#
    err = manager.SendMessage(gorilla.WebSocket.TextMessage, []byte("Hello WebSocket!"))#
    if err != nil {#
        panic(err)#
    }#
    fmt.Println("Message sent!")#
}#
```#

## Architecture#

### Core Structs#

| Struct | Description |#
|--------|-------------|#
| `GorillaWebsocketManager` | Main manager with connection, dialer, worker pool |#
| `WebSocketMessage` | Message with type, raw data, and optional JSON payload |#

### Concurrency Model#

```
┌─────────────────────────────────────────────────────────────┐#
│                   GorillaWebsocketManager                 │#
├─────────────────────────────────────────────────────────────┤#
│  ┌────────────────┐                                      │#
│  │  WebSocket │  → Gorilla WebSocket connection          │#
│  │  Conn      │  → Ping/pong keep-alive                 │#
│  └────────────────┘                                      │#
│                                                         │#
│  ┌─────────────┐      ┌──────────────┐                   │#
│  │  WorkerPool │◄─────│ AsyncResult │                   │#
│  │  (6 workers)│      │   Channel   │                   │#
│  └─────────────┘      └──────────────┘                   │#
│         ▲                      │                            │#
│         │                      ▼                            │#
│  ┌─────────────┐      ┌──────────────┐                  │#
│  │  SubmitJob  │      │ ExecuteAsync │                  │#
│  └─────────────┘      └──────────────┘                  │#
│                                                         │#
│  URL: WebSocket server URL (ws:// or wss://)            │#
│  ReconnectMax: Maximum reconnect attempts (default: 5)       │#
│  ReconnectDelay: Delay between reconnects (default: 2s)   │#
│  PingInterval: Ping period (default: 30s)                  │#
│  PongTimeout: Pong wait timeout (default: 10s)            │#
└─────────────────────────────────────────────────────────────┘#
```#

## How It Works#

### 1. Initialization Flow#

```
NewGorillaWebsocketManager(l)
    │#
    ├── Check viper config: "websocket.enabled"
    ├── Get url: "websocket.url" (required)#
    ├── Get reconnectMax: "websocket.reconnect_max" (default: 5)#
    ├── Get reconnectDelay: "websocket.reconnect_delay" (default: 2s)#
    ├── Get pingInterval: "websocket.ping_interval" (default: 30s)#
    ├── Get pongTimeout: "websocket.pong_timeout" (default: 10s)#
    │#
    ├── Create Dialer with HandshakeTimeout, ReadBufferSize, WriteBufferSize#
    ├── Create context with cancel#
    ├── Test connection: testConnection() → Dial and immediate close#
    ├── Create WorkerPool(6)#
    ├── Start worker pool#
    └── Return GorillaWebsocketManager#
```#

### 2. Connect Flow#

```
Connect()
    │#
    ├── Lock mutex#
    ├── Check enabled flag#
    ├── If conn exists: close it#
    ├── Call connect(false)#
    │   ├── Loop with reconnect attempts:#
    │   │   ├── Check context cancelled#
    │   │   ├── Dial: dialer.Dial(url, nil)#
    │   │   ├── If success:#
    │   │   │   ├── Set pong handler with read deadline#
    │   │   │   ├── Start pingLoop() goroutine#
    │   │   │   ├── Start readPump() goroutine#
    │   │   │   └── Return nil#
    │   │   └── If fail:#
    │   │       ├── Increment attempt#
    │   │       ├── If loop and attempt >= reconnectMax: return error#
    │   │       ├── Log warning#
    │   │       └── Wait for delay (with context)#
    │   └── Return error#
    └── Unlock mutex#
```#

### 3. Ping Loop Flow#

```
pingLoop()
    │#
    ├── Create ticker with pingInterval#
    ├── Loop:#
    │   ├── Select on ctx.Done() → return#
    │   └── On ticker:#
    │       ├── Lock and get conn#
    │       ├── Set write deadline#
    │       ├── WriteMessage(PingMessage, nil)#
    │       └── If error: trigger reconnect()#
    └── Stop ticker#
```#

### 4. Read Pump Flow#

```
readPump()
    │#
    ├── Defer reconnect()#
    ├── Loop:#
    │   ├── Lock and get conn#
    │   ├── ReadMessage() → msgType, data, err#
    │   ├── If error:#
    │   │   ├── If unexpected close: log error and reconnect#
    │   │   └── Else: log normal close#
    │   ├── Create WebSocketMessage with Type and Data#
    │   ├── If text/binary: try JSON unmarshal to Payload#
    │   ├── Lock and get messageHandler#
    │   └── If handler set: call handler(msg)#
    └── (loop continues)#
```#

### 5. Send Message Flow#

```
SendMessage(msgType, data)
    │#
    ├── Lock mutex#
    ├── Check conn != nil#
    ├── conn.WriteMessage(msgType, data)#
    └── Return error#
```#

## Configuration#

### Viper Configuration Options#

| Key | Type | Default | Description |#
|-----|------|---------|-------------|#
| `websocket.enabled` | bool | false | Enable/disable WebSocket manager |#
| `websocket.url` | string | "" | WebSocket server URL (ws:// or wss://) |#
| `websocket.reconnect_max` | int | 5 | Maximum reconnect attempts |#
| `websocket.reconnect_delay` | duration | 2s | Delay between reconnect attempts |#
| `websocket.ping_interval` | duration | 30s | Ping interval for keep-alive |#
| `websocket.pong_timeout` | duration | 10s | Timeout waiting for pong |#

### Environment Variables#

The WebSocket manager does not directly use environment variables. Configuration is read through Viper, which can load from:
- Configuration files (JSON, YAML, TOML)#
- Environment variables (if Viper is configured for it)#

You can set WebSocket parameters via:#
```go#
viper.Set("websocket.enabled", true)#
viper.Set("websocket.url", "ws://localhost:8080/ws")#
viper.Set("websocket.reconnect_max", 10)#
```#

## Usage Examples#

### Connecting and Sending Messages#

```go#
ctx := context.Background()#

// Connect to WebSocket server#
err := manager.Connect()#
if err != nil {#
    panic(err)#
}#
fmt.Println("Connected!")#

// Send a text message#
err = manager.SendMessage(gorilla.WebSocket.TextMessage, []byte("Hello!"))#
if err != nil {#
    panic(err)#
}#

// Send a binary message#
err = manager.SendMessage(gorilla.WebSocket.BinaryMessage, []byte{0x01, 0x02, 0x03})#
if err != nil {#
    panic(err)#
}#

// Send JSON payload (auto-marshaled)#
err = manager.SendJSON(map[string]interface{}{#
    "type": "greeting",#
    "message": "Hello from client",#
})#
if err != nil {#
    panic(err)#
}#
```#

### Handling Incoming Messages#

```go#
// Set a message handler#
manager.SetMessageHandler(func(msg gorilla.WebSocketMessage) {#
    fmt.Printf("Received message: type=%d, data=%s\n", msg.Type, string(msg.Data))#
    if msg.Payload != nil {#
        fmt.Printf("  Parsed payload: %v\n", msg.Payload)#
    }#
})#

// Keep the connection alive to receive messages#
// (readPump runs in background goroutine)#
// You can also run your own loop:#
for {#
    // Your logic here#
    time.Sleep(1 * time.Second)#
}#
```#

### Disconnecting#

```go#
// Disconnect gracefully#
err := manager.Disconnect()#
if err != nil {#
    fmt.Printf("Disconnect error: %v\n", err)#
}#
```#

### Async Operations#

```go#
ctx := context.Background()#

// Async connect#
result := manager.ConnectAsync()#
select {#
case <-result.Ch:#    fmt.Println("Connected asynchronously!")#
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)#
}#

// Async send message#
result = manager.SendMessageAsync(gorilla.WebSocket.TextMessage, []byte("Async hello"))#
select {#
case <-result.Ch:#    fmt.Println("Message sent asynchronously!")#
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)#
}#

// Async send JSON#
result = manager.SendJSONAsync(map[string]interface{}{#
    "async": true,#
    "message": "Async JSON",#
})#
select {#
case <-result.Ch:#    fmt.Println("JSON sent asynchronously!")#
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)#
}#
```#

### Get Status#

```go#
// Get manager status#
status := manager.GetStatus()#
fmt.Printf("Connected: %v\n", status["connected"])#
fmt.Printf("URL: %s\n", status["url"])#
fmt.Printf("Reconnect Max: %v\n", status["reconnect_max"])#
fmt.Printf("Ping Interval: %s\n", status["ping_interval"])#

if status["connected"] == false {#
    if errMsg, ok := status["error"].(string); ok {#
        fmt.Printf("Error: %s\n", errMsg)#
    }#
}#
```#

## Dependencies#

| Dependency | Role |#
|------------|------|#
| `github.com/gorilla/websocket` | WebSocket client library |#
| `github.com/spf13/viper` | Configuration management |#
| `stackyrd/config` | Internal configuration package |#
| `stackyrd/pkg/logger` | Structured logging |#
| Standard library | `context`, `encoding/json`, `fmt`, `net/http`, `sync`, `time` |#

## Error Handling#

The library uses Go error handling patterns:

- **Connection failures**: `testConnection()` validates via dial and immediate close#
- **Dial errors**: Handled with reconnect logic (max attempts, delay)#
- **Read errors**: Trigger reconnect via `readPump()` defer#
- **Write errors**: Ping failure triggers reconnect#
- **Context cancellation**: `pingLoop()` and `readPump()` respect context#
- **Nil checks**: Public methods handle nil receiver gracefully in `GetStatus()`#

Example error handling:#

```go#
err := manager.SendMessage(gorilla.WebSocket.TextMessage, []byte("test"))#
if err != nil {#
    if strings.Contains(err.Error(), "closed") {#
        // Connection lost, may reconnect automatically#
        fmt.Println("Connection closed, attempting reconnect...")#
        // Optionally call manager.Connect() again#
    }#
    return fmt.Errorf("failed to send message: %w", err)#
}#
```#

## Common Pitfalls#

### 1. WebSocket URL Not Configured#

**Problem**: `websocket.url is required` error#

**Solution**: 
- Configure URL: `viper.Set("websocket.url", "ws://localhost:8080/ws")`#
- For TLS: use `wss://` scheme#
- Ensure URL includes path if required#

### 2. Connection Refused#

**Problem**: `connection refused` or `no route to host` errors#

**Solution**:#
- Verify server is running and accessible#
- Check network connectivity (ping/telnet to server port)#
- Verify firewall allows WebSocket traffic#

### 3. Reconnect Not Working#

**Problem**: Connection lost and not reconnecting#

**Solution**:#
- Check `websocket.reconnect_max` is set > 0#
- Monitor logs for reconnect attempts#
- Ensure `readPump()` is running (it triggers reconnect on read error)#

### 4. Ping/Pong Issues#

**Problem**: Connection dropping due to ping timeout#

**Solution**:#
- Increase `websocket.pong_timeout` (default: 10s)#
- Adjust `websocket.ping_interval` (default: 30s)#
- Check network latency#

### 5. Message Handler Not Called#

**Problem**: Incoming messages not being processed#

**Solution**:#
- Ensure `SetMessageHandler()` is called before connecting#
- Verify server is sending messages in correct format (text/binary)#
- Check that `readPump()` goroutine is running (it starts on Connect)#

### 6. Context Cancellation#

**Problem**: Unable to stop WebSocket gracefully#

**Solution**:#
```go#
// The manager's internal context is cancelled on Close()#
// To stop externally, you can't directly; but Close() handles it.#
err := manager.Close()#
```#

## Advanced Usage#

### Custom Reconnection Logic#

```go#
func customReconnect(manager *gorilla.GorillaWebsocketManager) {#
    for attempt := 1; attempt <= 5; attempt++ {#
        fmt.Printf("Reconnect attempt %d...\n", attempt)#
        err := manager.Connect()#
        if err == nil {#
            fmt.Println("Reconnected!")#
            return#
        }#
        fmt.Printf("Failed: %v, waiting...\n", err)#
        time.Sleep(time.Duration(attempt) * time.Second)#
    }#
    fmt.Println("Max attempts reached")#
}#
```#

### JSON Message Processing#

```go#
type Message struct {#
    Type    string      `json:"type"`#
    Payload interface{} `json:"payload"`#
}#

func handleJSONMessages(manager *gorilla.GorillaWebsocketManager) {#
    manager.SetMessageHandler(func(msg gorilla.WebSocketMessage) {#
        if msg.Payload != nil {#
            // Payload already parsed by readPump#
            fmt.Printf("JSON message: %v\n", msg.Payload)#
            return#
        }#
        
        // Try to parse manually if not auto-parsed#
        var m Message#
        if err := json.Unmarshal(msg.Data, &m); err == nil {#
            fmt.Printf("Parsed message type: %s\n", m.Type)#
        } else {#
            fmt.Printf("Raw message: %s\n", string(msg.Data))#
        }#
    })#
}#
```#

### Monitoring Connection Health#

```go#
func monitorConnection(manager *gorilla.GorillaWebsocketManager) {#
    ticker := time.NewTicker(10 * time.Second)#
    defer ticker.Stop()#
    
    for {#
        select {#
        case <-ticker.C:#            status := manager.GetStatus()#
            if status["connected"] == false {#
                fmt.Println("WARNING: WebSocket disconnected!")#
                // Optionally trigger reconnect#
                go manager.Connect()#
            } else {#
                fmt.Println("WebSocket connection healthy")#
            }#
        }#
    }#
}#
```#

### Sending Binary Data#

```go#
func sendBinaryData(manager *gorilla.GorillaWebsocketManager, data []byte) error {#
    err := manager.SendMessage(gorilla.WebSocket.BinaryMessage, data)#
    if err != nil {#
        return fmt.Errorf("failed to send binary data: %w", err)#
    }#
    return nil#
}#
```#

## Internal Algorithms#

### Reconnection Algorithm#

```
connect(loop):
    │#
    ├── attempt = 0#
    ├── Loop:#
    │   ├── Check context cancelled → return error#
    │   ├── Dial: dialer.Dial(url, nil)#
    │   ├── If success:#
    │   │   ├── Set conn#
    │   │   ├── Set pong handler with deadline#
    │   │   ├── Start pingLoop() goroutine#
    │   │   ├── Start readPump() goroutine#
    │   │   └── Return nil#
    │   └── If fail:#
    │       ├── attempt++#
    │       ├── If loop and attempt >= reconnectMax: return error#
    │       ├── Log warning#
    │       └── Wait for delay (with context)#
    └── Return last error#
```#

### Ping/Pong Keep-Alive#

```
pingLoop():
    │#
    ├── ticker := time.NewTicker(pingInterval)#
    ├── Defer ticker.Stop()#
    ├── Loop:#
    │   ├── Select ctx.Done() → return#
    │   └── On ticker:#
    │       ├── Lock, get conn#
    │       ├── Set write deadline (10s)#
    │       ├── WriteMessage(PingMessage, nil)#
    │       └── If error: reconnect()#
    └── (exits on context cancel)#
```#

### Message Reading and Dispatching#

```
readPump():
    │#
    ├── Defer reconnect()#
    ├── Loop:#
    │   ├── Lock, get conn#
    │   ├── ReadMessage() → msgType, data, err#
    │   ├── If err:#
    │   │   ├── If unexpected close: log error#
    │   │   └── Else: log normal close#
    │   ├── Create WebSocketMessage{Type, Data}#
    │   ├── If text/binary: try JSON unmarshal to Payload#
    │   ├── Lock, get messageHandler#
    │   └── If handler: handler(msg)#
    └── (defer triggers reconnect on exit)#
```#

### Async Operation Pattern#

```
ConnectAsync():
    │#
    └── ExecuteAsync(ctx, func):#
        └── return Connect()#
            └── Returns (struct{}, error)#
```#

## License#

This code is part of the Stackyrd project. See the main project LICENSE file for details.#