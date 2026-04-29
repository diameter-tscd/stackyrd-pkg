# Elasticsearch Manager

## Overview

The `ElasticsearchManager` is a comprehensive Go library for managing Elasticsearch interactions. It provides full-featured Elasticsearch operations including index management, document CRUD, bulk operations, search, cluster operations, and alias management, using the `github.com/hashicorp/go-retryablehttp` client with retry logic.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Index Management**: Create, delete, check existence, refresh, update mapping, get mapping, list indices
- **Document Operations**: Index (create/update), get, update, delete documents
- **Bulk Operations**: Execute multiple index/update/delete operations in a single request
- **Search Operations**: Full Query DSL search, count documents, scroll search, clear scroll
- **Cluster Operations**: Get cluster health, get cluster statistics
- **Alias Operations**: Add, remove, and list aliases
- **Authentication**: Supports both API key and basic authentication (username/password)
- **Retry Logic**: HTTP client with retry support (3 retries, configurable wait times)
- **Worker Pool**: Async job execution support (8 workers)
- **Async Operations**: Async versions of most operations via `ExecuteAsync`
- **Status Monitoring**: Get connection status and cluster information

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
    
    // Create Elasticsearch manager (configuration via viper)
    manager, err := elastic.NewElasticsearchManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Check if index exists
    exists, err := manager.IndexExists(ctx, "myindex")
    if err != nil {
        panic(err)
    }
    fmt.Printf("Index exists: %v\n", exists)
    
    // Create an index
    if !exists {
        err = manager.CreateIndex(ctx, "myindex", nil, nil)
        if err != nil {
            panic(err)
        }
    }
    
    // Index a document
    doc := map[string]interface{}{
        "title": "Test Document",
        "content": "Hello Elasticsearch",
    }
    err = manager.IndexDocument(ctx, "myindex", "doc1", doc)
    if err != nil {
        panic(err)
    }
    fmt.Println("Document indexed")
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `ElasticsearchManager` | Main manager with HTTP client, base URL, auth config, worker pool |
| `ESIndex` | Index information from cat API (health, status, docs count, store size) |
| `ESSettings` | Index settings (shards, replicas, max result window, refresh interval) |
| `ESMapping` | Index mapping with properties |
| `ESProperty` | Single mapping property (type, analyzer, format, etc.) |
| `ESDocument` | Elasticsearch document with ID, index, source, version |
| `ESSearchRequest` | Search query body (query, from, size, sort, aggs, source) |
| `ESSearchResponse` | Search results (took, timed out, hits, aggregations) |
| `ESHits` | Hits container (total, max score, hits array) |
| `ESHit` | Single search hit (_index, _id, _score, _source) |
| `ESBulkItem` | Single bulk operation (action, index, ID, doc) |
| `ESBulkResponse` | Bulk API response (took, errors, items) |
| `ESClusterHealth` | Cluster health status (cluster name, status, nodes, shards) |
| `ESCountResponse` | Count API response (count) |
| `ESAlias` | Index alias (index, alias) |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   ElasticsearchManager                      │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  retryablehttp │  → HTTP requests with retry            │
│  │  Client       │  → 3 retries, 1-10s wait, 60s timeout│
│  └────────────────┘                                      │
│                                                         │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ AsyncResult │                   │
│  │  (8 workers)│      │   Channel   │                   │
│  └─────────────┘      └──────────────┘                   │
│         ▲                      │                            │
│         │                      ▼                            │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  SubmitJob  │      │ ExecuteAsync │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                         │
│  BaseURL: http://host:port (default: localhost:9200)      │
│  Auth: APIKey or Username/Password                        │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewElasticsearchManager(logger)
    │
    ├── Check viper config: "elasticsearch.enabled"
    ├── Get host: "elasticsearch.host" (default: "localhost")
    ├── Get port: "elasticsearch.port" (default: 9200)
    ├── Get username: "elasticsearch.username"
    ├── Get password: "elasticsearch.password"
    ├── Get api_key: "elasticsearch.api_key"
    │
    ├── Build baseURL: "http://host:port"
    │
    ├── Create retryablehttp.Client:
    │   ├── RetryMax = 3
    │   ├── RetryWaitMin = 1s
    │   ├── RetryWaitMax = 10s
    │   ├── HTTPClient.Timeout = 60s
    │   └── Logger = elasticsearchLoggerAdapter
    │
    ├── Test connection: testConnection()
    │   ├── GET baseURL+"/"
    │   ├── setAuthHeaders(req)
    │   └── Check status == 200
    │
    ├── Create WorkerPool(8)
    ├── Start worker pool
    └── Return ElasticsearchManager
