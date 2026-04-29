# MongoDB Manager

## Overview

The `MongoManager` is a Go library for managing MongoDB operations. It provides comprehensive MongoDB CRUD operations, aggregation, collection management, and supports multiple connections via `MongoConnectionManager`. It uses the official MongoDB Go driver (`go.mongodb.org/mongo-driver`) with configurable timeouts and a worker pool for async operations.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Single Connection**: `MongoManager` for a single MongoDB database connection
- **Multiple Connections**: `MongoConnectionManager` for managing multiple named MongoDB connections
- **CRUD Operations**: InsertOne, InsertMany, FindOne, Find, UpdateOne, UpdateMany, DeleteOne, DeleteMany
- **Aggregation**: Perform aggregation pipelines
- **Collection Management**: ListCollections, CreateCollection, DropCollection
- **Database Info**: GetDBInfo with stats and server status
- **Raw Query Execution**: Execute raw queries and get results as slice of maps
- **Helper Functions**: StringToObjectID, StringToFloat, StringToStringSlice
- **Async Operations**: Async versions of most operations via `ExecuteAsync`
- **Batch Operations**: Batch insert and update operations via `ExecuteBatchAsync`
- **Worker Pool**: Async job execution support (12 workers per connection)
- **Status Monitoring**: Get connection status, database stats, server info
- **Connection Pool**: Configurable connection pool settings (max idle time, heartbeat, etc.)

## Quick Start

```go
package main

import (
    "context"
    "fmt"
    "stackyrd/config"
    "stackyrd/pkg/infrastructure"
    "stackyrd/pkg/logger"
    "go.mongodb.org/mongo-driver/bson"
)

func main() {
    // Initialize logger
    log := logger.NewLogger()
    
    // Configuration (typically via viper/config)
    cfg := config.MongoConfig{
        Enabled:  true,
        URI:      "mongodb://localhost:27017",
        Database: "mydb",
    }
    
    // Create MongoDB manager
    manager, err := mongo.NewMongoDB(cfg, log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Insert a document
    doc := bson.M{"name": "John", "age": 30}
    result, err := manager.InsertOne(ctx, "users", doc)
    if err != nil {
        panic(err)
    }
    fmt.Printf("Inserted document with ID: %v\n", result.InsertedID)
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `MongoManager` | Single MongoDB connection with client, database, worker pool |
| `MongoConnectionManager` | Manages multiple named `MongoManager` connections |
| `MongoConfig` | Configuration for a single connection (URI, database, enabled) |
| `MongoMultiConfig` | Configuration for multiple connections |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   MongoManager                              │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  mongo.Client │  → Connection to MongoDB              │
│  │  mongo.Database│  → Specific database                  │
│  └────────────────┘                                      │
│                                                         │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ AsyncResult │                   │
│  │  (12 workers)│     │   Channel   │                   │
│  └─────────────┘      └──────────────┘                   │
│         ▲                      │                            │
│         │                      ▼                            │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  SubmitJob  │      │ ExecuteAsync │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                         │
│  URI: mongodb://host:port                                │
│  Database: database name                                  │
└─────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────┐
│              MongoConnectionManager                         │
├─────────────────────────────────────────────────────────────┤
│  connections: map[string]*MongoManager                    │
│  mu: sync.RWMutex                                       │
│                                                         │
│  Methods:                                                │
│  - GetConnection(name) → *MongoManager, bool             │
│  - GetDefaultConnection() → *MongoManager, bool          │
│  - GetAllConnections() → map[string]*MongoManager        │
│  - CloseAll() → error                                    │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Single Connection Initialization Flow

```
NewMongoDB(cfg, logger)
    │
    ├── Check cfg.Enabled
    ├── Create context with 10s timeout
    ├── Set client options:
    │   ├── ApplyURI(cfg.URI)
    │   ├── SetConnectTimeout(10s)
    │   ├── SetServerSelectionTimeout(5s)
    │   ├── SetSocketTimeout(10s)
    │   ├── SetMaxConnIdleTime(30s)
    │   ├── SetHeartbeatInterval(10s)
    │   └── SetReadPreference(PrimaryPreferred)
    │
    ├── Connect to MongoDB: mongo.Connect(ctx, clientOptions)
    ├── Ping with 5s timeout: client.Ping(pingCtx, Primary())
    ├── If ping fails: disconnect and return error
    ├── Get database: client.Database(cfg.Database)
    ├── Create WorkerPool(12)
    ├── Start worker pool
    └── Return MongoManager
