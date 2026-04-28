# Pulsar Manager

## Overview

The `PulsarManager` is a Go library for managing Apache Pulsar operations via the Pulsar Admin REST API. It provides a comprehensive interface for managing tenants, namespaces, topics, subscriptions, and brokers with built-in retry logic and async job support.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Tenant Management**: List and manage Pulsar tenants
- **Namespace Operations**: Create, list, and manage namespaces
- **Topic Management**: Full CRUD operations on topics with partition support
- **Subscription Management**: Create and delete subscriptions, list subscribers
- **Broker Management**: List and monitor broker instances
- **Topic Statistics**: Retrieve detailed topic metrics
- **Retry Logic**: Automatic retries with exponential backoff via `go-retryablehttp`
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
    
    // Create Pulsar manager (configuration via viper)
    manager, err := pulsar.NewPulsarManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // List brokers
    brokers, err := manager.GetBrokers(ctx)
    if err != nil {
        panic(err)
    }
    
    for _, broker := range brokers {
        fmt.Printf("Broker: %s (URL: %s)\n", broker.Broker, broker.URL)
    }
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `PulsarManager` | Main manager with retryable HTTP client and worker pool |
| `PulsarTenant` | Represents a Pulsar tenant with admin roles and allowed clusters |
| `PulsarNamespace` | Represents a namespace within a tenant |
| `PulsarTopic` | Represents a topic with partition and persistence info |
| `PulsarSubscription` | Represents a subscription with backlog and rate metrics |
| `PulsarBroker` | Represents a broker instance with URLs |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   PulsarManager                            │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │ retryablehttp  │  → Retry logic (max 3, 1-10s wait)│
│  │ Client         │  → HTTP timeout: 30s                   │
│  └────────────────┘                                      │
│                                                         │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ SubmitAsyncJob│                   │
│  │  (6 workers)│      └──────────────┘                   │
│  └─────────────┘                                          │
│                                                         │
│  BaseURL: Pulsar Admin API URL (default: http://localhost:8080)│
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewPulsarManager()
    │
    ├── Check viper config: "pulsar.enabled"
    ├── Get admin URL: "pulsar.admin_url" (default: "http://localhost:8080")
    ├── Create retryablehttp.Client (max 3 retries, 1-10s wait, 30s timeout)
    ├── Test connection: manager.testConnection() → GetBrokers()
    ├── Create WorkerPool(6)
    └── Return PulsarManager
```

### 2. API Request Flow

```
apiRequest(ctx, method, path, body)
    │
    ├── Build URL: BaseURL + "/admin/v2" + path
    ├── If body: JSON marshal → bytes.Reader
    ├── Create retryablehttp.NewRequestWithContext()
    ├── Set Content-Type: application/json
    ├── Execute client.Do(req)
    ├── Read response body
    ├── Check status code (200-299 = success)
    └── Return response bytes or error
```

### 3. Tenant Listing Flow

```
ListTenants(ctx)
    │
    ├── apiRequest(ctx, "GET", "/tenants", nil)
    ├── JSON unmarshal → []string (tenant names)
    ├── Convert to []PulsarTenant
    └── Return tenants
```

### 4. Topic Stats Flow

```
GetTopicStats(ctx, tenant, namespace, topic)
    │
    ├── Build path: "/persistent/{tenant}/{namespace}/{topic}/stats"
    ├── apiRequest(ctx, "GET", path, nil)
    ├── JSON unmarshal → map[string]interface{}
    └── Return stats map
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `pulsar.enabled` | bool | false | Enable/disable Pulsar manager |
| `pulsar.admin_url` | string | "http://localhost:8080" | Pulsar Admin API URL |

### Environment Variables

The Pulsar manager does not directly use environment variables. Configuration is read through Viper, which can load from:
- Configuration files (JSON, YAML, TOML)
- Environment variables (if Viper is configured for it)
- Command-line flags

## Usage Examples

### Basic Operations

```go
ctx := context.Background()

// List brokers
brokers, err := manager.GetBrokers(ctx)
for _, b := range brokers {
    fmt.Printf("Broker: %s\n", b.Broker)
}

// List tenants
tenants, err := manager.ListTenants(ctx)
for _, t := range tenants {
    fmt.Printf("Tenant: %s\n", t.Name)
}

// List namespaces for a tenant
namespaces, err := manager.ListNamespaces(ctx, "my-tenant")
for _, ns := range namespaces {
    fmt.Printf("Namespace: %s\n", ns.Name)
}

// List topics in a namespace
topics, err := manager.ListTopics(ctx, "my-tenant", "my-namespace")
for _, t := range topics {
    fmt.Printf("Topic: %s (persistent: %v)\n", t.Name, t.Persistent)
}
```

### Topic Operations

```go
// Get topic statistics
stats, err := manager.GetTopicStats(ctx, "my-tenant", "my-namespace", "my-topic")
if err == nil {
    // stats is map[string]interface{} with detailed metrics
    fmt.Printf("Stats: %+v\n", stats)
}

// Create a subscription
err = manager.CreateSubscription(ctx, "my-tenant", "my-namespace", "my-topic", "my-subscription")

// Delete a subscription
err = manager.DeleteSubscription(ctx, "my-tenant", "my-namespace", "my-topic", "my-subscription")

// List subscriptions for a topic
subscriptions, err := manager.ListSubscriptions(ctx, "my-tenant", "my-namespace", "my-topic")
for _, sub := range subscriptions {
    fmt.Printf("Subscription: %s (type: %s)\n", sub.Name, sub.Type)
}
```

### Get Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("URL: %s\n", status["url"])
fmt.Printf("Brokers: %v\n", status["brokers"])

if status["connected"] == false {
    if errMsg, ok := status["error"].(string); ok {
        fmt.Printf("Error: %s\n", errMsg)
    }
}
```

### Async Job Execution

```go
// Submit custom jobs to the worker pool
manager.SubmitAsyncJob(func() {
    ctx := context.Background()
    brokers, err := manager.GetBrokers(ctx)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    fmt.Printf("Found %d brokers\n", len(brokers))
})
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/hashicorp/go-retryablehttp` | HTTP client with retry logic for API resilience |
| `github.com/spf13/viper` | Configuration management for admin URL and enable/disable |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `context`, `encoding/json`, `bytes`, `io`, `fmt`, `time` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: `testConnection()` validates connectivity on startup via `GetBrokers()`
- **API errors**: `apiRequest()` checks HTTP status codes and returns formatted errors
- **JSON errors**: Marshaling/unmarshaling errors are wrapped with context
- **Context cancellation**: All operations respect `context.Context` for timeouts
- **Nil checks**: Public methods handle nil receiver gracefully in `GetStatus()`

Example error handling:

```go
brokers, err := manager.GetBrokers(ctx)
if err != nil {
    // err contains: "pulsar GET failed: <response body> (status: <code>)"
    return fmt.Errorf("failed to get brokers: %w", err)
}
```

## Common Pitfalls

### 1. Pulsar Not Running

**Problem**: `failed to connect to Pulsar: ...`

**Solution**: 
- Ensure Pulsar is running with Admin API available
- Check Admin API URL (default: `http://localhost:8080`)
- Verify Pulsar installation: `bin/pulsar standalone` for local dev

### 2. Admin API Not Enabled

**Problem**: Connection refused or 404 errors

**Solution**:
- Enable Admin API in `conf/broker.conf`:
  ```
  webServicePort=8080
  ```
- Restart Pulsar broker

### 3. Context Timeout

**Problem**: Operations timing out

**Solution**:
```go
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
// Operations will use this timeout (client has 30s default)
```

### 4. JSON Parsing Errors

**Problem**: `failed to decode <resource>: ...`

**Solution**:
- Check Pulsar version (API responses may vary)
- Verify the resource exists before querying
- Check network/proxy settings if response is HTML error page

### 5. Authentication Required

**Problem**: 401/403 errors

**Solution**:
- Configure authentication in Pulsar (if enabled)
- The current implementation doesn't support auth headers
- May need to extend `apiRequest()` to add token/OAuth headers

## Advanced Usage

### Monitoring Topic Health

```go
func monitorTopic(manager *pulsar.PulsarManager, tenant, namespace, topic string) {
    ctx := context.Background()
    
    stats, err := manager.GetTopicStats(ctx, tenant, namespace, topic)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    // Extract metrics (structure depends on Pulsar version)
    if msgIn, ok := stats["msgRateIn"]; ok {
        fmt.Printf("Message rate in: %v\n", msgIn)
    }
    if msgOut, ok := stats["msgRateOut"]; ok {
        fmt.Printf("Message rate out: %v\n", msgOut)
    }
}
```

### Batch Broker Monitoring

```go
func checkAllBrokers(manager *pulsar.PulsarManager) {
    ctx := context.Background()
    
    brokers, err := manager.GetBrokers(ctx)
    if err != nil {
        fmt.Printf("Failed to get brokers: %v\n", err)
        return
    }
    
    for _, broker := range brokers {
        fmt.Printf("✓ Broker %s is reachable\n", broker.Broker)
        fmt.Printf("  HTTP URL: %s\n", broker.HTTPURL)
        fmt.Printf("  Broker URL: %s\n", broker.URL)
    }
}
```

### Tenant Administration

```go
func listAllResources(manager *pulsar.PulsarManager) {
    ctx := context.Background()
    
    // List all tenants
    tenants, err := manager.ListTenants(ctx)
    if err != nil {
        fmt.Printf("Error listing tenants: %v\n", err)
        return
    }
    
    for _, tenant := range tenants {
        fmt.Printf("\n=== Tenant: %s ===\n", tenant.Name)
        
        // List namespaces for this tenant
        namespaces, err := manager.ListNamespaces(ctx, tenant.Name)
        if err != nil {
            fmt.Printf("  Error listing namespaces: %v\n", err)
            continue
        }
        
        for _, ns := range namespaces {
            fmt.Printf("  Namespace: %s\n", ns.Name)
            
            // List topics in namespace
            topics, err := manager.ListTopics(ctx, tenant.Name, ns.Name)
            if err != nil {
                fmt.Printf("    Error listing topics: %v\n", err)
                continue
            }
            
            for _, topic := range topics {
                fmt.Printf("    Topic: %s\n", topic.Name)
            }
        }
    }
}
```

### Custom API Requests

While the manager provides high-level methods, you can extend it for custom API calls:

```go
// Example: Get cluster information (not directly exposed)
func getClusters(manager *pulsar.PulsarManager, ctx context.Context) ([]string, error) {
    // Use a type assertion or modify the struct to expose apiRequest
    // This is a conceptual example:
    data, err := manager.APIRequest(ctx, "GET", "/clusters", nil)
    if err != nil {
        return nil, err
    }
    
    var clusters []string
    if err := json.Unmarshal(data, &clusters); err != nil {
        return nil, err
    }
    
    return clusters, nil
}
```

## Internal Algorithms

### API Request Construction

```
URL Format: BaseURL + "/admin/v2" + path

Example:
  BaseURL: http://localhost:8080
  Path: /tenants
  Full URL: http://localhost:8080/admin/v2/tenants
```

### Tenant/Namespace/Topic Hierarchy

```
Tenant (e.g., "acme-corp")
  └── Namespace (e.g., "production")
        └── Topic (e.g., "orders")
              └── Subscription (e.g., "order-processor")
```

API paths follow this hierarchy:
- Tenants: `/admin/v2/tenants`
- Namespaces: `/admin/v2/namespaces/{tenant}`
- Topics: `/admin/v2/persistent/{tenant}/{namespace}/{topic}`
- Subscriptions: `/admin/v2/persistent/{tenant}/{namespace}/{topic}/subscription/{sub}`
- Brokers: `/admin/v2/brokers/cluster`

### Retry Logic

Using `go-retryablehttp`:
- Maximum retries: 3
- Minimum wait: 1 second
- Maximum wait: 10 seconds
- HTTP client timeout: 30 seconds

Retries are automatic for transient network errors.

### Response Parsing

The manager uses JSON throughout:

**Tenant list** (returns string array):
```json
["tenant1", "tenant2"]
→ Converted to []PulsarTenant with Name field
```

**Topic stats** (returns object):
```json
{
  "msgRateIn": 100.5,
  "msgRateOut": 98.3,
  "msgBacklog": 150
}
→ Returned as map[string]interface{}
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.