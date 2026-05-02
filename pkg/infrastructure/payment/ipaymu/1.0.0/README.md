# Ipaymu Payment Gateway Manager

## Overview

The `IpaymuManager` is a Go library for interacting with the Ipaymu Payment Gateway API. It provides transaction creation, status checking, balance retrieval, and webhook signature verification. The library uses HMAC-SHA256 signatures for secure requests and supports sandbox/production environments with async operations.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Transaction Creation**: Create payment transactions with product details, quantities, prices
- **Transaction Status**: Check status of existing transactions
- **Balance Retrieval**: Get current account balance
- **Webhook Verification**: Verify incoming webhook signatures
- **Signature Generation**: HMAC-SHA256 signature creation for secure API calls
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
    
    // Create Ipaymu manager (configuration via viper)
    manager, err := ipaymu.NewIpaymuManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Create a transaction
    request := ipaymu.IpaymuTransactionRequest{
        Product:     []string{"Product A", "Product B"},
        Qty:         []int{1, 2},
        Price:       []float64{10000, 25000},
        Description: []string{"Nice product", "Another nice product"},
        ReturnURL:   "https://myapp.com/return",
        CancelURL:   "https://myapp.com/cancel",
        ReferenceID: "order-12345",
        BuyerName:   "John Doe",
        BuyerEmail:  "john@example.com",
        BuyerPhone:  "08123456789",
        Expired:     24,
    }
    
    response, err := manager.CreateTransaction(ctx, request)
    if err != nil {
        panic(err)
    }
    
    fmt.Printf("Transaction ID: %s\n", response.Data.TransactionID)
    fmt.Printf("Payment URL: %s\n", response.Data.PaymentURL)
    fmt.Printf("Amount: %.2f\n", response.Data.Amount)
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `IpaymuManager` | Main manager with HTTP client, API key, virtual account, worker pool |
| `IpaymuTransactionRequest` | Payment request with products, quantities, prices, buyer info |
| `IpaymuTransactionResponse` | Response with transaction ID, payment URL, amount, fee |
| `IpaymuPaymentStatus` | Payment status with transaction details |
| `IpaymuBalanceResponse` | Account balance information |
| `IpaymuWebhook` | Incoming webhook event with signature |

### Concurrency Model

```
┌─────────────────────────────────────────────────────┐
│                   IpaymuManager                        │
├─────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  retryablehttp │  → API requests with retry               │
│  │  .Client      │  → HMAC-SHA256 signature auth           │
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
│  BaseURL: https://my.ipaymu.com/api/v2 (or sandbox)│
│  APIKey: Ipaymu API key                                │
│  VirtualAccount: Ipaymu virtual account number          │
│  SecretKey: For HMAC-SHA256 signatures                 │
└─────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewIpaymuManager(logger)
    │
    ├── Check viper config: "ipaymu.enabled"
    ├── Get api_key: "ipaymu.api_key"
    ├── Get virtual_account: "ipaymu.virtual_account"
    ├── Get secret_key: "ipaymu.secret_key"
    ├── Get sandbox: "ipaymu.sandbox" (default: false)
    │
    ├── Set baseURL:
    │   ├── Production: "https://my.ipaymu.com/api/v2"
    │   └── Sandbox: "https://sandbox.ipaymu.com/api/v2"
    │
    ├── Create retryablehttp.Client:
    │   ├── RetryMax = 3
    │   ├── RetryWaitMin = 1s
    │   ├── RetryWaitMax = 10s
    │   └── HTTPClient.Timeout = 30s
    │
    ├── Test connection: testConnection()
    │   └── POST /balance
    │
    ├── Create WorkerPool(4)
    ├── Start worker pool
    └── Return IpaymuManager
```

### 2. Create Transaction Flow

```
CreateTransaction(ctx, request)
    │
    ├── Marshal request to JSON
    ├── POST /payment
    ├── Set headers: va, apiKey
    ├── Generate signature: HMAC-SHA256(payload)
    ├── Set header: signature
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON into IpaymuTransactionResponse
    └── Return *IpaymuTransactionResponse
```

### 3. Get Transaction Status Flow

