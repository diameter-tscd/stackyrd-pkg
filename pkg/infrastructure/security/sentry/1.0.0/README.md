# Sentry Error Tracking Manager#

## Overview#

The `SentryManager` is a Go library for managing Sentry error tracking and performance monitoring. It provides error capturing, message logging, custom event sending, performance transaction tracking, breadcrumb management, user context setting, and metrics recording. The library uses the official Sentry Go SDK with configurable sampling rates and async job support.

**Import Path:** `stackyrd/pkg/infrastructure/security/sentry`

## Features#

- **Error Capturing**: Capture exceptions and send to Sentry with tags#
- **Message Capturing**: Send custom messages with log levels (debug, info, warning, error, fatal)#
- **Custom Events**: Send structured events with user context, tags, extra data, fingerprints#
- **Performance Monitoring**: Start and finish transactions and spans for performance tracking#
- **Breadcrumbs**: Add breadcrumbs for event context#
- **User Context**: Set user information for all subsequent events#
- **Tags & Extra**: Set tags and extra context#
- **Request Context**: Attach HTTP request context to events#
- **Panic Recovery**: Capture panics and report to Sentry#
- **Metrics**: Record custom metrics (counter, gauge, distribution, set)#
- **Flush**: Wait for pending events to be sent#
- **Worker Pool**: Async job execution support (4 workers)#
- **Status Monitoring**: Get connection status and configuration#

## Quick Start#

```go
package main

import (
    "context"
    "errors"
    "stackyrd/pkg/infrastructure/security/sentry"
    "stackyrd/pkg/logger"
    "time"
)

func main() {
    // Initialize logger
    log := logger.NewLogger()
    
    // Create Sentry manager (configuration via viper)
    manager, err := sentry.NewSentryManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    // Capture an error
    err = errors.New("something went wrong")
    eventID := manager.CaptureError(err, map[string]string{"component": "main"})
    if eventID != nil {
        println("Error captured:", eventID.String())
    }
    
    // Capture a message
    msgID := manager.CaptureMessage("Application started", "info")
    if msgID != nil {
        println("Message captured:", msgID.String())
    }
    
    // Start a performance transaction
    ctx := context.Background()
    tx := manager.StartTransaction(ctx, "main-operation", "http.server")
    time.Sleep(100 * time.Millisecond) // simulate work
    manager.FinishTransaction(tx)
    
    // Flush pending events
    manager.Flush(5 * time.Second)
}
```

## Architecture#

### Core Structs#

| Struct | Description |
|--------|-------------|
| `SentryManager` | Main manager with Sentry SDK configuration and hub reference |
| `SentryEvent` | Custom event with message, level, tags, extra, user, request, fingerprint |
| `SentryUser` | User context (ID, email, username, IP, data) |
| `SentryRequest` | HTTP request context (URL, method, headers, data) |
| `SentryBreadcrumb` | Breadcrumb for event context (category, message, level, type, data) |
| `SentryTransaction` | Performance transaction (name, operation, description, tags, span) |
| `SentrySpan` | Nested span within a transaction |
| `SentryMetric` | Custom metric (name, type, value, tags, unit) |

### Concurrency Model#

```
┌─────────────────────────────────────────────────────┐
│                   SentryManager                        │
├─────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  sentry.Hub   │  → Sentry SDK hub for scoping          │
│  └────────────────┘                                      │
│                                                          │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ AsyncResult │                   │
│  │  (4 workers)│      │   Channel   │                   │
│  └─────────────┘      └──────────────┘                   │
│         ▲                      │                            │
│         │                      ▼                            │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  SubmitJob  │      │ ExecuteAsync │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                          │
│  DSN: Sentry DSN for authentication                  │
│  Environment: e.g., "production", "staging"            │
│  Release: e.g., "myapp@1.0.0"                      │
└─────────────────────────────────────────────────────┘
```

## How It Works#

### 1. Initialization Flow#

```
NewSentryManager(logger)
    │
    ├── Check viper config: "sentry.enabled"
    ├── Get dsn: "sentry.dsn"
    ├── Get environment: "sentry.environment"
    ├── Get release: "sentry.release"
    ├── Get debug: "sentry.debug"
    ├── Get sample_rate: "sentry.sample_rate" (default 1.0)
    ├── Get traces_sample_rate: "sentry.traces_sample_rate" (default 0.1)
    ├── Get profiles_sample_rate: "sentry.profiles_sample_rate" (default 0.0)
    ├── Get server_name: "sentry.server_name"
    ├── Get dist: "sentry.dist"
    ├── Get max_breadcrumbs: "sentry.max_breadcrumbs" (default 100)
    ├── Get ignore_errors: "sentry.ignore_errors"
    │
    ├── Initialize Sentry SDK: sentry.Init()
    │   ├── Dsn, Environment, Release, Debug
    │   ├── SampleRate, TracesSampleRate, ProfilesSampleRate
    │   ├── ServerName, Dist, MaxBreadcrumbs
    │   ├── IgnoreErrors, AttachStacktrace, EnableTracing
    │
    ├── Store hub reference: sentry.CurrentHub()
    ├── Set initialized = true
    │
    ├── Create WorkerPool(4)
    ├── Start worker pool
    └── Return SentryManager
```

