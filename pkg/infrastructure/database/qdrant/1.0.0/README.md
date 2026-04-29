# Qdrant Manager

## Overview

The `QdrantManager` is a Go library for managing Qdrant vector database operations via HTTP API. It provides vector upsert and search operations using the `github.com/hashicorp/go-retryablehttp` client with retry logic.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Vector Operations**: Upsert points (insert/update) and search by vector similarity
- **Collection Management**: Implicit via API (collections must be created separately)
- **HTTP API**: Uses Qdrant HTTP REST API
- **Retry Logic**: HTTP client with retry support (3 retries, configurable wait times)
- **Worker Pool**: Async job execution support (6 workers)
- **Status Monitoring**: Get connection status and basic info
- **Authentication**: Supports API key authentication via header
- **Async Operations**: Async versions of upsert and search via `ExecuteAsync`

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
    
    // Create Qdrant manager (configuration via viper)
    manager, err := qdrant.NewQdrantManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Upsert a vector point
    points := []qdrant.QdrantPoint{
        {
            ID:     "point1",
            Vector: []float32{0.1, 0.2, 0.3, 0.4},
            Payload: map[string]interface{}{"label": "sample"},
        },
    }
    err = manager.UpsertPoints(ctx, "mycollection", points)
    if err != nil {
        panic(err)
    }
    fmt.Println("Points upserted")
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `QdrantManager` | Main manager with HTTP client, base URL, API key, worker pool |
| `QdrantPoint` | Vector point with ID, vector, payload |
| `QdrantSearchRequest` | Search query with vector, top, filter, payload/vector flags |
| `QdrantSearchResult` | Search result with ID, score, payload, vector |
| `QdrantCollectionInfo` | Collection info (name, vector size, distance, points count, status) |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   QdrantManager                           │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  retryablehttp │  → HTTP requests with retry            │
│  │  Client       │  → 3 retries, 1-10s wait, 60s timeout│
│  └────────────────┘                                      │
│                                                         │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ AsyncResult │                   │
│  │  (6 workers)│      │   Channel   │                   │
│  └─────────────┘      └──────────────┘                   │
│         ▲                      │                            │
│         │                      ▼                            │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  SubmitJob  │      │ ExecuteAsync │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                         │
│  BaseURL: http://host:port (default: http://localhost:6333) │
│  Auth: API Key (header: api-key)                         │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewQdrantManager(logger)
    │
    ├── Check viper config: "qdrant.enabled"
    ├── Get host: "qdrant.host"
    ├── Get port: "qdrant.port"
    ├── Get apiKey: "qdrant.api_key"
    ├── Build baseURL: "http://host:port"
    │
    ├── Create retryablehttp.Client:
    │   ├── RetryMax = 3
    │   ├── RetryWaitMin = 1s
    │   ├── RetryWaitMax = 10s
    │   ├── HTTPClient.Timeout = 60s
    │   └── Logger = qdrantLoggerAdapter
    │
    ├── Test connection: testConnection()
    │   ├── GET baseURL+"/"
    │   ├── Set api-key header if APIKey != ""
    │   └── Check status == 200
    │
    ├── Create WorkerPool(6)
    ├── Start worker pool
    └── Return QdrantManager
```

### 2. Upsert Points Flow

```
UpsertPoints(ctx, collection, points)
    │
    ├── Build payload: {"points": points}
    ├── Marshal to JSON
    ├── PUT baseURL+"/collections/"+collection+"/points"
    ├── Set Content-Type: application/json
    ├── Set api-key header if APIKey != ""
    ├── Execute request via retryablehttp
    ├── If status != 200: return error with body
    └── Return nil
```

### 3. Search Flow

```
Search(ctx, collection, request)
    │
    ├── Marshal request to JSON
    ├── POST baseURL+"/collections/"+collection+"/points/search"
    ├── Set Content-Type: application/json
    ├── Set api-key header if APIKey != ""
    ├── Execute request
    ├── If status != 200: return error with body
    ├── Decode response to struct with Result []QdrantSearchResult
    └── Return result.Result
```

### 4. Get Status Flow

```
GetStatus()
    │
    ├── If nil: return connected=false
    ├── Create GET request to baseURL+"/" with 5s timeout
    ├── Set api-key header if APIKey != ""
    ├── Execute request
    ├── Return connected = (status == 200), base_url, pool_active
    └── Note: Tests actual connection via root endpoint
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `qdrant.enabled` | bool | false | Enable/disable Qdrant manager |
| `qdrant.host` | string | "localhost" | Qdrant host |
| `qdrant.port` | int | 6333 | Qdrant port (HTTP) |
| `qdrant.api_key` | string | "" | API key for authentication |