```

### 2. Authentication Flow

```
setAuthHeaders(req)
    │
    ├── If APIKey != "":
    │   └── Set "Authorization: ApiKey <api_key>"
    ├── Else if Username != "" && Password != "":
    │   └── SetBasicAuth(username, password)
    └── Set "Content-Type: application/json"
```

### 3. Index Management Flow

```
CreateIndex(ctx, index, settings, mapping)
    │
    ├── Build payload: {"settings": {"index": settings}, "mappings": mapping}
    ├── Marshal to JSON
    ├── PUT baseURL/index
    ├── setAuthHeaders(req)
    └── Check status: 200 or 201

DeleteIndex(ctx, index)
    │
    ├── DELETE baseURL/index
    ├── setAuthHeaders(req)
    └── Check status: 200

IndexExists(ctx, index)
    │
    ├── HEAD baseURL/index
    ├── setAuthHeaders(req)
    └── Return true if status 200, false if 404

RefreshIndex(ctx, index)
    │
    ├── POST baseURL/index/_refresh
    ├── setAuthHeaders(req)
    └── Check status: 200

PutMapping(ctx, index, mapping)
    │
    ├── Marshal mapping to JSON
    ├── PUT baseURL/index/_mapping
    ├── setAuthHeaders(req)
    └── Check status: 200

GetMapping(ctx, index)
    │
    ├── GET baseURL/index/_mapping
    ├── setAuthHeaders(req)
    ├── Decode response: map[string]{Mappings ESMapping}
    └── Return first mapping found

ListIndices(ctx)
    │
    ├── GET baseURL/_cat/indices?format=json
    ├── setAuthHeaders(req)
    ├── Decode to []ESIndex
    └── Return indices
```

### 4. Document Operations Flow

```
IndexDocument(ctx, index, id, doc)
    │
    ├── Marshal doc to JSON
    ├── PUT baseURL/index/_doc/id?refresh=wait_for
    ├── setAuthHeaders(req)
    └── Check status: 200 or 201

GetDocument(ctx, index, id)
    │
    ├── GET baseURL/index/_doc/id
    ├── setAuthHeaders(req)
    ├── If status 404: return Found=false
    ├── Decode to ESDocument
    └── Return document

UpdateDocument(ctx, index, id, doc)
    │
    ├── Build payload: {"doc": doc}
    ├── Marshal to JSON
    ├── POST baseURL/index/_update/id?refresh=wait_for
    ├── setAuthHeaders(req)
    └── Check status: 200

DeleteDocument(ctx, index, id)
    │
    ├── DELETE baseURL/index/_doc/id
    ├── setAuthHeaders(req)
    └── Check status: 200
```

### 5. Bulk Operations Flow

```
Bulk(ctx, items)
    │
    ├── buildBulkBody(items)
    │   └── For each item: write meta JSON + "\n" + doc JSON (if not nil) + "\n"
    ├── POST baseURL/_bulk
    ├── Content-Type: application/x-ndjson
    ├── setAuthHeaders(req)
    ├── Decode response to ESBulkResponse
    └── Return response
```

### 6. Search Operations Flow

```
Search(ctx, index, request)
    │
    ├── Marshal ESSearchRequest to JSON
    ├── POST baseURL/index/_search (or baseURL/_search if index empty)
    ├── setAuthHeaders(req)
    ├── Decode response to ESSearchResponse
    └── Return response