```
GetTransactionStatus(ctx, transactionID)
    │
    ├── Build payload: {"transactionId": transactionID}
    ├── Marshal to JSON
    ├── POST /transaction
    ├── Set headers: va, apiKey
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON into IpaymuPaymentStatus
    └── Return *IpaymuPaymentStatus
```

### 4. Get Balance Flow

```
GetBalance(ctx)
    │
    ├── POST /balance
    ├── Set headers: va, apiKey
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON into IpaymuBalanceResponse
    └── Return *IpaymuBalanceResponse
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `ipaymu.enabled` | bool | false | Enable/disable Ipaymu manager |
| `ipaymu.api_key` | string | "" | Ipaymu API key |
| `ipaymu.virtual_account` | string | "" | Ipaymu virtual account number |
| `ipaymu.secret_key` | string | "" | Secret key for HMAC-SHA256 signatures |
| `ipaymu.sandbox` | bool | false | Use sandbox environment |

## Usage Examples

### Create Transaction

```go
ctx := context.Background()

request := ipaymu.IpaymuTransactionRequest{
    Product:     []string{"Go Programming Book"},
    Qty:         []int{1},
    Price:       []float64{150000},
    Description: []string{"Learn Go fast"},
    ReturnURL:   "https://myapp.com/return",
    CancelURL:   "https://myapp.com/cancel",
    ReferenceID: "book-order-001",
}

response, err := manager.CreateTransaction(ctx, request)
if err != nil {
    panic(err)
}

fmt.Printf("Transaction ID: %s\n", response.Data.TransactionID)
fmt.Printf("Payment URL: %s\n", response.Data.PaymentURL)
fmt.Printf("Amount: %.2f (Fee: %.2f)\n", response.Data.Amount, response.Data.Fee)
```

### Check Transaction Status

```go
ctx := context.Background()

status, err := manager.GetTransactionStatus(ctx, "transaction-12345")
if err != nil {
    panic(err)
}

fmt.Printf("Transaction ID: %s\n", status.Data.TransactionID)
fmt.Printf("Status: %s\n", status.Data.Status)
fmt.Printf("Amount: %.2f\n", status.Data.Amount)
fmt.Printf("Paid At: %s\n", status.Data.PaidAt)
```

### Get Balance

```go
ctx := context.Background()

