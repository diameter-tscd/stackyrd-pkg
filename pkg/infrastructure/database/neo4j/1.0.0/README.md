# Neo4j Manager

## Overview

The `Neo4jManager` is a Go library for managing Neo4j graph database operations via HTTP API. It provides Cypher query execution, node and relationship management, and database listing using the `github.com/hashicorp/go-retryablehttp` client with retry logic.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Cypher Query Execution**: Run Cypher queries with parameters
- **Node Operations**: Create nodes with labels and properties, get node by ID, delete node
- **Relationship Operations**: Create relationships between nodes with type and properties
- **Database Management**: List databases
- **HTTP API**: Uses Neo4j HTTP API (transactional endpoint)
- **Retry Logic**: HTTP client with retry support (3 retries, configurable wait times)
- **Worker Pool**: Async job execution support (6 workers)
- **Status Monitoring**: Get connection status and database information
- **Authentication**: Supports basic authentication (username/password)

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
    
    // Create Neo4j manager (configuration via viper)
    manager, err := neo4j.NewNeo4jManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Create a node
    labels := []string{"Person"}
    props := map[string]interface{}{"name": "John", "age": 30}
    node, err := manager.CreateNode(ctx, labels, props)
    if err != nil {
        panic(err)
    }
    fmt.Printf("Created node with ID: %d\n", node.ID)
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `Neo4jManager` | Main manager with HTTP client, base URL, auth config, worker pool |
| `Neo4jNode` | Graph node with ID, labels, properties |
| `Neo4jRelationship` | Graph relationship with ID, type, start/end nodes, properties |
| `Neo4jQueryResult` | Query result with columns and data |
| `Neo4jDatabaseInfo` | Database info (name, status, default) |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   Neo4jManager                           │
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
│  BaseURL: http://host:port (default: http://localhost:7474) │
│  Auth: Username/Password                                │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewNeo4jManager(logger)
    │
    ├── Check viper config: "neo4j.enabled"
    ├── Get baseURL: "neo4j.url" (default: "http://localhost:7474")
    ├── Get username: "neo4j.username"
    ├── Get password: "neo4j.password"
    ├── Get database: "neo4j.database" (default: "neo4j")
    │
    ├── Create retryablehttp.Client:
    │   ├── RetryMax = 3
    │   ├── RetryWaitMin = 1s
    │   ├── RetryWaitMax = 10s
    │   └── HTTPClient.Timeout = 60s
    │
    ├── Test connection: testConnection()
    │   └── ListDatabases(ctx)
    │
    ├── Create WorkerPool(6)
    ├── Start worker pool
    └── Return Neo4jManager
```

### 2. API Request Flow

```
apiRequest(ctx, method, path, body)
    │
    ├── Build URL: baseURL + path
    ├── If body != nil: Marshal to JSON, create reader
    ├── Create request: retryablehttp.NewRequestWithContext(ctx, method, url, bodyReader)
    ├── setAuthHeader(req)
    │   ├── Set Content-Type: application/json
    │   ├── Set Accept: application/json;charset=UTF-8
    │   └── If username/password: SetBasicAuth()
    ├── Execute: Client.Do(req)
    ├── Read response body
    ├── If status < 200 or >= 300: return error with body
    └── Return response body bytes
```

### 3. Query Execution Flow

```
RunQuery(ctx, cypher, params)
    │
    ├── Build payload: {"statements": [{"statement": cypher, "parameters": params}]}
    ├── POST to "/db/" + database + "/tx/commit"
    ├── apiRequest(ctx, "POST", path, payload)
    ├── Unmarshal response to struct with Results and Errors
    ├── If len(Errors) > 0: return error
    ├── If len(Results) > 0: return Results[0]
    └── Return empty Neo4jQueryResult
```

### 4. Node Operations Flow

```
CreateNode(ctx, labels, properties)
    │
    ├── Build label string: ":Label1:Label2"
    ├── Build Cypher: "CREATE (n:Label $props) RETURN n"
    ├── RunQuery(ctx, cypher, {"props": properties})
    └── Return Neo4jNode{Labels: labels, Properties: properties}

GetNode(ctx, nodeID)
    │
    ├── Build Cypher: "MATCH (n) WHERE id(n) = $id RETURN n"
    ├── RunQuery(ctx, cypher, {"id": nodeID})
    └── Return Neo4jNode{ID: nodeID}

DeleteNode(ctx, nodeID)
    │
    ├── Build Cypher: "MATCH (n) WHERE id(n) = $id DELETE n"
    ├── RunQuery(ctx, cypher, {"id": nodeID})
    └── Return error
