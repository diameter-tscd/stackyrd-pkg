# OpenShift Manager

## Overview

The `OpenShiftManager` is a minimal Go library for OpenShift integration. It provides basic configuration handling, connection testing, and worker pool initialization for OpenShift container platform management.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Configuration Management**: Viper-based configuration for host, port, and authentication token
- **Connection Testing**: Basic health check endpoint verification
- **Worker Pool**: Async job execution support with configurable pool size
- **TLS Support**: Optional TLS with configurable insecure skip
- **Retry Logic**: Automatic retries with exponential backoff via `go-retryablehttp`
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
    
    // Create OpenShift manager (configuration via viper)
    manager, err := openshift.NewOpenShiftManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Check status
    status := manager.GetStatus()
    fmt.Printf("Connected: %v\n", status["connected"])
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `OpenShiftManager` | Main manager with HTTP client and worker pool |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   OpenShiftManager                         │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │ retryablehttp  │  → Retry logic (max 3, 1-10s wait)│
│  │ Client         │  → HTTP timeout: 30s                   │
│  └────────────────┘                                      │
│                                                         │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ SubmitAsyncJob│                   │
│  │  (4 workers)│      └──────────────┘                   │
│  └─────────────┘                                          │
│                                                         │
│  BaseURL: OpenShift API URL (default: https://localhost:6443)│
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewOpenShiftManager()
    │
    ├── Check viper config: "openshift.enabled"
    ├── Get host: "openshift.host" (default: "localhost")
    ├── Get port: "openshift.port" (default: 6443)
    ├── Get token: "openshift.token"
    ├── Build baseURL: "https://{host}:{port}"
    ├── Create retryablehttp.Client (max 3 retries, 1-10s wait, 30s timeout)
    ├── Configure logger adapter
    ├── Create OpenShiftManager with client, baseURL, token
    ├── Create WorkerPool(4)
    ├── Start worker pool
    └── Return OpenShiftManager
```

### 2. Status Check Flow

```
GetStatus()
    │
    ├── If manager is nil: return connected=false
    ├── Create context with 5s timeout
    ├── Build URL: baseURL + "/healthz"
    ├── Create GET request
    ├── If token != "": set Authorization header
    ├── Execute request
    ├── Check status code == 200
    └── Return status map
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `openshift.enabled` | bool | false | Enable/disable OpenShift manager |
| `openshift.host` | string | "localhost" | OpenShift server hostname |
| `openshift.port` | int | 6443 | OpenShift server port |
| `openshift.token` | string | "" | Authentication token |

### Environment Variables

The OpenShift manager does not directly use environment variables. Configuration is read through Viper, which can load from:
- Configuration files (JSON, YAML, TOML)
- Environment variables (if Viper is configured for it)
- Command-line flags

## Usage Examples

### Basic Operations

```go
ctx := context.Background()

// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])

if status["connected"] == true {
    fmt.Printf("OpenShift is reachable at: %s\n", status["url"])
} else {
    if errMsg, ok := status["error"].(string); ok {
        fmt.Printf("Error: %s\n", errMsg)
    }
}
```

### Health Check

```go
func checkOpenShiftHealth(manager *openshift.OpenShiftManager) bool {
    status := manager.GetStatus()
    
    connected, ok := status["connected"].(bool)
    if !ok || !connected {
        fmt.Println("OpenShift is not reachable")
        return false
    }
    
    fmt.Println("✓ OpenShift is healthy")
    return true
}
```

### Async Job Execution

```go
// Submit custom jobs to the worker pool
manager.SubmitAsyncJob(func() {
    ctx := context.Background()
    
    // Perform async OpenShift operations here
    status := manager.GetStatus()
    if status["connected"] == true {
        fmt.Println("Background check: OpenShift is connected")
    }
})
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/hashicorp/go-retryablehttp` | HTTP client with retry logic for API resilience |
| `github.com/spf13/viper` | Configuration management for host, port, token |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `context`, `crypto/tls`, `net/http`, `time`, `fmt` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: `GetStatus()` validates connectivity via `/healthz` endpoint
- **API errors**: HTTP status codes are checked and errors returned
- **Context cancellation**: Operations respect `context.Context` for timeouts
- **Nil checks**: Public methods handle nil receiver gracefully in `GetStatus()`

Example error handling:

```go
status := manager.GetStatus()

if connected, ok := status["connected"].(bool); ok && !connected {
    if errMsg, ok := status["error"].(string); ok {
        return fmt.Errorf("openshift not reachable: %s", errMsg)
    }
    return fmt.Errorf("openshift not reachable")
}
```

## Common Pitfalls

### 1. OpenShift Not Running

**Problem**: Connection test fails

**Solution**: 
- Ensure OpenShift is running: `oc status` should show cluster info
- Check host/port configuration (default: `localhost:6443`)
- Verify OpenShift is accessible at the configured URL

### 2. Authentication Token Missing

**Problem**: 401/403 errors (when implementing API calls)

**Solution**:
- Configure token: `viper.Set("openshift.token", "your-token")`
- Or use `oc whoami -t` to get current token
- For now, the manager only does health checks without auth

### 3. TLS Certificate Issues

**Problem**: Certificate validation errors

**Solution**:
- The current implementation uses HTTPS by default
- For development with self-signed certs, modify the client transport
- Future enhancement: add `insecure_skip_tls` config option

### 4. Context Timeout

**Problem**: Operations timing out

**Solution**:
```go
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()
// Operations will use this timeout
```

## Advanced Usage

### Custom API Requests

While the current implementation is minimal, you can extend it for custom API calls:

```go
// Example: Get cluster version (not directly exposed)
func getClusterVersion(manager *openshift.OpenShiftManager, ctx context.Context) (string, error) {
    url := manager.BaseURL + "/apis/apps.openshift.io/v1/deployments"
    
    req, err := retryablehttp.NewRequestWithContext(ctx, "GET", url, nil)
    if err != nil {
        return "", err
    }
    
    if manager.Token != "" {
        req.Header.Set("Authorization", "Bearer "+manager.Token)
    }
    
    resp, err := manager.Client.Do(req)
    if err != nil {
        return "", err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode != 200 {
        return "", fmt.Errorf("API returned status: %d", resp.StatusCode)
    }
    
    body, _ := io.ReadAll(resp.Body)
    return string(body), nil
}
```

### Cluster Monitoring

```go
func monitorOpenShift(manager *openshift.OpenShiftManager) {
    ticker := time.NewTicker(30 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            status := manager.GetStatus()
            
            if connected, ok := status["connected"].(bool); ok && connected {
                fmt.Println("✓ OpenShift is healthy")
            } else {
                fmt.Println("✗ OpenShift is not reachable")
                if errMsg, ok := status["error"].(string); ok {
                    fmt.Printf("  Error: %s\n", errMsg)
                }
            }
        }
    }
}
```

### Integration with Other Components

```go
// Example: Use OpenShift manager with CI/CD pipeline
type CICDPipeline struct {
    openshiftManager *openshift.OpenShiftManager
    jenkinsManager    *jenkins.JenkinsManager
}

func (p *CICDPipeline) DeployToOpenShift(ctx context.Context, appName string) error {
    // Check OpenShift health
    status := p.openshiftManager.GetStatus()
    
    if connected, ok := status["connected"].(bool); !ok || !connected {
        return fmt.Errorf("openshift not available")
    }
    
    // Trigger Jenkins job for deployment
    // (This is a conceptual example)
    fmt.Printf("Deploying %s to OpenShift...\n", appName)
    
    return nil
}
```

## Internal Algorithms

### URL Construction

```
BaseURL Format: protocol://host:port

Default:
  protocol: https
  host: localhost
  port: 6443
  
Full URL: https://localhost:6443
```

### Health Check Flow

```
GetStatus()
    │
    ├── URL: baseURL + "/healthz"
    │
    ├── Request: GET /healthz
    │
    ├── If token set:
    │   └── Add header: Authorization: Bearer {token}
    │
    ├── Execute request (5s timeout)
    │
    └── Return: connected = (statusCode == 200)
```

### Retry Logic

Using `go-retryablehttp`:
- Maximum retries: 3
- Minimum wait: 1 second
- Maximum wait: 10 seconds
- HTTP client timeout: 30 seconds

Retries are automatic for transient network errors.

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.