### Environment Variables

Configuration is read through Viper, which can load from files or environment variables.

Example programmatic configuration:

```go
viper.Set("qdrant.enabled", true)
viper.Set("qdrant.host", "localhost")
viper.Set("qdrant.port", 6333)
viper.Set("qdrant.api_key", "your-api-key")
```

## Usage Examples

### Upserting Points

```go
ctx := context.Background()

points := []qdrant.QdrantPoint{
    {
        ID:     "point1",
        Vector: []float32{0.1, 0.2, 0.3, 0.4, 0.5},
        Payload: map[string]interface{}{"label": "sample", "category": "test"},
    },
    {
        ID:     "point2",
        Vector: []float32{0.2, 0.3, 0.4, 0.5, 0.6},
        Payload: map[string]interface{}{"label": "another"},
    },
}

err := manager.UpsertPoints(ctx, "mycollection", points)
if err != nil {
    panic(err)
}
fmt.Println("Points upserted")
```

### Searching Vectors

```go
ctx := context.Background()

request := qdrant.QdrantSearchRequest{
    Vector:      []float32{0.1, 0.2, 0.3, 0.4, 0.5},
    Top:         10,
    WithPayload: true,
    WithVector:  false,
}

results, err := manager.Search(ctx, "mycollection", request)
if err != nil {
    panic(err)
}

fmt.Printf("Found %d results:\n", len(results))
for _, res := range results {
    fmt.Printf("  ID: %s, Score: %f, Payload: %v\n", res.ID, res.Score, res.Payload)
}
```

### Search with Filter

```go
ctx := context.Background()

// Search with filter (example: payload field "category" equals "test")
request := qdrant.QdrantSearchRequest{
    Vector: []float32{0.1, 0.2, 0.3, 0.4, 0.5},
    Top:    5,
    Filter: map[string]interface{}{
        "must": []map[string]interface{}{
            {"key": "category", "match": map[string]interface{}{"value": "test"}},
        },
    },
    WithPayload: true,
}

results, err := manager.Search(ctx, "mycollection", request)
if err != nil {
    panic(err)
}
fmt.Printf("Found %d filtered results\n", len(results))
```

### Async Operations

```go
ctx := context.Background()

// Async upsert
points := []qdrant.QdrantPoint{
    {ID: "async1", Vector: []float32{0.7, 0.8, 0.9, 1.0, 1.1}},
}
asyncResult := manager.UpsertPointsAsync(ctx, "mycollection", points)
select {
case <-asyncResult.Ch:#    fmt.Println("Async upsert completed")
case err := <-asyncResult.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async search
searchReq := qdrant.QdrantSearchRequest{
    Vector: []float32{0.1, 0.2, 0.3, 0.4, 0.5},
    Top:    5,
}
searchResult := manager.SearchAsync(ctx, "mycollection", searchReq)
select {
case results := <-searchResult.Ch:#    fmt.Printf("Async search found %d results\n", len(results))
case err := <-searchResult.ErrCh:#    fmt.Printf("Error: %v\n", err)
}
```

### Getting Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
if status["connected"] == true {
    fmt.Printf("Base URL: %v\n", status["base_url"])
}
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/hashicorp/go-retryablehttp` | HTTP client with retry logic |
| `github.com/spf13/viper` | Configuration management |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `bytes`, `context`, `encoding/json`, `fmt`, `io`, `net/http`, `time` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: Returned from `testConnection()` (root endpoint check)
- **HTTP errors**: Returned from API requests with response body
- **Nil checks**: `GetStatus()` handles nil receiver gracefully

Example error handling:

```go
err := manager.UpsertPoints(ctx, collection, points)
if err != nil {
    if strings.Contains(err.Error(), "connection refused") {
        return fmt.Errorf("qdrant not reachable: %w", err)
    }
    if strings.Contains(err.Error(), "unauthorized") {
        return fmt.Errorf("authentication failed: %w", err)
    }
    if strings.Contains(err.Error(), "not found") {
        return fmt.Errorf("collection not found: %w", err)
    }
    return fmt.Errorf("upsert failed: %w", err)
}
```

## Common Pitfalls

### 1. Qdrant Not Reachable

**Problem**: `connection refused` or `no route to host` errors

**Solution**: 
- Verify Qdrant is running
- Check host/port in configuration
- Default port is 6333 (HTTP)
- Check firewall settings

### 2. Authentication Failed

**Problem**: `unauthorized` or `authentication failed` errors

**Solution**:
- Verify API key in `qdrant.api_key`
- Check if API key is required
- Ensure the API key is valid

### 3. Collection Not Found

**Problem**: `not found` errors when upserting/searching

**Solution**:
- Create the collection first via Qdrant API or UI
- Verify collection name
- Check if collection exists using Qdrant's REST API

### 4. Vector Dimension Mismatch

**Problem**: Errors about vector size mismatch

**Solution**:
- Ensure all vectors have the same dimension
- Check collection's vector size configuration
- The vector size is set when creating the collection

### 5. Timeout Issues

**Problem**: Operations timing out

**Solution**:
- The default HTTP client timeout is 60 seconds
- For large operations, consider increasing timeout
- Note: Timeout is set in retryablehttp client, not configurable via viper in current implementation

### 6. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewQdrantManager()` succeeded
- Verify `qdrant.enabled` is true