Count(ctx, index, query)
    │
    ├── If query != nil: build body {"query": query}
    ├── GET baseURL/index/_count (or baseURL/_count)
    ├── setAuthHeaders(req)
    ├── Decode to ESCountResponse
    └── Return count

Scroll(ctx, scrollID, duration)
    │
    ├── Build payload: {"scroll": "duration_ms", "scroll_id": scrollID}
    ├── POST baseURL/_search/scroll
    ├── setAuthHeaders(req)
    ├── Decode to ESSearchResponse
    └── Return response

ClearScroll(ctx, scrollID)
    │
    ├── Build payload: {"scroll_id": scrollID}
    ├── DELETE baseURL/_search/scroll
    ├── setAuthHeaders(req)
    └── Check status: 200
```

### 7. Cluster Operations Flow

```
ClusterHealth(ctx)
    │
    ├── GET baseURL/_cluster/health
    ├── setAuthHeaders(req)
    ├── Decode to ESClusterHealth
    └── Return health

ClusterStats(ctx)
    │
    ├── GET baseURL/_cluster/stats
    ├── setAuthHeaders(req)
    ├── Decode to map[string]interface{}
    └── Return stats
```

### 8. Alias Operations Flow

```
AddAlias(ctx, index, alias)
    │
    ├── Build payload: {"actions": [{"add": {"index": index, "alias": alias}}]}
    ├── POST baseURL/_aliases
    ├── setAuthHeaders(req)
    └── Check status: 200

RemoveAlias(ctx, index, alias)
    │
    ├── Build payload: {"actions": [{"remove": {"index": index, "alias": alias}}]}
    ├── POST baseURL/_aliases
    ├── setAuthHeaders(req)
    └── Check status: 200

GetAliases(ctx, index)
    │
    ├── GET baseURL/index/_alias (or baseURL/_alias if index empty)
    ├── setAuthHeaders(req)
    ├── Decode to map[string]{Aliases map[string]interface{}}
    ├── Build []ESAlias from result
    └── Return aliases
```

### 9. Get Status Flow

```
GetStatus()
    │
    ├── If nil: return connected=false
    ├── Call ClusterHealth with 5s timeout
    ├── If error: return connected=false, error
    ├── Return connected=true, cluster_name, status, nodes, data_nodes, pool_active
    └── Note: Tests actual connection via cluster health API
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `elasticsearch.enabled` | bool | false | Enable/disable Elasticsearch manager |
| `elasticsearch.host` | string | "localhost" | Elasticsearch host |
| `elasticsearch.port` | int | 9200 | Elasticsearch port |
| `elasticsearch.username` | string | "" | Username for basic auth |
| `elasticsearch.password` | string | "" | Password for basic auth |
| `elasticsearch.api_key` | string | "" | API key for authentication |

### Environment Variables

The Elasticsearch manager does not directly use environment variables. Configuration is read through Viper, which can load from:
- Configuration files (JSON, YAML, TOML)
- Environment variables (if Viper is configured for it)

You can set Elasticsearch parameters via:
```go
viper.Set("elasticsearch.enabled", true)
viper.Set("elasticsearch.host", "localhost")
viper.Set("elasticsearch.port", 9200)
viper.Set("elasticsearch.username", "elastic")
viper.Set("elasticsearch.password", "changeme")
viper.Set("elasticsearch.api_key", "")
```

## Usage Examples

### Creating an Index with Settings and Mapping

```go
ctx := context.Background()

// Define settings
settings := &elastic.ESSettings{
    NumberOfShards:   1,
    NumberOfReplicas: 0,
    MaxResultWindow:  10000,
    RefreshInterval:  "1s",
}

// Define mapping
mapping := &elastic.ESMapping{
    Properties: map[string]elastic.ESProperty{
        "title": {
            Type: "text",
        },
        "content": {
            Type: "text",
            Analyzer: "standard",
        },
        "created_at": {
            Type: "date",
            Format: "strict_date_optional_time",
        },
    },
}

err := manager.CreateIndex(ctx, "myindex", settings, mapping)
if err != nil {
    panic(err)
}
fmt.Println("Index created")
```