```

### 5. Relationship Operations Flow

```
CreateRelationship(ctx, startID, endID, relType, properties)
    │
    ├── Build Cypher: "MATCH (a), (b) WHERE id(a) = $startID AND id(b) = $endID CREATE (a)-[r:Type $props]->(b) RETURN r"
    ├── RunQuery(ctx, cypher, {"startID": startID, "endID": endID, "props": properties})
    └── Return Neo4jRelationship{Type: relType, StartNode: startID, EndNode: endID, Properties: properties}
```

### 6. List Databases Flow

```
ListDatabases(ctx)
    │
    ├── GET "/db/manage/server/jmx/domain/org.neo4j/instance%3Dkernel%230%2Cname%3DDiagnostics"
    ├── apiRequest(ctx, "GET", path, nil)
    ├── Unmarshal to struct with Databases
    └── Return []Neo4jDatabaseInfo
```

### 7. Get Status Flow

```
GetStatus()
    │
    ├── If nil: return connected=false
    ├── Call ListDatabases with 5s timeout
    ├── If error: return connected=false, error
    ├── Return connected=true, url, databases (count), pool_active
    └── Note: Tests actual connection via list databases API
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `neo4j.enabled` | bool | false | Enable/disable Neo4j manager |
| `neo4j.url` | string | "http://localhost:7474" | Neo4j HTTP API URL |
| `neo4j.username` | string | "" | Username for basic auth |
| `neo4j.password` | string | "" | Password for basic auth |
| `neo4j.database` | string | "neo4j" | Database name |

### Environment Variables

Configuration is read through Viper, which can load from files or environment variables.

Example programmatic configuration:

```go
viper.Set("neo4j.enabled", true)
viper.Set("neo4j.url", "http://localhost:7474")
viper.Set("neo4j.username", "neo4j")
viper.Set("neo4j.password", "password")
viper.Set("neo4j.database", "neo4j")
```

## Usage Examples

### Creating a Node

```go
ctx := context.Background()

labels := []string{"Person", "Employee"}
props := map[string]interface{}{
    "name": "Alice",
    "age":  28,
    "email": "alice@example.com",
}

node, err := manager.CreateNode(ctx, labels, props)
if err != nil {
    panic(err)
}
fmt.Printf("Created node ID: %d\n", node.ID)
fmt.Printf("Labels: %v\n", node.Labels)
fmt.Printf("Properties: %v\n", node.Properties)
```

### Getting a Node

```go
ctx := context.Background()

// Note: This is a simplified example; actual implementation may vary
node, err := manager.GetNode(ctx, 123)
if err != nil {
    panic(err)
}
fmt.Printf("Node: %v\n", node)
```

### Deleting a Node

```go
ctx := context.Background()

err := manager.DeleteNode(ctx, 123)
if err != nil {
    panic(err)
}
fmt.Println("Node deleted")
```

### Creating a Relationship

```go
ctx := context.Background()

// Create a relationship between two nodes
rel, err := manager.CreateRelationship(ctx, 123, 456, "KNOWS", map[string]interface{}{
    "since": "2023",
})
if err != nil {
    panic(err)
}
fmt.Printf("Created relationship: %s (from %d to %d)\n", rel.Type, rel.StartNode, rel.EndNode)
```

### Running a Cypher Query

```go
ctx := context.Background()

// Query nodes
cypher := "MATCH (n:Person) WHERE n.age > $minAge RETURN n.name as name, n.age as age"
params := map[string]interface{}{"minAge": 21}

result, err := manager.RunQuery(ctx, cypher, params)
if err != nil {
    panic(err)
}

fmt.Printf("Columns: %v\n", result.Columns)
for _, data := range result.Data {
    fmt.Printf("Data: %v\n", data)
}
```

### Listing Databases

```go
ctx := context.Background()

databases, err := manager.ListDatabases(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Found %d databases:\n", len(databases))
for _, db := range databases {
    fmt.Printf("  - %s (status: %s, default: %v)\n", db.Name, db.Status, db.Default)
}
```

