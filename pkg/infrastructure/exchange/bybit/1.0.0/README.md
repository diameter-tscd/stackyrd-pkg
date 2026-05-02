# Bybit Manager

## Overview

The `BybitManager` is a Go library for interacting with the Bybit cryptocurrency exchange API v5. It supports both public endpoints (market data, tickers, klines, orderbook) and authenticated endpoints (wallet, positions, orders, trading) with HMAC-SHA256 signature generation. The library provides comprehensive trading features, wallet management, and market data access with async support via worker pool.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Market Data**: Get 24hr ticker, klines, orderbook (L2), recent trades, funding rates
- **Account Management**: Wallet balance, position management, leverage settings
- **Order Operations**: Place, cancel, query orders, execution history
- **Authentication**: HMAC-SHA256 signature generation for secure API calls
- **Position Mode**: Switch between One-Way and Hedge position modes
- **Retry Logic**: HTTP client with retry support (3 retries)
- **Worker Pool**: Async job execution support (6 workers)
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
    
    // Create Bybit manager (configuration via viper)
    manager, err := bybit.NewBybitManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Get ticker for BTCUSDT (linear category)
    ticker, err := manager.GetTicker(ctx, "linear", "BTCUSDT")
    if err != nil {
        panic(err)
    }
    fmt.Printf("BTCUSDT Price: %s (24h Change: %s%%)\n", ticker.LastPrice, ticker.Price24hPcnt)
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `BybitManager` | Main manager with HTTP client, API credentials, worker pool |
| `BybitTicker` | 24hr ticker price change statistics with funding info |
| `BybitKline` | Candlestick/kline data |
| `BybitOrderBook` | Order book (L2) with bids and asks |
| `BybitRecentTrade` | Recent public trades |
| `BybitWalletBalance` | Wallet balance per coin |
| `BybitPosition` | Trading position with PnL, leverage |
| `BybitOrder` | Trade order information |
| `BybitExecution` | Private execution / fill |
| `BybitFundingRate` | Funding rate history |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   BybitManager                           │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                     │
│  │  retryablehttp │  → API requests with retry             │
│  │  .Client      │  → HMAC signature for auth             │
│  └────────────────┘                                     │
│                                                          │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  WorkerPool │◄─────│ AsyncResult │                  │
│  │  (6 workers)│      │   Channel   │                  │
│  └─────────────┘      └──────────────┘                  │
│         ▲                      │                             │
│         │                      ▼                             │
│  ┌─────────────┐      ┌──────────────┐                 │
│  │  SubmitJob  │      │ ExecuteAsync │                 │
│  └─────────────┘      └──────────────┘                 │
│                                                          │
│  BaseURL: Production or Testnet                            │
│  APIKey/Secret: For authenticated endpoints               │
│  RecvWindow: Request timeout window (default: 5000ms)   │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewBybitManager(logger)
    │
    ├── Check viper config: "bybit.enabled"
    ├── Get api_key: "bybit.api_key"
    ├── Get api_secret: "bybit.api_secret"
    ├── Get testnet: "bybit.testnet"
    ├── Get recv_window: "bybit.recv_window" (default: 5000)
    │
    ├── Set BaseURL:
    │   ├── Production: "https://api.bybit.com"
    │   └── Testnet: "https://api-testnet.bybit.com"
    │
    ├── Create retryablehttp.Client:
    │   ├── RetryMax = 3
    │   ├── RetryWaitMin = 1s
    │   ├── RetryWaitMax = 10s
    │   └── HTTPClient.Timeout = 30s
    │
    ├── Test connection: testConnection()
    │   └── GET /v5/market/time
    │
    ├── Create WorkerPool(6)
    ├── Start worker pool
    └── Return BybitManager
```

### 2. Public Request Flow

```
doPublicRequest(ctx, method, endpoint, params)
    │
    ├── Build query string (sorted params)
    ├── Create request with URL + query
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Check retCode in response (0 = success)
    └── Return response body
```

### 3. Private Request Flow

```
doPrivateRequest(ctx, method, endpoint, params, jsonBody)
    │
    ├── Generate timestamp (Unix milliseconds)
    ├── Build payload:
    │   ├── GET: sorted query string
    │   ├── POST with JSON: JSON body string
    │   └── POST with form: sorted query string
    ├── Generate HMAC-SHA256 signature:
    │   └── HMAC(timestamp + apiKey + recvWindow + payload)
    ├── Set headers:
    │   ├── X-BAPI-API-KEY = apiKey
    │   ├── X-BAPI-TIMESTAMP = timestamp
    │   ├── X-BAPI-SIGN = signature
    │   └── X-BAPI-RECV-WINDOW = recvWindow
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Check retCode in response (0 = success)
    └── Return response body