### Indexing a Document

```go
ctx := context.Background()

doc := map[string]interface{}{
    "title": "My First Document",
    "content": "This is the content of my document.",
    "created_at": time.Now().Format(time.RFC3339),
}

err := manager.IndexDocument(ctx, "myindex", "doc1", doc)
if err != nil {
    panic(err)
}
fmt.Println("Document indexed")
```

### Getting a Document

```go
ctx := context.Background()

doc, err := manager.GetDocument(ctx, "myindex", "doc1")
if err != nil {
    panic(err)
}

if !doc.Found {
    fmt.Println("Document not found")
} else {
    fmt.Printf("Document: %v\n", doc.Source)
    fmt.Printf("Version: %d\n", doc.Version)
}
```

### Updating a Document

```go
ctx := context.Background()

update := map[string]interface{}{
    "content": "Updated content",
}

err := manager.UpdateDocument(ctx, "myindex", "doc1", update)
if err != nil {
    panic(err)
}
fmt.Println("Document updated")
```

### Deleting a Document

```go
ctx := context.Background()

err := manager.DeleteDocument(ctx, "myindex", "doc1")
if err != nil {
    panic(err)
}
fmt.Println("Document deleted")
```

### Bulk Operations

```go
ctx := context.Background()

items := []elastic.ESBulkItem{
    {
        Action: "index",
        Index:  "myindex",
        ID:     "doc1",
        Doc:    map[string]interface{}{"title": "Doc 1", "content": "Content 1"},
    },
    {
        Action: "index",
        Index:  "myindex",
        ID:     "doc2",
        Doc:    map[string]interface{}{"title": "Doc 2", "content": "Content 2"},
    },
    {
        Action: "delete",
        Index:  "myindex",
        ID:     "doc3",
    },
}

result, err := manager.Bulk(ctx, items)
if err != nil {
    panic(err)
}

fmt.Printf("Bulk took %d ms, errors: %v\n", result.Took, result.Errors)
```

### Searching

```go
ctx := context.Background()

// Build a search request
request := elastic.ESSearchRequest{
    Query: map[string]interface{}{
        "match": map[string]interface{}{
            "content": "document",
        },
    },
    From: 0,
    Size: 10,
    Sort: []map[string]interface{}{
        {"_score": map[string]interface{}{"order": "desc"}},
    },
}

response, err := manager.Search(ctx, "myindex", request)
if err != nil {
    panic(err)
}

fmt.Printf("Found %d documents (took %d ms)\n", response.Hits.Total.Value, response.Took)
for _, hit := range response.Hits.Hits {
    fmt.Printf("  - %s: %v\n", hit.ID, hit.Source)
}
```

### Counting Documents

```go
ctx := context.Background()

query := map[string]interface{}{
    "match": map[string]interface{}{
        "content": "document",
    },
}

count, err := manager.Count(ctx, "myindex", query)
if err != nil {
    panic(err)
}
fmt.Printf("Count: %d\n", count)
```

### Using Scroll

```go
ctx := context.Background()

// Initial search with scroll
request := elastic.ESSearchRequest{
    Query: map[string]interface{}{"match_all": map[string]interface{}{}},
    Size:  100,
}

response, err := manager.Search(ctx, "myindex", request)
if err != nil {
    panic(err)
}

scrollID := response.ScrollID
fmt.Printf("First batch: %d hits\n", len(response.Hits.Hits))

// Continue scrolling
for scrollID != "" {
    scrollResp, err := manager.Scroll(ctx, scrollID, 1*time.Minute)
    if err != nil {
        break
    }
    fmt.Printf("Next batch: %d hits\n", len(scrollResp.Hits.Hits))
    scrollID = scrollResp.ScrollID
}

// Clear scroll
if scrollID != "" {
    manager.ClearScroll(ctx, scrollID)
}
```