```

### 2. Multiple Connections Initialization Flow

```
NewMongoConnectionManager(cfg, logger)
    │
    ├── Check cfg.Enabled
    ├── Create MongoConnectionManager with empty connections map
    ├── For each connCfg in cfg.Connections:
    │   ├── Skip if !connCfg.Enabled
    │   ├── Convert to MongoConfig
    │   ├── Call NewMongoDB(connCfg, logger)
    │   ├── If error: log and continue
    │   └── Store in connections[connCfg.Name]
    │
    └── Return MongoConnectionManager
```

### 3. CRUD Operations Flow

```
InsertOne(ctx, collection, document)
    │
    ├── Get collection: Database.Collection(collection)
    └── coll.InsertOne(ctx, document)

InsertMany(ctx, collection, documents)
    │
    ├── Get collection
    └── coll.InsertMany(ctx, documents)

FindOne(ctx, collection, filter)
    │
    ├── Get collection
    └── coll.FindOne(ctx, filter) → *mongo.SingleResult

Find(ctx, collection, filter)
    │
    ├── Get collection
    └── coll.Find(ctx, filter) → *mongo.Cursor

UpdateOne(ctx, collection, filter, update)
    │
    ├── Get collection
    └── coll.UpdateOne(ctx, filter, update) → *mongo.UpdateResult

UpdateMany(ctx, collection, filter, update)
    │
    ├── Get collection
    └── coll.UpdateMany(ctx, filter, update) → *mongo.UpdateResult

DeleteOne(ctx, collection, filter)
    │
    ├── Get collection
    └── coll.DeleteOne(ctx, filter) → *mongo.DeleteResult

DeleteMany(ctx, collection, filter)
    │
    ├── Get collection
    └── coll.DeleteMany(ctx, filter) → *mongo.DeleteResult
```

### 4. Aggregation Flow

```
Aggregate(ctx, collection, pipeline)
    │
    ├── Get collection
    └── coll.Aggregate(ctx, pipeline) → *mongo.Cursor
```

### 5. Collection Management Flow

```
ListCollections(ctx)
    │
    └── Database.ListCollectionNames(ctx, map[string]interface{}{}) → []string

CreateCollection(ctx, name)
    │
    └── Database.CreateCollection(ctx, name)

DropCollection(ctx, name)
    │
    ├── Get collection: Database.Collection(name)
    └── coll.Drop(ctx)
```

### 6. Database Info Flow

```
GetDBInfo(ctx)
    │
    ├── Run dbStats command: Database.RunCommand(ctx, {"dbStats": 1})
    ├── Decode to map[string]interface{}
    ├── Run serverStatus command: client.Database("admin").RunCommand(ctx, {"serverStatus": 1})
    ├── Get list of collections: ListCollections(ctx)
    └── Return map with database name, collections, stats, server_info
```

### 7. Get Status Flow

```
GetStatus()
    │
    ├── If nil or Client == nil: return connected=false
    ├── Ping: Client.Ping(context.Background(), nil)
    ├── If error: return connected=false
    ├── Run dbStats command
    ├── Decode result to get db_name, collections, objects, data_size, storage_size, indexes, index_size
    └── Return stats map
```

### 8. Connection Manager Status Flow

```
MongoConnectionManager.GetStatus()
    │
    ├── For each connection in connections:
    │   └── Call conn.GetStatus()
    └── Return map of statuses