```

### 4. Get Ticker Flow

```
GetTicker(ctx, category, symbol)
    │
    ├── Build params: category, symbol
    ├── GET /v5/market/tickers
    ├── Parse response: result.list[0]
    └── Return *BybitTicker
```

### 5. Place Order Flow

```
PlaceOrder(ctx, req)
    │
    ├── Marshal request to JSON
    ├── POST /v5/order/create
    ├── Parse response: result (BybitOrder)
    └── Return *BybitOrder
```

### 6. Get Status Flow

```
GetStatus()
    │
    ├── If nil: return connected=false
    ├── GetServerTime(ctx)
    ├── Return connected, base_url, testnet, api_key_configured, pool_active
    └── Note: Tests actual API connectivity
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `bybit.enabled` | bool | false | Enable/disable Bybit manager |
| `bybit.api_key` | string | "" | Bybit API key |
| `bybit.api_secret` | string | "" | Bybit API secret |
| `bybit.testnet` | bool | false | Use testnet instead of production |
| `bybit.recv_window` | int | 5000 | Request timeout window in ms |

### Environment Variables

The Bybit manager does not directly use environment variables. Configuration is read through Viper, which can load from:
- Configuration files (JSON, YAML, TOML)
- Environment variables (if Viper is configured for it)

You can set Bybit parameters via:
```go
viper.Set("bybit.enabled", true)
viper.Set("bybit.api_key", "your_api_key")
viper.Set("bybit.api_secret", "your_api_secret")
viper.Set("bybit.testnet", false)
viper.Set("bybit.recv_window", 5000)
```

## Usage Examples

### Getting Ticker Data

```go
ctx := context.Background()

// Get ticker for ETHUSDT (spot category)
ticker, err := manager.GetTicker(ctx, "spot", "ETHUSDT")
if err != nil {
    panic(err)
}

fmt.Printf("Symbol: %s\n", ticker.Symbol)
fmt.Printf("Last Price: %s\n", ticker.LastPrice)
fmt.Printf("24h Change: %s%%\n", ticker.Price24hPcnt)
fmt.Printf("24h High: %s, Low: %s\n", ticker.HighPrice24h, ticker.LowPrice24h)
fmt.Printf("Volume: %s\n", ticker.Volume24h)
fmt.Printf("Funding Rate: %s\n", ticker.FundingRate)
```

### Getting Kline Data

```go
ctx := context.Background()

// Get 1-hour klines for BTCUSDT (linear category), limit 100
klines, err := manager.GetKlines(ctx, "linear", "BTCUSDT", "60", 100)
if err != nil {
    panic(err)
}

for _, k := range klines {
    fmt.Printf("Time: %s, Open: %s, High: %s, Low: %s, Close: %s, Volume: %s\n",
        k.StartTime, k.Open, k.High, k.Low, k.Close, k.Volume)
}
```

### Getting Order Book

```go
ctx := context.Background()

// Get order book for BTCUSDT (linear category), limit 50
orderBook, err := manager.GetOrderBook(ctx, "linear", "BTCUSDT", 50)
if err != nil {
    panic(err)
}

fmt.Printf("Order Book for %s (Time: %d)\n", orderBook.Symbol, orderBook.Time)
fmt.Println("Bids:")
for i, bid := range orderBook.Bids {
    if i >= 5 { break } // Show top 5
    fmt.Printf("  Price: %s, Size: %s\n", bid[0], bid[1])
}
fmt.Println("Asks:")
for i, ask := range orderBook.Asks {
    if i >= 5 { break }
    fmt.Printf("  Price: %s, Size: %s\n", ask[0], ask[1])
}
```

### Getting Wallet Balance

```go
ctx := context.Background()

// Get wallet balance for UNIFIED account
balances, err := manager.GetWalletBalance(ctx, "UNIFIED", "")
if err != nil {
    panic(err)
}

for _, balance := range balances {
    fmt.Printf("Coin: %s, Equity: %s, Available: %s, PnL: %s\n",
        balance.Coin, balance.Equity, balance.AvailableToWithdraw, balance.UnrealisedPnl)
}
```

### Getting Positions

```go
ctx := context.Background()

// Get positions for BTCUSDT (linear category)
positions, err := manager.GetPositions(ctx, "linear", "BTCUSDT")
if err != nil {
    panic(err)
}

for _, pos := range positions {
    fmt.Printf("Symbol: %s, Side: %s, Size: %s, Entry: %s, Mark: %s, PnL: %s\n",
        pos.Symbol, pos.Side, pos.Size, pos.EntryPrice, pos.MarkPrice, pos.UnrealisedPnl)
}
```