### Cluster Health

```go
ctx := context.Background()

health, err := manager.ClusterHealth(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Cluster: %s\n", health.ClusterName)
fmt.Printf("Status: %s\n", health.Status)
fmt.Printf("Nodes: %d, Data Nodes: %d\n", health.NumberOfNodes, health.NumberOfDataNodes)
fmt.Printf("Active Shards: %d, Unassigned: %d\n", health.ActiveShards, health.UnassignedShards)
```

### Cluster Stats

```go
ctx := context.Background()

stats, err := manager.ClusterStats(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Cluster stats: %v\n", stats)
```

### Alias Operations

```go
ctx := context.Background()

// Add alias
err := manager.AddAlias(ctx, "myindex", "myalias")
if err != nil {
    panic(err)
}
fmt.Println("Alias added")

// List aliases
aliases, err := manager.GetAliases(ctx, "myindex")
if err != nil {
    panic(err)
}
for _, a := range aliases {
    fmt.Printf("Alias: %s -> %s\n", a.Alias, a.Index)
}

// Remove alias
err = manager.RemoveAlias(ctx, "myindex", "myalias")
if err != nil {
    panic(err)
}
fmt.Println("Alias removed")
```

### Getting Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
if status["connected"] == true {
    fmt.Printf("Cluster: %s\n", status["cluster_name"])
    fmt.Printf("Status: %s\n", status["status"])
    fmt.Printf("Nodes: %v\n", status["nodes"])
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

- **Connection failures**: Returned from `testConnection()` (health check)
- **HTTP errors**: Returned from various operations via `handleErrorResponse()`
- **JSON errors**: Returned when marshaling/unmarshaling fails
- **Nil checks**: `GetStatus()` handles nil receiver gracefully

Example error handling:

```go
resp, err := manager.Search(ctx, index, request)
if err != nil {
    if strings.Contains(err.Error(), "connection refused") {
        return fmt.Errorf("elasticsearch not reachable: %w", err)
    }
    if strings.Contains(err.Error(), "index_not_found_exception") {
        return fmt.Errorf("index not found: %w", err)
    }
    if strings.Contains(err.Error(), "parsing_exception") {
        return fmt.Errorf("invalid query: %w", err)
    }
    return fmt.Errorf("search failed: %w", err)
}
```

## Common Pitfalls

### 1. Elasticsearch Not Reachable

**Problem**: `connection refused` or `no route to host` errors

**Solution**: 
- Verify Elasticsearch is running
- Check host/port in `elasticsearch.host` and `elasticsearch.port`
- Default port is 9200
- Check firewall settings

### 2. Authentication Failed

**Problem**: `unauthorized` or `authentication failed` errors

**Solution**:
- Verify API key or username/password
- Check if authentication is required
- For API key, ensure it's set in `elasticsearch.api_key`
- For basic auth, ensure both username and password are set

### 3. Index Not Found

**Problem**: `index_not_found_exception` errors

**Solution**:
- Check index name
- Create index if it doesn't exist using `CreateIndex()`
- Verify index name case (Elasticsearch index names are lowercase by convention)

### 4. Mapping Issues

**Problem**: `mapper_parsing_exception` or similar errors

**Solution**:
- Ensure mapping is valid
- Check field types and analyzers
- Use `PutMapping()` to update mapping after index creation (some settings can't be changed)

### 5. Timeout Issues

**Problem**: Operations timing out

**Solution**:
- The default HTTP client timeout is 60 seconds
- For long operations (e.g., large bulk), consider increasing timeout
- Note: Timeout is set in retryablehttp client, not configurable via viper in current implementation

### 6. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewElasticsearchManager()` succeeded
- Verify `elasticsearch.enabled` is true

### 7. Refresh Not Happening

**Problem**: Recently indexed documents not appearing in search

**Solution**:
- Use `refresh=wait_for` parameter (already used in `IndexDocument`)
- Or call `RefreshIndex()` explicitly
- Or set `refresh_interval` in index settings

## Advanced Usage

### Building Complex Queries

```go
func buildComplexQuery() map[string]interface{} {
    return map[string]interface{}{
        "bool": map[string]interface{}{
            "must": []map[string]interface{}{
                {"match": map[string]interface{}{"title": "elasticsearch"}},
                {"range": map[string]interface{}{
                    "created_at": map[string]interface{}{
                        "gte": "now-1d/d",
                    },
                }},
            },
            "filter": []map[string]interface{}{
                {"term": map[string]interface{}{"status": "published"}},
            },
        },
    }
}

func searchComplex(manager *elastic.ElasticsearchManager) error {
    ctx := context.Background()
    
    request := elastic.ESSearchRequest{
        Query: buildComplexQuery(),
        Size:  20,
        Sort:  []map[string]interface{}{{"created_at": "desc"}},
        Aggs: map[string]interface{}{
            "by_status": map[string]interface{}{
                "terms": map[string]interface{}{"field": "status"},
            },
        },
    }
    
    response, err := manager.Search(ctx, "myindex", request)
    if err != nil {
        return err
    }
    
    fmt.Printf("Aggregations: %v\n", response.Aggs)
    return nil
}
```

### Using Async Operations

```go
func asyncIndex(manager *elastic.ElasticsearchManager) {
    ctx := context.Background()
    
    // Async index document
    result := manager.IndexDocumentAsync(ctx, "myindex", "doc1", map[string]interface{}{
        "title": "Async Doc",
    })
    
    select {
    case <-result.Ch:
        fmt.Println("Document indexed asynchronously")
    case err := <-result.ErrCh:
        fmt.Printf("Error: %v\n", err)
    }
}
```

### Health Check with Retries

```go
func healthCheck(manager *elastic.ElasticsearchManager) error {
    ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
    defer cancel()
    
    status := manager.GetStatus()
    if status["connected"] == false {
        return fmt.Errorf("not connected to Elasticsearch")
    }
    
    // Additional check: cluster health
    health, err := manager.ClusterHealth(ctx)
    if err != nil {
        return err
    }
    
    if health.Status == "red" {
        return fmt.Errorf("cluster status is red")
    }
    
    fmt.Printf("Cluster status: %s\n", health.Status)
    return nil
}
```

### Bulk Indexing with Error Handling

```go
func bulkIndex(manager *elastic.ElasticsearchManager, docs []map[string]interface{}) error {
    ctx := context.Background()
    
    var items []elastic.ESBulkItem
    for i, doc := range docs {
        items = append(items, elastic.ESBulkItem{
            Action: "index",
            Index:  "myindex",
            ID:     fmt.Sprintf("doc%d", i),
            Doc:    doc,
        })
    }
    
    result, err := manager.Bulk(ctx, items)
    if err != nil {
        return err
    }
    
    if result.Errors {
        for _, item := range result.Items {
            if item.Index != nil && item.Index.Status >= 400 {
                fmt.Printf("Error indexing %s: %s\n", item.Index.ID, item.Index.Error.Reason)
            }
        }
    }
    
    return nil
}
```

## Internal Algorithms

### Bulk Body Construction

```
buildBulkBody(items):
    │
    ├── Create bytes.Buffer
    ├── For each item:
    │   ├── Build meta: {item.Action: {"_index": item.Index, "_id": item.ID}}
    │   ├── Marshal meta to JSON
    │   ├── Write meta JSON + "\n"
    │   ├── If item.Doc != nil:
    │   │   ├── Marshal doc to JSON
    │   │   └── Write doc JSON + "\n"
    │   └── (If delete, no doc)
    └── Return buffer as io.Reader
```

### Error Response Handling

```
handleErrorResponse(resp):
    │
    ├── Read body: io.ReadAll(resp.Body)
    └── Return formatted error with body and status code
```

### Connection Test

```
testConnection():
    │
    ├── GET baseURL+"/"
    ├── setAuthHeaders(req)
    ├── Execute request
    ├── If status != 200: return error
    └── Return nil
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.