### 2. Capture Error Flow#

```
CaptureError(err, tags...)
    │
    ├── Check initialized
    ├── Create new scope
    ├── Set tags if provided
    ├── Call sentry.CaptureException(err)
    └── Return *sentry.EventID
```

### 3. Transaction Flow#

```
StartTransaction(ctx, name, operation)
    │
    ├── Check initialized
    ├── Start span: sentry.StartSpan(ctx, operation)
    ├── Set description = name
    └── Return *SentryTransaction

FinishTransaction(tx)
    │
    ├── Check initialized and tx valid
    ├── Set tags if any
    ├── Call tx.Span.Finish()
    └── Log duration
```

## Configuration#

### Viper Configuration Options#

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `sentry.enabled` | bool | false | Enable/disable Sentry manager |
| `sentry.dsn` | string | "" | Sentry DSN (Data Source Name) |
| `sentry.environment` | string | "" | Environment name (e.g., "production") |
| `sentry.release` | string | "" | Release version (e.g., "myapp@1.0.0") |
| `sentry.debug` | bool | false | Enable debug logging |
| `sentry.sample_rate` | float64 | 1.0 | Error sample rate (0.0-1.0) |
| `sentry.traces_sample_rate` | float64 | 0.1 | Performance trace sample rate |
| `sentry.profiles_sample_rate` | float64 | 0.0 | Profiling sample rate |
| `sentry.server_name` | string | "" | Server name override |
| `sentry.dist` | string | "" | Distribution identifier |
| `sentry.max_breadcrumbs` | int | 100 | Maximum breadcrumbs to record |
| `sentry.ignore_errors` | []string | [] | Error patterns to ignore |

## Usage Examples#

### Capture Error#

```go
err := errors.New("database connection failed")
eventID := manager.CaptureError(err, map[string]string{
    "component": "database",
    "operation": "connect",
})
if eventID != nil {
    fmt.Printf("Error captured: %s\n", eventID.String())
}
```

### Capture Message#

```go
msgID := manager.CaptureMessage("User logged in", "info", map[string]string{
    "user_id": "12345",
})
if msgID != nil {
    fmt.Printf("Message captured: %s\n", msgID.String())
}
```

### Capture Custom Event#

```go
event := sentry.SentryEvent{
    Message: "Payment processed",
    Level:   "info",
    Tags: map[string]string{
        "payment_id": "pay_123",
        "amount":      "99.99",
    },
    User: &sentry.SentryUser{
        ID:    "user_456",
        Email: "user@example.com",
    },
    Extra: map[string]interface{}{
        "processor": "stripe",
        "currency": "USD",
    },
}
eventID := manager.CaptureEvent(event)
if eventID != nil {
    fmt.Printf("Event captured: %s\n", eventID.String())
}
```

### Performance Transaction#

```go
ctx := context.Background()

// Start transaction
tx := manager.StartTransaction(ctx, "process-order", "http.server")
defer manager.FinishTransaction(tx)

// Add tags
tx.Tags = map[string]string{"order_id": "order_789"}

// Start a child span
span := manager.StartSpan(tx, "payment", "payment.process")
// ... do payment work ...
manager.FinishSpan(span)

// ... more work ...
```

### Breadcrumbs#

```go
manager.AddBreadcrumb(sentry.SentryBreadcrumb{
    Category: "ui.click",
    Message:  "User clicked checkout button",
    Level:    "info",
    Data: map[string]interface{}{
        "button": "checkout",
        "page":   "cart",
    },
})
```

### Set User Context#

```go
manager.SetUser(sentry.SentryUser{
    ID:       "user_123",
    Email:    "john@example.com",
    Username: "john_doe",
    IP:       "192.168.1.1",
    Data:     map[string]string{"plan": "premium"},
})
```

### Set Tags and Extra#

```go
manager.SetTag("environment", "production")
manager.SetExtra("server_region", "us-east-1")
manager.SetContext("device", map[string]interface{}{
    "brand": "Apple",
    "model": "iPhone 12",
})
```

### Panic Recovery#

