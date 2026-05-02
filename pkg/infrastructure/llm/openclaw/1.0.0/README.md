# OpenClaw Vector Store Manager

## Overview

The `OpenClawManager` is a Go library for interacting with the OpenClaw vector store. It provides health checking and status monitoring for the OpenClaw vector database. The library uses HTTP API with optional API key authentication and includes retry logic for reliability.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Health Monitoring**: Check system health via `/health` endpoint
- **Status Monitoring**: Get connection status and API health
- **API Key Authentication**: Optional API key via `api-key` header
- **Retry Logic**: HTTP client with retry support (3 retries)
- **Worker Pool**: Async job execution support (4 workers)
- **Simple Initialization**: Minimal configuration required

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
    
    // Create OpenClaw manager (configuration via viper)
    manager, err := openclaw.NewOpenClawManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    // Get status
    status := manager.GetStatus()
    fmt.Printf("Connected: %v\n", status["connected"])
    fmt.Printf("Base URL: %s\n", status["base_url"])
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `OpenClawManager` | Main manager with HTTP client, API key, worker pool |

### Concurrency Model

```
┌─────────────────────────────────────────────────────┐
│                 OpenClawManager                        │
├─────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  retryablehttp │  → API requests with retry               │
│  │  .Client      │  → Optional API key authentication       │
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
│  BaseURL: http://{host}:{port}              │
│  APIKey: OpenClaw API key (api-key header)          │
└─────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewOpenClawManager(logger)
    │
    ├── Check viper config: "openclaw.enabled"
    ├── Get host: "openclaw.host" (default: "localhost")
    ├── Get port: "openclaw.port" (default: 8080)
    ├── Get api_key: "openclaw.api_key"
    │
    ├── Build baseURL: "http://{host}:{port}"
    │
    ├── Create retryablehttp.Client:
    │   ├── RetryMax = 3
    │   ├── RetryWaitMin = 1s
    │   ├── RetryWaitMax = 10s
    │   └── HTTPClient.Timeout = 60s
    │
    ├── Test connection: testConnection()
    │   └── GET /health (with optional api-key header)
    │
    ├── Create WorkerPool(4)
    ├── Start worker pool
    └── Return OpenClawManager
```

### 2. Health Check Flow

```
testConnection()
    │
    ├── GET /health
    ├── Set header: api-key = <api_key> (if provided)
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    └── Return error if failed
```

### 3. Get Status Flow

```
GetStatus()
    │
    ├── If nil: return connected=false
    ├── Create context with 5s timeout
    ├── GET /health
    ├── Set header: api-key = <api_key> (if provided)
    ├── Return connected, base_url, pool_active
    └── Note: Tests actual API connectivity
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `openclaw.enabled` | bool | false | Enable/disable OpenClaw manager |
| `openclaw.host` | string | "localhost" | OpenClaw host |
| `openclaw.port` | int | 8080 | OpenClaw port |
| `openclaw.api_key` | string | "" | OpenClaw API key (optional) |

## Usage Examples

### Get Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Base URL: %s\n", status["base_url"])

if status["connected"] == false {
    if errMsg, ok := status["error"].(string); ok {
        fmt.Printf("Error: %s\n", errMsg)
    }
}
```

### Health Check

```go
func healthCheck(manager *openclaw.OpenClawManager) error {
    status := manager.GetStatus()
    if status["connected"] == false {
        return fmt.Errorf("not connected to OpenClaw")
    }
    fmt.Printf("OpenClaw is healthy (URL: %s)\n", status["base_url"])
    return nil
}
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/hashicorp/go-retryablehttp` | HTTP client with retry logic |
| `github.com/spf13/viper` | Configuration management |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `net/http`, `time`, `context` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: OpenClaw not running or not reachable
- **API errors**: Returned from OpenClaw API calls
- **Nil checks**: Public methods check manager state

Example error handling:

```go
status := manager.GetStatus()
if status["connected"] == false {
    if errMsg, ok := status["error"].(string); ok {
        if strings.Contains(errMsg, "connection refused") {
            return fmt.Errorf("OpenClaw is not running: %s", errMsg)
        }
        return fmt.Errorf("OpenClaw connection failed: %s", errMsg)
    }
}
```

## Common Pitfalls

### 1. Manager Not Enabled

**Problem**: `openclaw.enabled` is false

**Solution**:
- Set `openclaw.enabled = true` in configuration
- Check viper configuration is loaded

### 2. OpenClaw Not Running

**Problem**: Cannot connect to OpenClaw

**Solution**:
- Ensure OpenClaw is running locally or at the specified host/port
- Check that the OpenClaw server is accessible
- Verify host and port settings

### 3. Invalid API Key

**Problem**: Authentication fails

**Solution**:
- Set `openclaw.api_key` to your OpenClaw API key
- Verify the API key is correct
- Note: API key is optional for some OpenClaw configurations

### 4. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewOpenClawManager()` succeeded
- Verify `openclaw.enabled` is true

## Advanced Usage

### Custom Health Check

```go
func customHealthCheck(manager *openclaw.OpenClawManager) error {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    req, err := retryablehttp.NewRequestWithContext(ctx, "GET", manager.BaseURL+"/health", nil)
    if err != nil {
        return err
    }
    
    if manager.APIKey != "" {
        req.Header.Set("api-key", manager.APIKey)
    }
    
    resp, err := manager.Client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode != http.StatusOK {
        return fmt.Errorf("health check failed with status: %d", resp.StatusCode)
    }
    
    fmt.Println("OpenClaw is healthy")
    return nil
}
```

## Internal Algorithms

### Connection Test

```
testConnection():
    │
    ├── GET /health
    ├── Set api-key header (if APIKey is set)
    ├── Check status code (200 OK)
    └── Return error if failed
```

### Status Check

```
GetStatus():
    │
    ├── GET /health
    ├── Set api-key header (if APIKey is set)
    ├── Check status code
    └── Return connected status and metadata
```

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/health` | GET | Health check |

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.