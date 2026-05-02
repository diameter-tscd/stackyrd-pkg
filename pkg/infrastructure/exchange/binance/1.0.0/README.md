# Binance Manager

## Overview

The `BinanceManager` is a Go library for interacting with the Binance cryptocurrency exchange API. It supports both public endpoints (ticker, klines) and authenticated endpoints (account information) with HMAC-SHA256 signature generation. The library provides market data retrieval, account management, and candlestick data access with async support via worker pool.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Market Data**: Get 24hr ticker price change statistics
- **Account Management**: Retrieve account information including balances and commissions
- **Candlestick Data**: Get kline/candlestick data for technical analysis
- **Authentication**: HMAC-SHA256 signature generation for secure API calls
- **Retry Logic**: HTTP client with retry support (3 retries)
- **Worker Pool**: Async job execution support (4 workers)
- **Testnet Support**: Switch between production and testnet environments
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
    
    // Create Binance manager (configuration via viper)
    manager, err := binance.NewBinanceManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Get ticker for BTCUSDT
    ticker, err := manager.GetTicker(ctx, "BTCUSDT")
    if err != nil {
        panic(err)
    }
    fmt.Printf("BTCUSDT Price: %s (Change: %s%%)\n", ticker.LastPrice, ticker.PriceChangePercent)
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `BinanceManager` | Main manager with HTTP client, API credentials, worker pool |
| `BinanceTicker` | 24hr ticker price change statistics |
| `BinanceBalance` | Account balance (asset, free, locked) |
| `BinanceAccount` | Full account information including balances |
| `BinanceOrder` | Trade order information |
| `BinanceKline` | Candlestick/kline data |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   BinanceManager                          │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  retryablehttp │  → API requests with retry            │
│  │  .Client      │  → HMAC signature for auth              │
│  └────────────────┘                                      │
│                                                         │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ AsyncResult │                   │
│  │  (4 workers)│      │   Channel   │                   │
│  └─────────────┘      └──────────────┘                   │
│         ▲                      │                            │
│         │                      ▼                            │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  SubmitJob  │      │ ExecuteAsync │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                         │
│  BaseURL: Production or Testnet                           │
│  APIKey/Secret: For authenticated endpoints              │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewBinanceManager(logger)
    │
    ├── Check viper config: "binance.enabled"
    ├── Get api_key: "binance.api_key"
    ├── Get api_secret: "binance.api_secret"
    ├── Get testnet: "binance.testnet" (default: false)
    │
    ├── Set BaseURL:
    │   ├── Production: "https://api.binance.com"
    │   └── Testnet: "https://testnet.binance.vision"
    │
    ├── Create retryablehttp.Client:
    │   ├── RetryMax = 3
    │   ├── RetryWaitMin = 1s
    │   ├── RetryWaitMax = 10s
    │   └── HTTPClient.Timeout = 30s
    │
    ├── Test connection: testConnection()
    │   └── GET /api/v3/ping
    │
    ├── Create WorkerPool(4)
    ├── Start worker pool
    └── Return BinanceManager
```

### 2. Get Ticker Flow

```
GetTicker(ctx, symbol)
    │
    ├── Build query params: symbol
    ├── Create GET request: /api/v3/ticker/24hr
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON into BinanceTicker
    └── Return *BinanceTicker
```

### 3. Get Account Information Flow

```
GetAccountInformation(ctx)
    │
    ├── Generate timestamp
    ├── Build params: timestamp
    ├── Generate HMAC-SHA256 signature with api_secret
    ├── Add signature to params
    ├── Create GET request: /api/v3/account
    ├── Set header: X-MBX-APIKEY = api_key
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON into BinanceAccount
    └── Return *BinanceAccount
```

### 4. Get Klines Flow

```
GetKlines(ctx, symbol, interval, limit)
    │
    ├── Build query params: symbol, interval, limit
    ├── Create GET request: /api/v3/klines
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON array of arrays
    ├── Convert to []BinanceKline
    └── Return []BinanceKline
