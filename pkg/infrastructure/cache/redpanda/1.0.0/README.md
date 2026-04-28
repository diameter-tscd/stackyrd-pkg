# Redpanda Manager

## Overview

The `RedpandaManager` is a Go library for managing Redpanda (Kafka-compatible) operations via the Redpanda Admin REST API. It provides a comprehensive interface for managing topics, partitions, consumer groups, and brokers with built-in retry logic and async job support.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Topic Management**: Full CRUD operations on topics with partition and replication configuration
- **Partition Management**: Query partition details, leaders, replicas, and ISR
- **Consumer Group Management**: List and monitor consumer groups
- **Broker Management**: List broker instances with host/port details
- **Cluster Health**: Check cluster readiness and health status
- **Kafka Compatibility**: API compatible with Kafka ecosystems
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
    
    // Create Redpanda manager (configuration via viper)
    manager, err := redpanda.NewRedpandaManager(log)
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
        fmt.Printf("Broker ID: %d, Host: %s:%d\n", broker.ID, broker.Host, broker.Port)
    }
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `RedpandaManager` | Main manager with retryable HTTP client and worker pool |
| `RedpandaTopic` | Represents a topic with partition count and replication factor |
| `RedpandaPartition` | Represents a partition with leader, replicas, and ISR info |
| `RedpandaBroker` | Represents a broker instance with host/port/rack |
| `RedpandaConsumerGroup` | Represents a consumer group with state and members |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   RedpandaManager                          │
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
│  BaseURL: Redpanda Admin API URL (default: http://localhost:9644)│
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewRedpandaManager()
    │
    ├── Check viper config: "redpanda.enabled"
    ├── Get admin URL: "redpanda.admin_url" (default: "http://localhost:9644")
    ├── Create retryablehttp.Client (max 3 retries, 1-10s wait, 30s timeout)
    ├── Test connection: manager.testConnection() → GetBrokers()
    ├── Create WorkerPool(6)
    └── Return RedpandaManager
```

### 2. API Request Flow

```
apiRequest(ctx, method, path, body)
    │
    ├── Build URL: BaseURL + path
    ├── If body: JSON marshal → bytes.Reader
    ├── Create retryablehttp.NewRequestWithContext()
    ├── Set Content-Type: application/json
    ├── Execute client.Do(req)
    ├── Read response body
    ├── Check status code (200-299 = success)
    └── Return response bytes or error
```

### 3. Topic Listing Flow

```
ListTopics(ctx)
    │
    ├── apiRequest(ctx, "GET", "/v1/topics", nil)
    ├── JSON unmarshal → []RedpandaTopic
    └── Return topics
```

### 4. Topic Creation Flow

```
CreateTopic(ctx, topic)
    │
    ├── apiRequest(ctx, "POST", "/v1/topics", topic)
    ├── If status 200-299: return nil
    └── Return error with response body
```

### 5. Partition Query Flow

```
GetPartitions(ctx, topicName)
    │
    ├── Build path: "/v1/topics/{topicName}/partitions"
    ├── apiRequest(ctx, "GET", path, nil)
    ├── JSON unmarshal → []RedpandaPartition
    └── Return partitions
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `redpanda.enabled` | bool | false | Enable/disable Redpanda manager |
| `redpanda.admin_url` | string | "http://localhost:9644" | Redpanda Admin API URL |

### Environment Variables

The Redpanda manager does not directly use environment variables. Configuration is read through Viper, which can load from:
- Configuration files (JSON, YAML, TOML)
- Environment variables (if Viper is configured for it)
- Command-line flags

## Usage Examples

### Basic Operations

```go
ctx := context.Background()

// List brokers
brokers, err := manager.GetBrokers(ctx)
if err != nil {
    panic(err)
}
for _, b := range brokers {
    fmt.Printf("Broker ID: %d, Address: %s:%d\n", b.ID, b.Host, b.Port)
    if b.Rack != "" {
        fmt.Printf("  Rack: %s\n", b.Rack)
    }
}

// List topics
topics, err := manager.ListTopics(ctx)
if err != nil {
    panic(err)
}
for _, t := range topics {
    fmt.Printf("Topic: %s (partitions: %d, replication: %d)\n", 
        t.Name, t.Partitions, t.Replication)
}

// Get a specific topic
topic, err := manager.GetTopic(ctx, "my-topic")
if err != nil {
    panic(err)
}
fmt.Printf("Topic: %s\n", topic.Name)
fmt.Printf("  Partitions: %d\n", topic.Partitions)
fmt.Printf("  Replication: %d\n", topic.Replication)
```

### Topic Management

```go
// Create a topic
err := manager.CreateTopic(ctx, redpanda.RedpandaTopic{
    Name:       "orders",
    Partitions: 3,
    Replication: 2,
    Config: map[string]string{
        "retention.ms": "604800000", // 7 days
    },
})
if err != nil {
    fmt.Printf("Failed to create topic: %v\n", err)
}

// Get partitions for a topic
partitions, err := manager.GetPartitions(ctx, "orders")
if err != nil {
    panic(err)
}
for _, p := range partitions {
    fmt.Printf("Partition %d: leader=%d, replicas=%v, isr=%v\n",
        p.Partition, p.Leader, p.Replicas, p.ISR)
}

// Delete a topic
err = manager.DeleteTopic(ctx, "old-topic")
if err != nil {
    fmt.Printf("Failed to delete topic: %v\n", err)
}
```

### Consumer Group Operations

```go
// List consumer groups
groups, err := manager.ListConsumerGroups(ctx)
if err != nil {
    panic(err)
}
for _, g := range groups {
    fmt.Printf("Group: %s (state: %s, protocol: %s, members: %d)\n",
        g.ID, g.State, g.Protocol, g.Members)
}
```

### Cluster Health

```go
// Get cluster health status
health, err := manager.GetClusterHealth(ctx)
if err != nil {
    fmt.Printf("Cluster not healthy: %v\n", err)
} else {
    fmt.Printf("Cluster health: %+v\n", health)
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
    topics, err := manager.ListTopics(ctx)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    fmt.Printf("Found %d topics\n", len(topics))
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
topics, err := manager.ListTopics(ctx)
if err != nil {
    // err contains: "redpanda GET failed: <response body> (status: <code>)"
    return fmt.Errorf("failed to list topics: %w", err)
}
```

## Common Pitfalls

### 1. Redpanda Not Running

**Problem**: `failed to connect to Redpanda: ...`

**Solution**: 
- Ensure Redpanda is running with Admin API available
- Check Admin API URL (default: `http://localhost:9644`)
- Start Redpanda: `redpanda start` or use Docker image

### 2. Admin API Not Enabled

**Problem**: Connection refused or 404 errors

**Solution**:
- The Admin API is enabled by default on port 9644
- Check Redpanda configuration: `redpanda.yaml`
- Verify: `curl http://localhost:9644/v1/brokers`

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
- Check Redpanda version (API responses may vary)
- Verify the resource exists before querying
- Check network/proxy settings if response is HTML error page

### 5. Topic Already Exists

**Problem**: CreateTopic fails with conflict

**Solution**:
- Check if topic exists first: `manager.GetTopic(ctx, name)`
- Use update operations if you need to modify configuration
- Delete and recreate if necessary (caution: data loss)

### 6. Partition Count and Replication

**Problem**: Topic creation fails with invalid configuration

**Solution**:
- Ensure replication factor ≤ number of brokers
- Partitions should be appropriate for throughput needs
- Common config: 3 partitions, replication factor 2-3

## Advanced Usage

### Monitoring Topic Health

```go
func monitorTopics(manager *redpanda.RedpandaManager) {
    ctx := context.Background()
    
    topics, err := manager.ListTopics(ctx)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    for _, topic := range topics {
        fmt.Printf("\n=== Topic: %s ===\n", topic.Name)
        fmt.Printf("  Partitions: %d\n", topic.Partitions)
        fmt.Printf("  Replication: %d\n", topic.Replication)
        
        // Get partitions
        partitions, err := manager.GetPartitions(ctx, topic.Name)
        if err != nil {
            fmt.Printf("  Error getting partitions: %v\n", err)
            continue
        }
        
        for _, p := range partitions {
            fmt.Printf("  Partition %d: leader=%d, isr=%v\n",
                p.Partition, p.Leader, p.ISR)
        }
    }
}
```

### Cluster Health Monitoring

```go
func checkClusterHealth(manager *redpanda.RedpandaManager) bool {
    ctx := context.Background()
    
    // Check cluster readiness
    health, err := manager.GetClusterHealth(ctx)
    if err != nil {
        fmt.Printf("Cluster health check failed: %v\n", err)
        return false
    }
    
    fmt.Printf("Cluster Status: %+v\n", health)
    
    // Check brokers
    brokers, err := manager.GetBrokers(ctx)
    if err != nil {
        fmt.Printf("Failed to get brokers: %v\n", err)
        return false
    }
    
    fmt.Printf("Active brokers: %d\n", len(brokers))
    
    // Check consumer groups
    groups, err := manager.ListConsumerGroups(ctx)
    if err != nil {
        fmt.Printf("Failed to list groups: %v\n", err)
        return false
    }
    
    fmt.Printf("Consumer groups: %d\n", len(groups))
    
    return true
}
```

### Topic Administration

```go
func administerTopics(manager *redpanda.RedpandaManager) {
    ctx := context.Background()
    
    // List all topics
    topics, err := manager.ListTopics(ctx)
    if err != nil {
        fmt.Printf("Error listing topics: %v\n", err)
        return
    }
    
    fmt.Printf("Found %d topics:\n", len(topics))
    
    for _, topic := range topics {
        fmt.Printf("\n- %s\n", topic.Name)
        
        // Get detailed topic info
        detailed, err := manager.GetTopic(ctx, topic.Name)
        if err != nil {
            fmt.Printf("  Error: %v\n", err)
            continue
        }
        
        fmt.Printf("  Partitions: %d\n", detailed.Partitions)
        fmt.Printf("  Replication: %d\n", detailed.Replication)
        
        if detailed.Config != nil {
            fmt.Printf("  Config:\n")
            for k, v := range detailed.Config {
                fmt.Printf("    %s = %s\n", k, v)
            }
        }
    }
}
```

### Custom API Requests

While the manager provides high-level methods, you can extend it for custom API calls:

```go
// Example: Get broker configuration (not directly exposed)
func getBrokerConfig(manager *redpanda.RedpandaManager, ctx context.Context, brokerID int) (map[string]interface{}, error) {
    // This is a conceptual example - modify the struct to expose apiRequest
    data, err := manager.APIRequest(ctx, "GET", fmt.Sprintf("/v1/brokers/%d/config", brokerID), nil)
    if err != nil {
        return nil, err
    }
    
    var config map[string]interface{}
    if err := json.Unmarshal(data, &config); err != nil {
        return nil, err
    }
    
    return config, nil
}
```

## Internal Algorithms

### API Request Construction

```
URL Format: BaseURL + path

Example:
  BaseURL: http://localhost:9644
  Path: /v1/topics
  Full URL: http://localhost:9644/v1/topics
```

### Topic/Partition Hierarchy

```
Cluster
  └── Topic (e.g., "orders")
        ├── Partition 0 (leader: broker1, replicas: [1,2], ISR: [1,2])
        ├── Partition 1 (leader: broker2, replicas: [2,3], ISR: [2,3])
        └── Partition 2 (leader: broker3, replicas: [3,1], ISR: [3,1])
              └── Consumer Group (e.g., "order-processor")
```

API paths follow Kafka-compatible patterns:
- Brokers: `/v1/brokers`
- Topics: `/v1/topics`
- Topic details: `/v1/topics/{topic}`
- Partitions: `/v1/topics/{topic}/partitions`
- Consumer groups: `/v1/groups`
- Cluster status: `/v1/status/ready`

### Retry Logic

Using `go-retryablehttp`:
- Maximum retries: 3
- Minimum wait: 1 second
- Maximum wait: 10 seconds
- HTTP client timeout: 30 seconds

Retries are automatic for transient network errors.

### Response Parsing

The manager uses JSON throughout:

**Topic list** (returns array):
```json
[
  {"name": "orders", "partitions_count": 3, "replication_factor": 2},
  {"name": "events", "partitions_count": 6, "replication_factor": 3}
]
→ Unmarshaled to []RedpandaTopic
```

**Broker list** (returns array):
```json
[
  {"node_id": 1, "host": "broker1", "port": 9092, "rack": "us-east"},
  {"node_id": 2, "host": "broker2", "port": 9092, "rack": "us-west"}
]
→ Unmarshaled to []RedpandaBroker
```

**Cluster health** (returns object):
```json
{"status": "ready"}
→ Returned as map[string]interface{}
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.