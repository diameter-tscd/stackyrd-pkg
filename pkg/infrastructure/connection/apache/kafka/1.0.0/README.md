# Kafka Manager

## Overview

The `KafkaManager` is a Go library for managing Apache Kafka operations using the IBM Sarama client library. It provides both producer and consumer functionality with support for publishing messages (with or without keys), consuming messages via consumer groups, and async operations via a worker pool.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Message Publishing**: Publish messages to Kafka topics synchronously or asynchronously
- **Keyed Messages**: Publish messages with keys for partition routing
- **Batch Publishing**: Publish multiple messages in batch (with or without keys)
- **Message Consumption**: Consume messages via Kafka consumer groups
- **Async Operations**: Async publishing and consumption via worker pool
- **Worker Pool**: Async job execution support (5 workers)
- **Configurable Producer**: Configurable acknowledgments, retries, and return success settings
- **Consumer Groups**: Support for consumer group rebalancing with round-robin strategy

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
    
    // Create Kafka manager (configuration via config)
    cfg := config.KafkaConfig{
        Enabled:  true,
        Brokers:  []string{"localhost:9092"},
        GroupID: "my-group",
    }
    
    manager, err := kafka.NewKafkaManager(cfg, log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Publish a message
    err = manager.Publish(ctx, "my-topic", []byte("Hello Kafka!"))
    if err != nil {
        panic(err)
    }
    fmt.Println("Message published!")
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `KafkaManager` | Main manager with Sarama sync producer and worker pool |
| `consumerHandler` | Implements Sarama ConsumerGroupHandler for message processing |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   KafkaManager                            │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  Sarama    │  → SyncProducer for publishing           │
│  │  Producer  │  → Configurable acks, retries         │
│  └────────────────┘                                      │
│                                                         │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ AsyncResult │                   │
│  │  (5 workers)│      │   Channel   │                   │
│  └─────────────┘      └──────────────┘                   │
│         ▲                      │                            │
│         │                      ▼                            │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  SubmitJob  │      │ ExecuteAsync │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                         │
│  Brokers: Kafka broker addresses (e.g., localhost:9092)  │
│  GroupID: Consumer group ID for consuming              │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewKafkaManager(cfg, logger)
    │
    ├── Check cfg.Enabled
    ├── Create Sarama config:
    │   ├── Producer.Return.Successes = true
    │   ├── Producer.RequiredAcks = WaitForAll
    │   └── Producer.Retry.Max = 5
    │
    ├── Create SyncProducer with cfg.Brokers
    ├── Create WorkerPool(5)
    ├── Start worker pool
    └── Return KafkaManager
```

### 2. Publish Flow

```
Publish(ctx, topic, message)
    │
    ├── Create ProducerMessage:
    │   ├── Topic
    │   └── Value (ByteEncoder)
    │
    ├── producer.SendMessage(msg)
    └── Return error
```

### 3. Consume Flow

```
Consume(ctx, topic, handler)
    │
    ├── Create Sarama config:
    │   ├── Consumer.Group.Rebalance.Strategy = RoundRobin
    │   └── Consumer.Offsets.Initial = OffsetOldest
    │
    ├── Create ConsumerGroup with brokers and groupID
    ├── Create consumerHandler with handler function
    │
    └── Loop:
        ├── Wait for context.Done()
        └── consumerGroup.Consume(ctx, topics, consumer)
```

### 4. Async Publish Flow

```
PublishAsync(ctx, topic, message)
    │
    └── ExecuteAsync(ctx, func)
        └── Publish(ctx, topic, message)
```

## Configuration

### Config Struct (passed to NewKafkaManager)

| Field | Type | Description |
|-----|------|-------------|
| `Enabled` | bool | Enable/disable Kafka manager |
| `Brokers` | []string | List of Kafka broker addresses |
| `GroupID` | string | Consumer group ID |

### Example Configuration

```go
cfg := config.KafkaConfig{
    Enabled:  true,
    Brokers:  []string{"broker1:9092", "broker2:9092", "broker3:9092"},
    GroupID: "my-consumer-group",
}
```

## Usage Examples

### Publishing Messages

```go
ctx := context.Background()

// Publish a simple message
err := manager.Publish(ctx, "my-topic", []byte("Hello, Kafka!"))
if err != nil {
    panic(err)
}
fmt.Println("Message published!")

// Publish a message with a key
err = manager.PublishWithKey(ctx, "my-topic", []byte("key1"), []byte("Message with key"))
if err != nil {
    panic(err)
}
fmt.Println("Keyed message published!")
```

### Batch Publishing

```go
ctx := context.Background()

// Publish multiple messages
messages := [][]byte{
    []byte("Message 1"),
    []byte("Message 2"),
    []byte("Message 3"),
}

// Async batch publish
result := manager.PublishBatchAsync(ctx, "my-topic", messages)
select {
case <-result.Ch:
    fmt.Println("Batch published!")
case err := <-result.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Batch publish with keys
keyValuePairs := [][2][]byte{
    {[]byte("key1"), []byte("Value 1")},
    {[]byte("key2"), []byte("Value 2")},
}

result2 := manager.PublishBatchWithKeysAsync(ctx, "my-topic", keyValuePairs)
select {
case <-result2.Ch:
    fmt.Println("Batch with keys published!")
case err := <-result2.ErrCh:
    fmt.Printf("Error: %v\n", err)
}
```

### Consuming Messages

```go
ctx := context.Background()

// Define message handler
handler := func(key, value []byte) error {
    fmt.Printf("Received: key=%s, value=%s\n", string(key), string(value))
    return nil
}

// Consume (this blocks, run in a goroutine)
go func() {
    err := manager.Consume(ctx, "my-topic", handler)
    if err != nil {
        fmt.Printf("Consumer error: %v\n", err)
    }
}()

// Or use async consumer
manager.ConsumeAsync(ctx, "my-topic", handler)
```

### Async Operations

```go
ctx := context.Background()

// Async publish
result := manager.PublishAsync(ctx, "my-topic", []byte("Async message"))
select {
case <-result.Ch:
    fmt.Println("Async publish complete!")
case err := <-result.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async publish with key
result2 := manager.PublishWithKeyAsync(ctx, "my-topic", []byte("key"), []byte("Async keyed"))
select {
case <-result2.Ch:
    fmt.Println("Async keyed publish complete!")
case err := <-result2.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async batch publish
messages := [][]byte{[]byte("msg1"), []byte("msg2")}
result3 := manager.PublishBatchAsync(ctx, "my-topic", messages)
select {
case <-result3.Ch:
    fmt.Println("Async batch complete!")
case err := <-result3.ErrCh:
    fmt.Printf("Error: %v\n", err)
}
```

### Get Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Brokers: %v\n", status["brokers"])
fmt.Printf("Group ID: %s\n", status["group_id"])

if status["connected"] == false {
    if errMsg, ok := status["error"].(string); ok {
        fmt.Printf("Error: %s\n", errMsg)
    }
}
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/IBM/sarama` | Kafka client library for Go |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `context`, `fmt` |

## Error Handling

The library uses Go error handling patterns:

- **Producer errors**: Returned from `SendMessage()` calls
- **Consumer errors**: Returned from `Consume()` and consumer group operations
- **Context cancellation**: Consumer respects `context.Context` for graceful shutdown
- **Nil checks**: Public methods handle nil receiver gracefully in `GetStatus()`

Example error handling:

```go
err := manager.Publish(ctx, "my-topic", []byte("test"))
if err != nil {
    if strings.Contains(err.Error(), "topic not found") {
        return fmt.Errorf("topic does not exist: %w", err)
    }
    if strings.Contains(err.Error(), "not leader for partition") {
        // Sarama will retry automatically based on config
        return fmt.Errorf("partition leader error: %w", err)
    }
    return fmt.Errorf("failed to publish: %w", err)
}
```

## Common Pitfalls

### 1. Kafka Brokers Not Reachable

**Problem**: `connection refused` or `no route to host` errors

**Solution**: 
- Verify broker addresses: `cfg.Brokers = []string{"localhost:9092"}`
- Check network connectivity (ping/telnet to broker port)
- Verify firewall allows access to broker ports (default: 9092)

### 2. Topic Does Not Exist

**Problem**: `topic not found` or `unknown topic` errors

**Solution**:
- Create topic manually via Kafka CLI or enable auto-creation
- Verify topic name spelling
- Check Kafka broker configuration for `auto.create.topics.enable=true`

### 3. Consumer Group Issues

**Problem**: Consumer not receiving messages or rebalancing errors

**Solution**:
- Verify GroupID is set correctly
- Check that topic has messages to consume
- Ensure consumer is started before messages are published
- Check consumer group status via Kafka CLI

### 4. Producer Timeout

**Problem**: Publish operations timing out

**Solution**:
```go
// Sarama config has built-in retry (default: 5 retries)
// For longer timeout, adjust the underlying network settings
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
```

### 5. Consumer Blocking

**Problem**: `Consume()` blocks the calling goroutine

**Solution**:
- Always run consumer in a separate goroutine
- Or use `ConsumeAsync()` for non-blocking start
- Use context cancellation to stop consumption

### 6. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewKafkaManager()` succeeded
- Verify `cfg.Enabled` is true

## Advanced Usage

### Message Handler with Error Handling

```go
func createHandler(logger *logger.Logger) func(key, value []byte) error {
    return func(key, value []byte) error {
        if len(value) == 0 {
            logger.Warn("Received empty message", "key", string(key))
            return nil // Skip empty messages
        }
        
        // Process the message
        logger.Info("Processing message", 
            "key", string(key), 
            "size", len(value))
        
        // Your business logic here
        // ...
        
        return nil
    }
}
```

### Graceful Consumer Shutdown

```go
func startConsumer(manager *kafka.KafkaManager) {
    ctx, cancel := context.WithCancel(context.Background())
    defer cancel()
    
    handler := func(key, value []byte) error {
        fmt.Printf("Received: %s\n", string(value))
        return nil
    }
    
    // Start consumer in goroutine
    errCh := make(chan error, 1)
    go func() {
        errCh <- manager.Consume(ctx, "my-topic", handler)
    }()
    
    // Wait for shutdown signal
    // (e.g., from OS signal)
    // ...
    
    // Cancel context to stop consumer
    cancel()
    
    // Wait for consumer to finish
    if err := <-errCh; err != nil {
        fmt.Printf("Consumer stopped with error: %v\n", err)
    }
}
```

### Batch Message Processing

```go
func publishBatch(manager *kafka.KafkaManager, topic string, messages []string) error {
    ctx := context.Background()
    
    // Convert to byte slices
    byteMessages := make([][]byte, len(messages))
    for i, msg := range messages {
        byteMessages[i] = []byte(msg)
    }
    
    // Async batch publish
    result := manager.PublishBatchAsync(ctx, topic, byteMessages)
    
    select {
    case <-result.Ch:
        fmt.Printf("Published %d messages\n", len(messages))
        return nil
    case err := <-result.ErrCh:
        return fmt.Errorf("batch publish failed: %w", err)
    }
}
```

### Monitoring Consumer Lag

```go
func monitorConsumer(manager *kafka.KafkaManager, topic, groupID string) {
    // Note: This requires additional Kafka admin client
    // or using Sarama's ClusterAdmin
    
    // Example using Sarama admin:
    // admin, err := sarama.NewClusterAdmin(manager.Brokers, nil)
    // if err != nil {
    //     fmt.Printf("Error creating admin: %v\n", err)
    //     return
    // }
    // defer admin.Close()
    //
    // groups, err := admin.ListConsumerGroups()
    // ...
    
    fmt.Printf("Monitoring consumer group: %s on topic: %s\n", groupID, topic)
}
```

## Internal Algorithms

### Producer Message Creation

```
Publish(topic, message):
    │
    ├── Create ProducerMessage:
    │   ├── Topic = topic
    │   └── Value = ByteEncoder(message)
    │
    ├── producer.SendMessage(msg)
    │   └── Returns (partition, offset, error)
    │
    └── Return error (partition and offset ignored)
```

### Consumer Group Processing

```
Consume(ctx, topic, handler):
    │
    ├── Create Sarama config:
    │   ├── Rebalance.Strategy = RoundRobin
    │   └── Offsets.Initial = OffsetOldest
    │
    ├── Create ConsumerGroup(brokers, groupID, config)
    │
    └── Loop:
        ├── Check ctx.Done()
        │   └── Return nil (graceful shutdown)
        │
        └── consumerGroup.Consume(ctx, topics, consumer)
            └── For each message in claim.Messages():
                ├── Call handler(message.Key, message.Value)
                └── session.MarkMessage(message, "")
```

### Async Operation Pattern

```
PublishAsync(ctx, topic, message):
    │
    └── ExecuteAsync(ctx, func):
        └── return Publish(ctx, topic, message)
            └── Returns (struct{}, error)
```

### Worker Pool Integration

```
Async Operations:
    │
    ├── ExecuteAsync(ctx, func)
    │   └── Returns *AsyncResult[struct{}]
    │
    ├── Result channels:
    │   ├── Ch: Receives empty struct
    │   └── ErrCh: Receives error
    │
    └── Usage: select on Ch and ErrCh
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.