```

### 5. Get Status Flow

```
GetStatus()
    │
    ├── If nil: return connected=false
    ├── Create context with 5s timeout
    ├── Ping API: GET /api/v3/ping
    ├── Return connected, base_url, api_key_configured, pool_active
    └── Note: Tests actual API connectivity
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `binance.enabled` | bool | false | Enable/disable Binance manager |
| `binance.api_key` | string | "" | Binance API key |
| `binance.api_secret` | string | "" | Binance API secret |
| `binance.testnet` | bool | false | Use testnet instead of production |

### Environment Variables

The Binance manager does not directly use environment variables. Configuration is read through Viper, which can load from:
- Configuration files (JSON, YAML, TOML)
- Environment variables (if Viper is configured for it)

You can set Binance parameters via:
```go
viper.Set("binance.enabled", true)
viper.Set("binance.api_key", "your_api_key")
viper.Set("binance.api_secret", "your_api_secret")
viper.Set("binance.testnet", false)
```

## Usage Examples

### Getting Ticker Data

```go
ctx := context.Background()

ticker, err := manager.GetTicker(ctx, "ETHUSDT")
if err != nil {
    panic(err)
}

fmt.Printf("Symbol: %s\n", ticker.Symbol)
fmt.Printf("Last Price: %s\n", ticker.LastPrice)
fmt.Printf("24h Change: %s (%s%%)\n", ticker.PriceChange, ticker.PriceChangePercent)
fmt.Printf("24h High: %s, Low: %s\n", ticker.HighPrice, ticker.LowPrice)
fmt.Printf("Volume: %s\n", ticker.Volume)
```

### Getting Account Information

```go
ctx := context.Background()

account, err := manager.GetAccountInformation(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Can Trade: %v\n", account.CanTrade)
fmt.Printf("Maker Commission: %d\n", account.MakerCommission)

for _, balance := range account.Balances {
    if balance.Free != "0" || balance.Locked != "0" {
        fmt.Printf("Asset: %s, Free: %s, Locked: %s\n", 
            balance.Asset, balance.Free, balance.Locked)
    }
}
```

### Getting Candlestick Data

```go
ctx := context.Background()

// Get 1-hour klines for BTCUSDT, limit 100
klines, err := manager.GetKlines(ctx, "BTCUSDT", "1h", 100)
if err != nil {
    panic(err)
}

for _, k := range klines {
    fmt.Printf("Time: %d, Open: %s, High: %s, Low: %s, Close: %s, Volume: %s\n",
        k.OpenTime, k.Open, k.High, k.Low, k.Close, k.Volume)
}
```

### Async Operations

```go
ctx := context.Background()

// Get ticker asynchronously
asyncResult := manager.GetTickerAsync(ctx, "BTCUSDT")
result, err := asyncResult.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Async BTC Price: %s\n", result.LastPrice)

// Get account info asynchronously
accountResult := manager.GetAccountInformationAsync(ctx)
account, err := accountResult.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Account has %d balances\n", len(account.Balances))
```

### Getting Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Base URL: %s\n", status["base_url"])
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
| Standard library | `crypto/hmac`, `crypto/sha256`, `encoding/hex`, `encoding/json`, `net/http`, `net/url`, `strconv`, `time` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: Returned from API ping
- **Authentication errors**: Returned from authenticated endpoints
- **Rate limiting**: Handled by retry logic
- **Nil checks**: Public methods check manager state

Example error handling:

```go
ticker, err := manager.GetTicker(ctx, symbol)
if err != nil {
    if strings.Contains(err.Error(), "400") {
        return fmt.Errorf("invalid symbol or parameters: %w", err)
    }
    if strings.Contains(err.Error(), "429") {
        return fmt.Errorf("rate limit exceeded: %w", err)
    }
    if strings.Contains(err.Error(), "-1022") {
        return fmt.Errorf("signature for authentication failed: %w", err)
    }
    return fmt.Errorf("API request failed: %w", err)
}
```