### Getting Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
if status["connected"] == true {
    fmt.Printf("URL: %s\n", status["url"])
    fmt.Printf("Databases: %v\n", status["databases"])
}
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/hashicorp/go-retryablehttp` | HTTP client with retry logic |
| `github.com/spf13/viper` | Configuration management |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `bytes`, `context`, `encoding/json`, `fmt`, `io`, `time` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: Returned from `testConnection()` (ListDatabases)
- **HTTP errors**: Returned from `apiRequest()` with response body
- **Query errors**: Returned from `RunQuery()` if Neo4j returns errors
- **Nil checks**: `GetStatus()` handles nil receiver gracefully

Example error handling:

```go
result, err := manager.RunQuery(ctx, cypher, params)
if err != nil {
    if strings.Contains(err.Error(), "connection refused") {
        return fmt.Errorf("neo4j not reachable: %w", err)
    }
    if strings.Contains(err.Error(), "authentication failure") {
        return fmt.Errorf("authentication failed: %w", err)
    }
    if strings.Contains(err.Error(), "syntax error") {
        return fmt.Errorf("invalid cypher query: %w", err)
    }
    return fmt.Errorf("query failed: %w", err)
}
```

## Common Pitfalls

### 1. Neo4j Not Reachable

**Problem**: `connection refused` or `no route to host` errors

**Solution**: 
- Verify Neo4j is running
- Check URL in `neo4j.url`
- Default port is 7474 (HTTP API)
- Check firewall settings

### 2. Authentication Failed

**Problem**: `authentication failure` or `unauthorized` errors

**Solution**:
- Verify username and password
- Check if authentication is required
- Default credentials are often neo4j/password for fresh installs

### 3. Database Not Found

**Problem**: `database not found` errors

**Solution**:
- Check database name in `neo4j.database`
- Default is "neo4j"
- Verify database exists using `ListDatabases()`

### 4. Cypher Syntax Errors

**Problem**: `syntax error` in Cypher queries

**Solution**:
- Check Cypher syntax
- Use parameters instead of string concatenation
- Test queries in Neo4j Browser first

### 5. Timeout Issues

**Problem**: Operations timing out

**Solution**:
- The default HTTP client timeout is 60 seconds
- For long queries, consider increasing timeout
- Note: Timeout is set in retryablehttp client, not configurable via viper in current implementation

### 6. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewNeo4jManager()` succeeded
- Verify `neo4j.enabled` is true

## Advanced Usage

### Complex Cypher Queries

```go
func complexQuery(manager *neo4j.Neo4jManager) error {
    ctx := context.Background()
    
    // Query with multiple conditions and relationships
    cypher := `
        MATCH (p:Person)-[:KNOWS]->(friend)
        WHERE p.age > $minAge AND friend.active = $active
        RETURN p.name as name, count(friend) as friendCount
        ORDER BY friendCount DESC
    `
    params := map[string]interface{}{
        "minAge": 21,
        "active": true,
    }
    
    result, err := manager.RunQuery(ctx, cypher, params)
    if err != nil {
        return err
    }
    
    fmt.Printf("Results: %v\n", result.Data)
    return nil
}
```

### Creating a Graph

```go
func createGraph(manager *neo4j.Neo4jManager) error {
    ctx := context.Background()
    
    // Create nodes
    node1, err := manager.CreateNode(ctx, []string{"Person"}, map[string]interface{}{"name": "Alice"})
    if err != nil {
        return err
    }
    
    node2, err := manager.CreateNode(ctx, []string{"Person"}, map[string]interface{}{"name": "Bob"})
    if err != nil {
        return err
    }
    
    // Create relationship
    _, err = manager.CreateRelationship(ctx, node1.ID, node2.ID, "KNOWS", map[string]interface{}{"since": 2023})
    if err != nil {
        return err
    }
    
    fmt.Println("Graph created")
    return nil
}
```

### Health Check

```go
func healthCheck(manager *neo4j.Neo4jManager) error {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    status := manager.GetStatus()
    if status["connected"] == false {
        return fmt.Errorf("not connected to Neo4j")
    }
    
    // Additional check: list databases
    databases, err := manager.ListDatabases(ctx)
    if err != nil {
        return err
    }
    
    fmt.Printf("Neo4j healthy, %d databases available\n", len(databases))
    return nil
}
```

### Using Parameters Safely

```go
func safeQuery(manager *neo4j.Neo4jManager, userInput string) error {
    ctx := context.Background()
    
    // Always use parameters to avoid Cypher injection
    cypher := "MATCH (n:Person) WHERE n.name = $name RETURN n"
    params := map[string]interface{}{"name": userInput}
    
    _, err := manager.RunQuery(ctx, cypher, params)
    return err
}
```

## Internal Algorithms

### API Request Construction

```
apiRequest():
    │
    ├── Build URL: baseURL + path
    ├── Set headers (Content-Type, Accept)
    ├── If username/password: SetBasicAuth
    ├── Execute HTTP request via retryablehttp
    ├── Check status code (200-299 = success)
    └── Return body bytes or error
```

### Query Execution

```
RunQuery():
    │
    ├── Build JSON payload with statement and parameters
    ├── POST to transactional endpoint
    ├── Parse response for results and errors
    └── Return first result or error
```

### Connection Test

```
testConnection():
    │
    └── ListDatabases(ctx)
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.