balance, err := manager.GetBalance(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Balance: %.2f\n", balance.Data.Balance)
```

### Verify Webhook Signature

```go
// In your webhook handler
func webhookHandler(w http.ResponseWriter, r *http.Request) {
    var webhook ipaymu.IpaymuWebhook
    json.NewDecoder(r.Body).Decode(&webhook)
    
    // Verify signature
    if !manager.VerifyWebhookSignature(&webhook) {
        http.Error(w, "Invalid signature", http.StatusBadRequest)
        return
    }
    
    // Process webhook...
    fmt.Printf("Transaction %s: %s\n", webhook.TransactionID, webhook.Status)
}
```

### Async Operations

```go
ctx := context.Background()

// Async create transaction
request := ipaymu.IpaymuTransactionRequest{
    Product: []string{"Async Product"},
    Price:   []float64{50000},
}
createChan := manager.CreateTransactionAsync(ctx, request)
response, err := createChan.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Created: %s\n", response.Data.TransactionID)

// Async get balance
balanceChan := manager.GetBalanceAsync(ctx)
balance, err := balanceChan.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Balance: %.2f\n", balance.Data.Balance)
```

### Getting Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Base URL: %s\n", status["base_url"])
fmt.Printf("Virtual Account: %s\n", status["virtual_account"])
fmt.Printf("API Key Configured: %v\n", status["api_key_configured"])

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
| Standard library | `crypto/hmac`, `crypto/sha256`, `encoding/hex`, `encoding/json`, `net/http`, `bytes`, `io`, `time` |

## Error Handling

The library uses Go error handling patterns:

- **Authentication failures**: Returned from API (invalid API key)
- **Signature errors**: Invalid HMAC-SHA256 signatures
- **API errors**: Returned from Ipaymu API calls
- **Nil checks**: Public methods check manager state

Example error handling:

```go
response, err := manager.CreateTransaction(ctx, request)
if err != nil {
    if strings.Contains(err.Error(), "401") {
        return fmt.Errorf("invalid API key: %w", err)
    }
    if strings.Contains(err.Error(), "403") {
        return fmt.Errorf("access forbidden: %w", err)
    }
    return fmt.Errorf("transaction creation failed: %w", err)
}
```

## Common Pitfalls

### 1. Manager Not Enabled

**Problem**: `ipaymu.enabled` is false

**Solution**:
- Set `ipaymu.enabled = true` in configuration
- Check viper configuration is loaded

### 2. API Key Missing

**Problem**: `api_key` not set

**Solution**:
- Set `ipaymu.api_key` to your Ipaymu API key
- Get API key from Ipaymu dashboard

### 3. Virtual Account Missing

**Problem**: `virtual_account` not set

**Solution**:
- Set `ipaymu.virtual_account` to your Ipaymu virtual account number
- This is required for API authentication

### 4. Secret Key Missing

**Problem**: `secret_key` not set

**Solution**:
- Set `ipaymu.secret_key` to your secret key
- Used for generating HMAC-SHA256 signatures
- Get from Ipaymu dashboard

### 5. Wrong Environment

**Problem**: Using production when should use sandbox or vice versa

**Solution**:
- Set `ipaymu.sandbox = true` for testing
- Set `ipaymu.sandbox = false` for production

### 6. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewIpaymuManager()` succeeded
- Verify `ipaymu.enabled` is true

## Advanced Usage

### Custom Signature Generation

```go
func generateCustomSignature(manager *ipaymu.IpaymuManager, transactionID string, status string, amount float64) string {
    payload := fmt.Sprintf("%s:%s:%.2f", transactionID, status, amount)
    return manager.GenerateSignature(payload)
}
```

### Health Check

```go
func healthCheck(manager *ipaymu.IpaymuManager) error {
    ctx := context.Background()
    _, err := manager.GetBalance(ctx)
    if err != nil {
        return fmt.Errorf("not connected to Ipaymu API: %w", err)
    }
    fmt.Println("Ipaymu API is healthy")
    return nil
}
```

### Batch Transaction Status Checks

```go
func batchCheckStatus(manager *ipaymu.IpaymuManager, transactionIDs []string) ([]*ipaymu.IpaymuPaymentStatus, error) {
    ctx := context.Background()
    results := make([]*ipaymu.IpaymuPaymentStatus, len(transactionIDs))
    errors := make([]error, len(transactionIDs))
    
    for i, id := range transactionIDs {
        func(idx int, tid string) {
            manager.SubmitAsyncJob(func() {
                status, err := manager.GetTransactionStatus(ctx, tid)
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
generateSignature(payload):
    │
    ├── Create HMAC with SHA256 and secret key
    ├── Write payload bytes
    ├── Compute hash
    ├── Encode to hex string
    └── Return signature
```

### API Request Flow

```
CreateTransaction():
    │
    ├── Build JSON payload with transaction details
    ├── Set headers: va, apiKey
    ├── Generate signature from payload
    ├── POST to /payment
    ├── Parse response JSON
    └── Return transaction response
```

### Connection Test

```
testConnection():
    │
    ├── POST /balance
    ├── Set headers: va, apiKey
    ├── Check status code (200 OK)
    └── Return error if failed
```

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/balance` | POST | Get account balance (health check) |
| `/payment` | POST | Create new transaction |
| `/transaction` | POST | Check transaction status |

## Webhook Handling

When Ipaymu sends a webhook notification:

1. Parse the incoming JSON body into `IpaymuWebhook` struct
2. Verify the signature using `VerifyWebhookSignature()`
3. Process the transaction status update
4. Respond with appropriate HTTP status code

Example webhook handler:

```go
func webhookHandler(w http.ResponseWriter, r *http.Request) {
    body, _ := io.ReadAll(r.Body)
    
    var webhook ipaymu.IpaymuWebhook
    json.Unmarshal(body, &webhook)
    
    if !manager.VerifyWebhookSignature(&webhook) {
        http.Error(w, "Invalid signature", http.StatusBadRequest)
        return
    }
    
    // Process the notification
    log.Printf("Transaction %s status: %s\n", webhook.TransactionID, webhook.Status)
    
    w.WriteHeader(http.StatusOK)
}
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.