## Common Pitfalls

### 1. Manager Not Enabled

**Problem**: `binance.enabled` is false

**Solution**:
- Set `binance.enabled = true` in configuration
- Check viper configuration is loaded

### 2. API Credentials Missing

**Problem**: `api_key` or `api_secret` not set

**Solution**:
- Set `binance.api_key` and `binance.api_secret`
- For public endpoints (ticker, klines), credentials are not required
- For account endpoints, credentials are mandatory

### 3. Invalid Symbol

**Problem**: `Invalid symbol` or `400` error

**Solution**:
- Check symbol format (e.g., "BTCUSDT", "ETHUSDT")
- Verify symbol exists on Binance
- Use correct case (usually uppercase)

### 4. Signature Generation Failure

**Problem**: `-1022` signature for authentication failed

**Solution**:
- Verify `api_secret` is correct
- Ensure timestamp is included in params
- Check signature generation logic

### 5. Rate Limiting

**Problem**: `429` Too Many Requests

**Solution**:
- Library retries up to 3 times
- Consider using async operations to spread requests
- Respect Binance rate limits

### 6. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewBinanceManager()` succeeded
- Verify `binance.enabled` is true

## Advanced Usage

### Custom API Request

```go
func customRequest(manager *binance.BinanceManager) error {
    ctx := context.Background()
    
    // Build custom query
    params := url.Values{}
    params.Set("symbol", "BTCUSDT")
    
    // Create request
    req, err := retryablehttp.NewRequestWithContext(ctx, "GET", 
        fmt.Sprintf("%s/api/v3/ticker/price?%s", manager.BaseURL, params.Encode()), nil)
    if err != nil {
        return err
    }
    
    resp, err := manager.Client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    // Process response...
    return nil
}
```

### Health Check

```go
func healthCheck(manager *binance.BinanceManager) error {
    status := manager.GetStatus()
    if status["connected"] == false {
        return fmt.Errorf("not connected to Binance API")
    }
    fmt.Printf("Binance API is healthy (URL: %s)\n", status["base_url"])
    return nil
}
```

### Using Testnet

```go
// Set testnet mode
viper.Set("binance.testnet", true)
viper.Set("binance.api_key", "test_api_key")
viper.Set("binance.api_secret", "test_api_secret")

// Initialize manager (will use testnet URL)
manager, err := binance.NewBinanceManager(log)
if err != nil {
    panic(err)
}
// Now all requests go to testnet.binance.vision
```

### Batch Operations with Async

```go
func getMultipleTickers(manager *binance.BinanceManager, symbols []string) ([]*binance.BinanceTicker, error) {
    ctx := context.Background()
    results := make([]*binance.BinanceTicker, len(symbols))
    errors := make([]error, len(symbols))
    
    // Submit all requests asynchronously
    for i, symbol := range symbols {
        func(idx int, sym string) {
            manager.SubmitAsyncJob(func() {
                ticker, err := manager.GetTicker(ctx, sym)
                results[idx] = ticker
                errors[idx] = err
            })
        }(i, symbol)
    }
    
    // Collect results (in real scenario, use proper synchronization)
    for i, err := range errors {
        if err != nil {
            return nil, fmt.Errorf("failed to get %s: %w", symbols[i], err)
        }
    }
    return results, nil
}
```

## Internal Algorithms

### Signature Generation

```
generateSignature(payload)
    │
    ├── Create HMAC-SHA256 with api_secret as key
    ├── Write payload bytes
    ├── Compute sum
    └── Return hex-encoded string
```

### Connection Test

```
testConnection():
    │
    ├── GET /api/v3/ping
    ├── Check status code
    └── Return error if not 200
```

### Kline Conversion

```
GetKlines() response is array of arrays:
[
  [openTime, open, high, low, close, volume, closeTime, quoteVolume, trades, ...]
]

Convert to []BinanceKline:
  ├── Iterate raw arrays
  ├── Map fields to BinanceKline struct
  └── Return slice
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.