## Advanced Usage

### Batch Upsert with Error Handling

```go
func batchUpsert(manager *qdrant.QdrantManager, collection string) error {
    ctx := context.Background()
    
    var allPoints []qdrant.QdrantPoint
    for i := 0; i < 100; i++ {
        point := qdrant.QdrantPoint{
            ID:     fmt.Sprintf("point%d", i),
            Vector: []float32{float32(i) * 0.01, float32(i) * 0.02, float32(i) * 0.03},
            Payload: map[string]interface{}{"index": i},
        }
        allPoints = append(allPoints, point)
    }
    
    // Upsert in batches if needed (Qdrant may have limits)
    err := manager.UpsertPoints(ctx, collection, allPoints)
    return err
}
```

### Advanced Search with Params

```go
func advancedSearch(manager *qdrant.QdrantManager, collection string) error {
    ctx := context.Background()
    
    request := qdrant.QdrantSearchRequest{
        Vector:      []float32{0.1, 0.2, 0.3, 0.4, 0.5},
        Top:         20,
        WithPayload: true,
        WithVector:  true,
        Params: map[string]interface{}{
            "hnsw_ef": 128, // Optional HNSW parameter
        },
    }
    
    results, err := manager.Search(ctx, collection, request)
    if err != nil {
        return err
    }
    
    fmt.Printf("Advanced search found %d results\n", len(results))
    for _, res := range results {
        fmt.Printf("  %s: score=%f, vector=%v\n", res.ID, res.Score, res.Vector)
    }
    return nil
}
```

### Health Check

```go
func healthCheck(manager *qdrant.QdrantManager) error {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    status := manager.GetStatus()
    if status["connected"] == false {
        return fmt.Errorf("not connected to Qdrant")
    }
    
    fmt.Println("Qdrant connection healthy")
    return nil
}
```

### Using Filters

```go
func searchWithRangeFilter(manager *qdrant.QdrantManager, collection string) error {
    ctx := context.Background()
    
    // Search with range filter on payload field "price"
    request := qdrant.QdrantSearchRequest{
        Vector: []float32{0.1, 0.2, 0.3},
        Top:    10,
        Filter: map[string]interface{}{
            "must": []map[string]interface{}{
                {
                    "key": "price",
                    "range": map[string]interface{}{
                        "gte": 100,
                        "lte": 500,
                    },
                },
            },
        },
        WithPayload: true,
    }
    
    results, err := manager.Search(ctx, collection, request)
    if err != nil {
        return err
    }
    
    fmt.Printf("Found %d results in price range\n", len(results))
    return nil
}
```

## Internal Algorithms

### Connection Test

```
testConnection():
    │
    ├── GET baseURL+"/"
    ├── Set api-key header if APIKey != ""
    ├── Execute request
    ├── If status != 200: return error
    └── Return nil
```

### Upsert Operation

```
UpsertPoints():
    │
    ├── Build JSON payload with points
    ├── PUT to collections/{collection}/points
    ├── Set headers (Content-Type, api-key)
    ├── Execute HTTP request via retryablehttp
    ├── Check status code (200 = success)
    └── Return nil or error
```

### Search Operation

```
Search():
    │
    ├── Build JSON request with vector, top, filter, flags
    ├── POST to collections/{collection}/points/search
    ├── Set headers
    ├── Execute request
    ├── Decode response for results
    └── Return []QdrantSearchResult
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.