### Placing an Order

```go
ctx := context.Background()

// Place a limit buy order for BTCUSDT
orderReq := &bybit.BybitPlaceOrderRequest{
    Category: "linear",
    Symbol:   "BTCUSDT",
    Side:     "Buy",
    OrderType: "Limit",
    Qty:      "0.001",
    Price:    "50000",
}

order, err := manager.PlaceOrder(ctx, orderReq)
if err != nil {
    panic(err)
}

fmt.Printf("Order placed: ID=%s, Status=%s, Price=%s, Qty=%s\n",
    order.OrderID, order.OrderStatus, order.Price, order.Qty)
```

### Canceling an Order

```go
ctx := context.Background()

// Cancel an order by order ID
cancelReq := &bybit.BybitCancelOrderRequest{
    Category: "linear",
    Symbol:   "BTCUSDT",
    OrderID:  "123456789",
}

cancelled, err := manager.CancelOrder(ctx, cancelReq)
if err != nil {
    panic(err)
}

fmt.Printf("Order cancelled: ID=%s, Status=%s\n", cancelled.OrderID, cancelled.OrderStatus)
```

### Setting Leverage

```go
ctx := context.Background()

// Set leverage for BTCUSDT (linear category)
leverageReq := &bybit.BybitSetLeverageRequest{
    Category:    "linear",
    Symbol:      "BTCUSDT",
    BuyLeverage:  "10",
    SellLeverage: "10",
}

err := manager.SetLeverage(ctx, leverageReq)
if err != nil {
    panic(err)
}
fmt.Println("Leverage set to 10x")
```

### Async Operations

```go
ctx := context.Background()

// Get ticker asynchronously
asyncResult := manager.GetTickerAsync(ctx, "linear", "BTCUSDT")
result, err := asyncResult.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Async BTC Price: %s\n", result.LastPrice)

// Get order book asynchronously
obResult := manager.GetOrderBookAsync(ctx, "linear", "BTCUSDT", 50)
orderBook, err := obResult.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Async Order Book has %d bids\n", len(orderBook.Bids))
```

### Getting Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Base URL: %s\n", status["base_url"])
fmt.Printf("Testnet: %v\n", status["testnet"])
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
| Standard library | `crypto/hmac`, `crypto/sha256`, `encoding/hex`, `encoding/json`, `net/http`, `net/url`, `strconv`, `time`, `sort`, `bytes`, `strings` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: Returned from API ping
- **Authentication errors**: Returned from authenticated endpoints
- **API errors**: Checked via `retCode` in response (0 = success)
- **Rate limiting**: Handled by retry logic
- **Nil checks**: Public methods check manager state

Example error handling:

```go
ticker, err := manager.GetTicker(ctx, category, symbol)
if err != nil {
    if strings.Contains(err.Error(), "10001") {
        return fmt.Errorf("invalid parameter: %w", err)
    }
    if strings.Contains(err.Error(), "10002") {
        return fmt.Errorf("authentication failed: %w", err)
    }
    if strings.Contains(err.Error(), "10006") {
        return fmt.Errorf("rate limit exceeded: %w", err)
    }
    return fmt.Errorf("API request failed: %w", err)
}
```

## Common Pitfalls

### 1. Manager Not Enabled

**Problem**: `bybit.enabled` is false

**Solution**:
- Set `bybit.enabled = true` in configuration
- Check viper configuration is loaded

### 2. API Credentials Missing

**Problem**: `api_key` or `api_secret` not set

**Solution**:
- Set `bybit.api_key` and `bybit.api_secret`
- For public endpoints (ticker, klines, orderbook), credentials are not required
- For account/order endpoints, credentials are mandatory

### 3. Invalid Category or Symbol

**Problem**: `Invalid category` or `Invalid symbol` error

**Solution**:
- Check category: "spot", "linear", "option", "inverse"
- Check symbol format (e.g., "BTCUSDT", "ETHUSDT")
- Verify symbol exists on Bybit for the given category

### 4. Signature Generation Failure

**Problem**: Authentication failed (retCode 10002)

**Solution**:
- Verify `api_secret` is correct
- Ensure timestamp is included in signature
- Check signature generation logic
- Verify recv_window is reasonable (default 5000ms)

### 5. Rate Limiting

**Problem**: Rate limit exceeded (retCode 10006)

**Solution**:
- Library retries up to 3 times
- Consider using async operations to spread requests
- Respect Bybit rate limits per category

### 6. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewBybitManager()` succeeded
- Verify `bybit.enabled` is true