```

## Configuration

### Configuration Structs

```go
type MongoConfig struct {
    Enabled  bool   `mapstructure:"enabled"`
    URI      string `mapstructure:"uri"`
    Database string `mapstructure:"database"`
}

type MongoMultiConfig struct {
    Enabled     bool           `mapstructure:"enabled"`
    Connections []MongoConfig `mapstructure:"connections"`
}
```

### Viper Configuration Keys

For single connection:
- `mongo.enabled` (bool)
- `mongo.uri` (string) - e.g., "mongodb://localhost:27017"
- `mongo.database` (string)

For multiple connections:
- `mongo-multi.enabled` (bool)
- `mongo-multi.connections` (list of maps with keys: `name`, `enabled`, `uri`, `database`)

### Environment Variables

Configuration is read through Viper, which can load from files or environment variables.

Example programmatic configuration:

```go
cfg := config.MongoConfig{
    Enabled:  true,
    URI:      "mongodb://localhost:27017",
    Database: "mydb",
}
```

## Usage Examples

### Single Connection

```go
ctx := context.Background()

// Insert a document
doc := bson.M{"name": "Alice", "email": "alice@example.com"}
result, err := manager.InsertOne(ctx, "users", doc)
if err != nil {
    panic(err)
}
fmt.Printf("Inserted ID: %v\n", result.InsertedID)
```

### Finding Documents

```go
ctx := context.Background()

// Find one document
filter := bson.M{"name": "Alice"}
singleResult := manager.FindOne(ctx, "users", filter)
var user bson.M
if err := singleResult.Decode(&user); err != nil {
    panic(err)
}
fmt.Printf("User: %v\n", user)

// Find multiple documents
cursor, err := manager.Find(ctx, "users", bson.M{})
if err != nil {
    panic(err)
}
defer cursor.Close(ctx)

var users []bson.M
if err := cursor.All(ctx, &users); err != nil {
    panic(err)
}
fmt.Printf("Found %d users\n", len(users))
```

### Updating Documents

```go
ctx := context.Background()

filter := bson.M{"name": "Alice"}
update := bson.M{"$set": bson.M{"age": 25}}

result, err := manager.UpdateOne(ctx, "users", filter, update)
if err != nil {
    panic(err)
}
fmt.Printf("Matched: %d, Modified: %d\n", result.MatchedCount, result.ModifiedCount)
```

### Deleting Documents

```go
ctx := context.Background()

filter := bson.M{"name": "Alice"}
result, err := manager.DeleteOne(ctx, "users", filter)
if err != nil {
    panic(err)
}
fmt.Printf("Deleted %d document(s)\n", result.DeletedCount)
```

### Aggregation

```go
ctx := context.Background()

pipeline := mongo.Pipeline{
    {{"$group", bson.M{"_id": "$status", "count": bson.M{"$sum": 1}}}},
}
cursor, err := manager.Aggregate(ctx, "users", pipeline)
if err != nil {
    panic(err)
}
defer cursor.Close(ctx)

var results []bson.M
if err := cursor.All(ctx, &results); err != nil {
    panic(err)
}
for _, res := range results {
    fmt.Printf("Status: %v, Count: %v\n", res["_id"], res["count"])
}
```

### Listing Collections

```go
ctx := context.Background()

collections, err := manager.ListCollections(ctx)
if err != nil {
    panic(err)
}
fmt.Println("Collections:")
for _, coll := range collections {
    fmt.Printf("  - %s\n", coll)
}
```

### Getting Database Info

```go
ctx := context.Background()

