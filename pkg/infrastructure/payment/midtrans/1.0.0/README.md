# Midtrans Payment Gateway Manager

## Overview

The `MidtransManager` is a Go library for interacting with the Midtrans Payment Gateway API. It provides transaction creation, status checking, and webhook signature verification. The library uses Basic authentication (Server Key) and HMAC-SHA512 signatures for secure requests, with support for sandbox/production environments and async operations.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Transaction Creation**: Create payment transactions with order details, customer info, item details
- **Transaction Status**: Check status of existing transactions
- **Webhook Verification**: Verify incoming webhook signatures using SHA512
- **Basic Authentication**: Secure API access with Server Key
- **Sandbox Support**: Switch between sandbox and production environments
- **Retry Logic**: HTTP client with retry support (3 retries)
- **Worker Pool**: Async job execution support (4 workers)
- **Status Monitoring**: Get connection status and API health

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
    
    // Create Midtrans manager (configuration via viper)
    manager, err := midtrans.NewMidtransManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Create a transaction
    request := midtrans.MidtransTransactionRequest{
        TransactionDetails: midtrans.MidtransTransactionDetails{
            OrderID:     "order-123",
            GrossAmount: 150000,
        },
        CustomerDetails: &midtrans.MidtransCustomerDetails{
            FirstName: "John",
            LastName:  "Doe",
            Email:     "john@example.com",
            Phone:     "08123456789",
        },
        ItemDetails: []midtrans.MidtransItemDetails{
            {
                ID:       "item-1",
                Name:     "Go Programming Book",
                Price:    150000,
                Quantity: 1,
                Category: "Books",
            },
        },
        PaymentType: "gopay",
    }
    
    response, err := manager.CreateTransaction(ctx, request)
    if err != nil {
        panic(err)
    }
    
    fmt.Printf("Transaction ID: %s\n", response.TransactionID)
    fmt.Printf("Order ID: %s\n", response.OrderID)
    fmt.Printf("Status: %s\n", response.StatusMessage)
    fmt.Printf("Token: %s\n", response.Token)
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `MidtransManager` | Main manager with HTTP client, credentials, worker pool |
| `MidtransTransactionRequest` | Transaction request with order, customer, item details |
| `MidtransTransactionResponse` | Response with token, redirect URL, transaction details |
| `MidtransStatusResponse` | Transaction status with fraud status, payment amount |
| `MidtransNotification` | Incoming webhook notification |
| `MidtransCustomerDetails` | Customer information |
| `MidtransItemDetails` | Item details with price, quantity |

### Concurrency Model

```
┌─────────────────────────────────────────────────────┐
│                   MidtransManager                       │
├─────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  retryablehttp │  → API requests with retry               │
│  │  .Client      │  → Basic auth (Server Key)              │
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
│  BaseURL: https://api.midtrans.com/v2 (or sandbox) │
│  ServerKey: Midtrans Server Key                         │
│  ClientKey: Midtrans Client Key                         │
│  IsProduction: true/false                             │
└─────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewMidtransManager(logger)
    │
    ├── Check viper config: "midtrans.enabled"
    ├── Get server_key: "midtrans.server_key"
    ├── Get client_key: "midtrans.client_key"
    ├── Get is_production: "midtrans.production"
    │
    ├── Set baseURL:
    │   ├── Production: "https://api.midtrans.com/v2"
    │   └── Sandbox: "https://api.sandbox.midtrans.com/v2"
    │
    ├── Create retryablehttp.Client:
    │   ├── RetryMax = 3
    │   ├── RetryWaitMin = 1s
    │   ├── RetryWaitMax = 10s
    │   └── HTTPClient.Timeout = 60s
    │
    ├── Test connection: testConnection()
    │   └── GET /health
    │
    ├── Create WorkerPool(4)
    ├── Start worker pool
    └── Return MidtransManager
```

### 2. Create Transaction Flow

```
CreateTransaction(ctx, request)
    │
    ├── Marshal request to JSON
    ├── POST /charge
    ├── Set Basic Auth header (ServerKey, "")
    ├── Set header: Content-Type = application/json
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON into MidtransTransactionResponse
    └── Return *MidtransTransactionResponse
```

### 3. Get Transaction Status Flow