```go
func riskyFunction() {
    defer manager.Recover()
    // ... code that may panic ...
    panic("something went wrong")
}

func riskyFunctionWithContext(ctx context.Context) {
    defer manager.RecoverWithContext(ctx)
    // ... code that may panic ...
}
```

### Flush Events#

```go
// Wait up to 5 seconds for all pending events to be sent
flushed := manager.Flush(5 * time.Second)
if flushed {
    fmt.Println("All events flushed")
} else {
    fmt.Println("Flush timed out")
}
```

### Async Operations#

```go
ctx := context.Background()

// Async capture error
err := errors.New("async error")
asyncResult := manager.CaptureErrorAsync(ctx, err, map[string]string{"async": "true"})
eventID, err := asyncResult.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Async error captured: %s\n", eventID.String())

// Async capture message
asyncMsg := manager.CaptureMessageAsync(ctx, "Async message", "info")
msgID, err := asyncMsg.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Async message captured: %s\n", msgID.String())
```

### Getting Status#

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Environment: %s\n", status["environment"])
fmt.Printf("Release: %s\n", status["release"])
fmt.Printf("Sample Rate: %.2f\n", status["sample_rate"])
fmt.Printf("Debug: %v\n", status["debug"])

if status["connected"] == false {
    if errMsg, ok := status["error"].(string); ok {
        fmt.Printf("Error: %s\n", errMsg)
    }
}
```

## Dependencies#

| Dependency | Role |
|------------|------|
| `github.com/getsentry/sentry-go` | Official Sentry Go SDK |
| `github.com/spf13/viper` | Configuration management |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `context`, `fmt`, `time` |

## Error Handling#

The library uses Go error handling patterns:

- **SDK initialization errors**: Returned from `sentry.Init()`
- **Nil checks**: Public methods check `initialized` flag

Example error handling:

```go
manager, err := sentry.NewSentryManager(log)
if err != nil {
    if strings.Contains(err.Error(), "dsn") {
        return fmt.Errorf("invalid Sentry DSN: %w", err)
    }
    return fmt.Errorf("Sentry initialization failed: %w", err)
}
```

## Common Pitfalls#

### 1. Manager Not Enabled#

**Problem**: `sentry.enabled` is false

**Solution**:
- Set `sentry.enabled = true` in configuration
- Check viper configuration is loaded

### 2. Invalid DSN#

**Problem**: `sentry.dsn` is missing or invalid

**Solution**:
- Set `sentry.dsn` to your Sentry DSN
- DSN format: `https://<key>@sentry.io/<project>`
- Verify DSN is correct in Sentry project settings

### 3. Events Not Sent#

**Problem**: Events not appearing in Sentry

**Solution**:
- Check `sentry.dsn` is correct
- Ensure network can reach Sentry
- Call `manager.Flush(timeout)` before exiting
- Check `sentry.sample_rate` is not 0

### 4. Performance Transactions Not Working#

**Problem**: Transactions not showing in Sentry

**Solution**:
- Set `sentry.traces_sample_rate > 0` (e.g., 0.1 for 10%)
- Ensure `EnableTracing: true` in init
- Finish transactions with `FinishTransaction()`

### 5. Worker Pool Not Available#

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewSentryManager()` succeeded
- Verify `sentry.enabled` is true

## Advanced Usage#

### Health Check#

```go
func healthCheck(manager *sentry.SentryManager) error {
    status := manager.GetStatus()
    if status["connected"] == false {
        return fmt.Errorf("Sentry not connected")
    }
    fmt.Printf("Sentry is healthy (env: %s)\n", status["environment"])
    return nil
}
```

### Batch Error Reporting#

```go
func reportMultipleErrors(manager *sentry.SentryManager, errs []error) {
    for _, err := range errs {
        manager.CaptureError(err, map[string]string{"batch": "true"})
    }
    manager.Flush(5 * time.Second)
}
```

### Custom Error Filtering#

```go
func captureIfRelevant(manager *sentry.SentryManager, err error) {
    if err == nil {
        return
    }
    // Only capture specific errors
    if strings.Contains(err.Error(), "critical") {
        manager.CaptureError(err, map[string]string{"priority": "high"})
    }
}
```

## Internal Algorithms#

### Error Capture Flow#

```
CaptureError():
    │
    ├── Check initialized
    ├── Create scope
    ├── Set tags if provided
    ├── Call sentry.CaptureException()
    └── Return EventID
```

### Transaction Tracking#

```
StartTransaction():
    │
    ├── Check initialized
    ├── sentry.StartSpan(ctx, operation)
    ├── Set description
    └── Return transaction wrapper

FinishTransaction():
    │
    ├── Check initialized and valid
    ├── Set tags if any
    ├── span.Finish()
    └── Log duration
```

## License#

This code is part of the Stackyrd project. See the main project LICENSE file for details.