### 7. Position Mode

**Problem**: Cannot place hedge mode orders

**Solution**:
- Switch position mode using `SwitchPositionMode()`
- Mode 0 = One-Way, Mode 1 = Hedge
- Some categories only support specific modes

## Advanced Usage

### Custom API Request

```go
func customRequest(manager *bybit.BybitManager) error {
    ctx := context.Background()
    
    // Build custom query
    params := url.Values{}
    params.Set("category", "linear")
    params.Set("symbol", "BTCUSDT")
    
    // Use doPublicRequest for public endpoints
    respBody, err := manager.DoPublicRequest(ctx, "GET", "/v5/market/tickers", params)
    if err != nil {
        return err
    }
    
    // Process response...
    fmt.Printf("Response: %s\n", string(respBody))
    return nil
}
```

### Health Check

```go
func healthCheck(manager *bybit.BybitManager) error {
    status := manager.GetStatus()
    if status["connected"] == false {
        return fmt.Errorf("not connected to Bybit API")
    }
    fmt.Printf("Bybit API is healthy (URL: %s)\n", status["base_url"])
    return nil
}
```

### Using Testnet

```go
// Set testnet mode
viper.Set("bybit.testnet", true)
viper.Set("bybit.api_key", "test_api_key")
viper.Set("bybit.api_secret", "test_api_secret")

// Initialize manager (will use testnet URL)
manager, err := bybit.NewBybitManager(log)
if err != nil {
    panic(err)
}
// Now all requests go to api-testnet.bybit.com
```

### Batch Operations with Async

```go
func getMultipleTickers(manager *bybit.BybitManager, category string, symbols []string) ([]*bybit.BybitTicker, error) {
    ctx := context.Background()
    results := make([]*bybit.BybitTicker, len(symbols))
    errors := make([]error, len(symbols))
    
    // Submit all requests asynchronously
    for i, symbol := range symbols {
        func(idx int, sym string) {
            manager.SubmitAsyncJob(func() {
                ticker, err := manager.GetTicker(ctx, category, sym)
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

### Switch Position Mode

```go
ctx := context.Background()

// Switch to Hedge Mode for linear category
switchReq := &bybit.BybitSwitchPositionModeRequest{
    Category: "linear",
    Mode:     1, // 0 = One-Way, 1 = Hedge
}

err := manager.SwitchPositionMode(ctx, switchReq)
if err != nil {
    panic(err)
}
fmt.Println("Position mode switched to Hedge")
```

### Get Execution History

```go
ctx := context.Background()

// Get execution history for linear category
executions, err := manager.GetExecutionList(ctx, "linear", "BTCUSDT", 50)
if err != nil {
    panic(err)
}

for _, exec := range executions {
    fmt.Printf("Exec: %s, Symbol: %s, Side: %s, Price: %s, Qty: %s, Fee: %s %s\n",
        exec.ExecID, exec.Symbol, exec.Side, exec.ExecPrice, exec.ExecQty, exec.FeeRate, exec.FeeToken)
}
```

### Cancel All Orders

```go
ctx := context.Background()

// Cancel all orders for BTCUSDT (linear category)
err := manager.CancelAllOrders(ctx, "linear", "BTCUSDT")
if err != nil {
    panic(err)
}
fmt.Println("All orders cancelled")
```

## Internal Algorithms

### Signature Generation

```
generateSignature(timestamp, payload)
    │
    ├── Create HMAC-SHA256 with api_secret as key
    ├── Write: timestamp + apiKey + recvWindow + payload
    ├── Compute sum
    └── Return hex-encoded string
```

### Query String Building

```
buildQueryString(params)
    │
    ├── Extract all keys
    ├── Sort keys alphabetically
    ├── URL-encode each key=value pair
    ├── Join with "&"
    └── Return sorted query string
```

### Connection Test

```
testConnection():
    │
    ├── GET /v5/market/time
    ├── Check status code
    └── Return error if not 200
```

### Response Parsing

```
All API responses wrapped in BybitResponse:
{
  "retCode": 0,
  "retMsg": "OK",
  "result": { ... },
  "time": 1234567890
}

Check retCode == 0 for success.
```

## API Categories

Bybit v5 API uses categories to distinguish different product types:

| Category | Description | Examples |
|----------|-------------|---------|
| `spot` | Spot trading | BTCUSDT, ETHUSDT |
| `linear` | USDT-margined futures | BTCUSDT, ETHUSDT |
| `inverse` | Coin-margined futures | BTCUSD, ETHUSD |
| `option` | Options trading | BTC-29APR22-40000-C |

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.