```
GetTransactionStatus(ctx, orderID)
    │
    ├── Build URL: {baseURL}/{orderID}/status
    ├── Set Basic Auth header (ServerKey, "")
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON into MidtransStatusResponse
    └── Return *MidtransStatusResponse
```

### 4. Signature Verification Flow

```
VerifySignature(notification)
    │
    ├── Build input: orderID + status_code + gross_amount + ServerKey
    ├── Compute SHA512 hash
    ├── Encode to hex string
    └── Compare with notification.SignatureKey
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `midtrans.enabled` | bool | false | Enable/disable Midtrans manager |
| `midtrans.server_key` | string | "" | Midtrans Server Key |
| `midtrans.client_key` | string | "" | Midtrans Client Key |
| `midtrans.production` | bool | false | Use production environment |

## Usage Examples

### Create Transaction

```go
ctx := context.Background()

request := midtrans.MidtransTransactionRequest{
    TransactionDetails: midtrans.MidtransTransactionDetails{
        OrderID:     "order-456",
        GrossAmount: 250000,
    },
    CustomerDetails: &midtrans.MidtransCustomerDetails{
        FirstName: "Jane",
        LastName:  "Smith",
        Email:     "jane@example.com",
    },
    ItemDetails: []midtrans.MidtransItemDetails{
        {
            Name:     "Premium Course",
            Price:    250000,
            Quantity: 1,
        },
    },
}

response, err := manager.CreateTransaction(ctx, request)
if err != nil {
    panic(err)
}

fmt.Printf("Transaction ID: %s\n", response.TransactionID)
fmt.Printf("Status: %s\n", response.StatusMessage)
fmt.Printf("Amount: %.2f\n", response.Data.Amount)
```

### Check Transaction Status

```go
ctx := context.Background()

status, err := manager.GetTransactionStatus(ctx, "order-456")
if err != nil {
    panic(err)
}

fmt.Printf("Transaction ID: %s\n", status.Data.TransactionID)
fmt.Printf("Order ID: %s\n", status.Data.OrderID)
fmt.Printf("Status: %s\n", status.Data.Status)
fmt.Printf("Amount: %.2f\n", status.Data.Amount)
fmt.Printf("Paid At: %s\n", status.Data.PaidAt)
```

### Verify Webhook Signature

```go
// In your webhook handler
func webhookHandler(w http.ResponseWriter, r *http.Request) {
    var notification midtrans.MidtransNotification
    json.NewDecoder(r.Body).Decode(&notification)
    
    // Verify signature
    if !manager.VerifySignature(notification) {
        http.Error(w, "Invalid signature", http.StatusBadRequest)
        return
    }
    
    // Process notification
    fmt.Printf("Order %s: %s\n", notification.OrderID, notification.Status)
}
```

### Async Operations

```go
ctx := context.Background()

// Async create transaction
request := midtrans.MidtransTransactionRequest{
    TransactionDetails: midtrans.MidtransTransactionDetails{
        OrderID:     "async-order",
        GrossAmount: 100000,
    },
}

createChan := manager.CreateTransactionAsync(ctx, request)
response, err := createChan.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Created: %s\n", response.TransactionID)

// Async get status
statusChan := manager.GetTransactionStatusAsync(ctx, "async-order")
status, err := statusChan.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Status: %s\n", status.Data.Status)
```

### Getting Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Base URL: %s\n", status["base_url"])
fmt.Printf("Production: %v\n", status["is_production"])

if status["connected"] == false {
    if errMsg, ok := status["error"].(string); ok {
        fmt.Printf("Error: %s\n", errMsg)
    }
}
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/hashicorp/go-retryablehttp` | HTTP client with retry logic |
| `github.com/spf13/viper` | Configuration management |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `crypto/sha512`, `encoding/hex`, `encoding/json`, `net/http`, `bytes`, `io`, `time` |

## Error Handling

The library uses Go error handling patterns:

- **Authentication failures**: Returned from API (invalid Server Key)
- **Signature errors**: Invalid SHA512 signatures
- **API errors**: Returned from Midtrans API calls
- **Nil checks**: Public methods check manager state

Example error handling:

```go
response, err := manager.CreateTransaction(ctx, request)
if err != nil {
    if strings.Contains(err.Error(), "401") {
        return fmt.Errorf("invalid server key: %w", err)
    }
    if strings.Contains(err.Error(), "404") {
        return fmt.Errorf("transaction not found: %w", err)
    }
    return fmt.Errorf("transaction creation failed: %w", err)
}
```

## Common Pitfalls

### 1. Manager Not Enabled

**Problem**: `midtrans.enabled` is false

**Solution**:
- Set `midtrans.enabled = true` in configuration
- Check viper configuration is loaded

### 2. Server Key Missing

**Problem**: `server_key` not set

**Solution**:
- Set `midtrans.server_key` to your Midtrans Server Key
- Get Server Key from Midtrans Dashboard → Settings → Access Keys

### 3. Wrong Environment

**Problem**: Using production when should use sandbox or vice versa

**Solution**:
- Set `midtrans.production = true` for production
- Set `midtrans.production = false` for sandbox testing

### 4. Signature Verification Fails

**Problem**: Webhook signature mismatch

**Solution**:
- Ensure Server Key is correct
- Verify the signature input format: orderID + status_code + gross_amount + ServerKey
- Check that webhook payload is not modified

### 5. Transaction Not Found

**Problem**: `404` when checking status

**Solution**:
- Verify order ID is correct
- Check transaction exists and is accessible

### 6. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewMidtransManager()` succeeded
- Verify `midtrans.enabled` is true

## Advanced Usage

### Health Check

```go
func healthCheck(manager *midtrans.MidtransManager) error {
    ctx := context.Background()
    _, err := manager.GetBalance(ctx)
    if err != nil {
        return fmt.Errorf("not connected to Midtrans API: %w", err)
    }
    fmt.Printf("Midtrans API is healthy (URL: %s)\n", manager.BaseURL)
    return nil
}
```

### Batch Status Checks

```go
func batchCheckStatus(manager *midtrans.MidtransManager, orderIDs []string) ([]*midtrans.MidtransStatusResponse, error) {
    ctx := context.Background()
    results := make([]*midtrans.MidtransStatusResponse, len(orderIDs))
    errors := make([]error, len(orderIDs))
    
    for i, id := range orderIDs {
        func(idx int, orderID string) {
            manager.SubmitAsyncJob(func() {
                status, err := manager.GetTransactionStatus(ctx, orderID)
                results[idx] = status
                errors[idx] = err
            })
        }(i, id)
    }
    
    for i, err := range errors {
        if err != nil {
            return nil, fmt.Errorf("failed to check status %d: %w", i, err)
        }
    }
    return results, nil
}
```

## Internal Algorithms

### Signature Generation

```
VerifySignature(notification):
    │
    ├── Build input: notification.OrderID + notification.StatusCode + notification.GrossAmount + ServerKey
    ├── Compute SHA512 hash of input
    ├── Encode hash to hex string
    └── Compare with notification.SignatureKey
```

### API Request Flow

```
CreateTransaction():
    │
    ├── Build JSON payload with transaction details
    ├── POST to /charge
    ├── Set Basic Auth header (ServerKey)
    ├── Execute with retryablehttp (3 retries)
    ├── Parse response JSON
    └── Return transaction response
```

### Connection Test

```
testConnection():
    │
    ├── POST /health
    ├── Set Basic Auth header (ServerKey)
    ├── Check status code (200 OK)
    └── Return error if failed
```

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/health` | POST | Health check |
| `/charge` | POST | Create transaction |
| `/{orderID}/status` | GET | Check transaction status |

## Webhook Handling

When Midtrans sends a webhook notification:

1. Parse the incoming JSON body into `MidtransNotification` struct
2. Verify the signature using `VerifySignature()`
3. Process the transaction status update
4. Respond with appropriate HTTP status code

Example webhook handler:

```go
func webhookHandler(w http.ResponseWriter, r *http.Request) {
    body, _ := io.ReadAll(r.Body)
    
    var notification midtrans.MidtransNotification
    json.Unmarshal(body, &notification)
    
    if !manager.VerifySignature(notification) {
        http.Error(w, "Invalid signature", http.StatusBadRequest)
        return
    }
    
    // Process the notification
    log.Printf("Order %s status: %s\n", notification.OrderID, notification.Status)
    
    w.WriteHeader(http.StatusOK)
}
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.