info, err := manager.GetDBInfo(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Database: %s\n", info["database"])
fmt.Printf("Collections: %v\n", info["collections"])
fmt.Printf("Stats: %v\n", info["stats"])
```

### Using Multiple Connections

```go
// Assuming you have a MongoConnectionManager
connManager, err := mongo.NewMongoConnectionManager(cfg, log)
if err != nil {
    panic(err)
}

// Get a specific connection
conn, exists := connManager.GetConnection("primary")
if !exists {
    panic("connection not found")
}

// Use the connection
result, err := conn.InsertOne(ctx, "users", bson.M{"test": 1})
```

### Async Operations

```go
ctx := context.Background()

// Async insert
asyncResult := manager.InsertOneAsync(ctx, "users", bson.M{"async": true})
select {
case result := <-asyncResult.Ch:
    fmt.Printf("Async inserted: %v\n", result.InsertedID)
case err := <-asyncResult.ErrCh:
    fmt.Printf("Error: %v\n", err)
}
```

### Batch Operations

```go
ctx := context.Background()

inserts := []struct {
    Collection string
    Document   interface{}
}{
    {Collection: "users", Document: bson.M{"name": "Batch1"}},
    {Collection: "users", Document: bson.M{"name": "Batch2"}},
}

batchResult := manager.InsertBatchAsync(ctx, inserts)
// Wait for all operations
for i, opResult := range batchResult.Results {
    if opResult.Error != nil {
        fmt.Printf("Insert %d failed: %v\n", i, opResult.Error)
    } else {
        fmt.Printf("Insert %d succeeded\n", i)
    }
}
```

### Helper Functions

```go
// Convert string to ObjectID
id, err := mongo.StringToObjectID("507f1f77bcf86cd799439011")
if err != nil {
    panic(err)
}
fmt.Printf("ObjectID: %v\n", id)

// Convert string to float64
f := mongo.StringToFloat("3.14")
fmt.Printf("Float: %f\n", f)

// Convert comma-separated string to slice
slice := mongo.StringToStringSlice("a, b, c")
fmt.Printf("Slice: %v\n", slice)
```

### Getting Status

```go
// For single connection
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
if status["connected"] == true {
    fmt.Printf("Database: %v\n", status["db_name"])
    fmt.Printf("Collections: %v\n", status["collections"])
    fmt.Printf("Objects: %v\n", status["objects"])
}

// For connection manager
connStatus := connManager.GetStatus()
for name, stat := range connStatus {
    fmt.Printf("Connection %s: %v\n", name, stat)
}
```

## Dependencies

| Dependency | Role |
|------------|------|
| `go.mongodb.org/mongo-driver/mongo` | Official MongoDB driver |
| `go.mongodb.org/mongo-driver/bson` | BSON handling |
| `go.mongodb.org/mongo-driver/mongo/options` | Client options |
| `go.mongodb.org/mongo-driver/mongo/readpref` | Read preference |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `context`, `fmt`, `strconv`, `strings`, `sync`, `time` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: Returned from `mongo.Connect()` or `client.Ping()`
- **Operation failures**: Returned from driver methods
- **Nil checks**: `GetStatus()` handles nil receiver gracefully

Example error handling:

```go
result, err := manager.InsertOne(ctx, collection, doc)
if err != nil {
    if strings.Contains(err.Error(), "connection refused") {
        return fmt.Errorf("mongodb not reachable: %w", err)
    }
    if strings.Contains(err.Error(), "not authorized") {
        return fmt.Errorf("authentication failed: %w", err)
    }
    if strings.Contains(err.Error(), "no reachable servers") {
        return fmt.Errorf("no mongodb servers reachable: %w", err)
    }
    return fmt.Errorf("insert failed: %w", err)
}
```

## Common Pitfalls

### 1. MongoDB Not Reachable

**Problem**: `connection refused` or `no reachable servers` errors

**Solution**: 
- Verify MongoDB is running
- Check URI in configuration
- Default port is 27017
- Check firewall settings

### 2. Authentication Failed

**Problem**: `not authorized` or `Authentication failed` errors

**Solution**:
- Verify username and password in URI (e.g., `mongodb://user:pass@host:port`)
- Check if authentication is required
- Ensure user has access to the database

### 3. Database Not Found

**Problem**: `database not found` errors

**Solution**:
- Check database name in configuration
- The database will be created automatically on first write, but ensure the name is valid

### 4. Timeout Issues

**Problem**: Operations timing out

**Solution**:
- The default timeouts are: ConnectTimeout 10s, ServerSelectionTimeout 5s, SocketTimeout 10s
- Adjust via client options if needed (currently set in `NewMongoDB`)

### 5. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewMongoDB()` succeeded
- Verify `mongo.enabled` is true

### 6. Connection Leak

**Problem**: Not closing connections

**Solution**:
- Always call `manager.Close()` or `connManager.CloseAll()`
- Use `defer` to ensure cleanup

### 7. Using ObjectIDs

**Problem**: Issues with MongoDB ObjectIDs

**Solution**:
- Use `mongo.StringToObjectID()` to convert string to ObjectID
- When querying by `_id`, use ObjectID type, not string
- Example: `filter := bson.M{"_id": objectID}`

## Advanced Usage

### Complex Queries

```go
func complexQuery(manager *mongo.MongoManager) error {
    ctx := context.Background()
    
    // Query with multiple conditions
    filter := bson.M{
        "age": bson.M{"$gte": 18, "$lte": 65},
        "status": "active",
        "tags": bson.M{"$in": []string{"go", "mongodb"}},
    }
    
    cursor, err := manager.Find(ctx, "users", filter)
    if err != nil {
        return err
    }
    defer cursor.Close(ctx)
    
    var users []bson.M
    return cursor.All(ctx, &users)
}
```

### Transaction Example

Note: Transactions require replica set.

```go
func transactionExample(manager *mongo.MongoManager) error {
    ctx := context.Background()
    
    session, err := manager.Client.StartSession()
    if err != nil {
        return err
    }
    defer session.EndSession(ctx)
    
    _, err = session.WithTransaction(ctx, func(sessCtx mongo.SessionContext) (interface{}, error) {
        // Operation 1
        _, err := manager.InsertOne(sessCtx, "accounts", bson.M{"balance": 100})
        if err != nil {
            return nil, err
        }
        // Operation 2
        _, err = manager.UpdateOne(sessCtx, "accounts", bson.M{"balance": 100}, bson.M{"$inc": bson.M{"balance": -50}})
        if err != nil {
            return nil, err
        }
        return nil, nil
    })
    
    return err
}
```

### Health Check

```go
func healthCheck(manager *mongo.MongoManager) error {
    ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
    defer cancel()
    
    status := manager.GetStatus()
    if status["connected"] == false {
        return fmt.Errorf("not connected to MongoDB")
    }
    
    // Additional check: ping
    err := manager.Client.Ping(ctx, nil)
    if err != nil {
        return fmt.Errorf("ping failed: %w", err)
    }
    
    fmt.Printf("MongoDB connection healthy\n")
    return nil
}
```

### Raw Query Execution

```go
func rawQuery(manager *mongo.MongoManager) error {
    ctx := context.Background()
    
    query := bson.M{"status": "active"}
    results, err := manager.ExecuteRawQuery(ctx, "users", query)
    if err != nil {
        return err
    }
    
    fmt.Printf("Found %d documents\n", len(results))
    for _, doc := range results {
        fmt.Printf("  %v\n", doc)
    }
    return nil
}
```

## Internal Algorithms

### Connection Test

```
Ping():
    │
    └── client.Ping(context.Background(), readpref.Primary())
```

### Database Stats

```
GetDBInfo():
    │
    ├── Run "dbStats" command
    ├── Decode to map
    ├── Run "serverStatus" command on admin database
    ├── Get list of collections
    └── Return combined map
```

### Batch Async Operations

```
InsertBatchAsync(ctx, inserts):
    │
    ├── Create []AsyncOperation[*mongo.InsertOneResult]
    ├── For each insert: create function calling InsertOne
    └── ExecuteBatchAsync(ctx, operations)
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.