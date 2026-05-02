# CoinMarketCap Manager

## Overview

The `CoinMarketCapManager` is a Go library for interacting with the CoinMarketCap API v1. It provides cryptocurrency market data, quotes, global metrics, and metadata with API key authentication. The library offers comprehensive market information including prices, market caps, volume data, and cryptocurrency metadata with async support via worker pool.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Market Data**: Get latest cryptocurrency listings with rankings
- **Quotes**: Get real-time price quotes for specific cryptocurrencies
- **Global Metrics**: Access global market metrics (total market cap, BTC dominance, etc.)
- **Metadata**: Retrieve detailed cryptocurrency metadata (description, tags, logos, URLs)
- **Price Lookup**: Simple price retrieval by symbol
- **API Key Authentication**: Secure API access with X-CMC_PRO_API_KEY header
- **Retry Logic**: HTTP client with retry support (3 retries)
- **Worker Pool**: Async job execution support (4 workers)
- **Sandbox Support**: Switch between production and sandbox environments
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
    
    // Create CoinMarketCap manager (configuration via viper)
    manager, err := coinmarketcap.NewCoinMarketCapManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Get Bitcoin price in USD
    price, err := manager.GetSimplePrice(ctx, "BTC", "USD")
    if err != nil {
        panic(err)
    }
    fmt.Printf("BTC Price: $%.2f\n", price)
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `CoinMarketCapManager` | Main manager with HTTP client, API key, worker pool |
| `CMCCryptocurrency` | Cryptocurrency data with rankings and quotes |
| `CMCQuote` | Market quote data (price, volume, market cap, changes) |
| `CMCGlobalMetrics` | Global market metrics (BTC dominance, total market cap) |
| `CMCMetadata` | Cryptocurrency metadata (description, tags, logos) |
| `CMCInfo` | API response status information |
| `CMCResponse` | Standard API response wrapper |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   CoinMarketCapManager                     │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  retryablehttp │  → API requests with retry               │
│  │  .Client      │  → X-CMC_PRO_API_KEY authentication    │
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
│  BaseURL: Production or Sandbox                           │
│  APIKey: For all authenticated endpoints                   │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewCoinMarketCapManager(logger)
    │
    ├── Check viper config: "coinmarketcap.enabled"
    ├── Get api_key: "coinmarketcap.api_key"
    ├── Get sandbox: "coinmarketcap.sandbox" (default: false)
    │
    ├── Set BaseURL:
    │   ├── Production: "https://pro-api.coinmarketcap.com/v1"
    │   └── Sandbox: "https://sandbox-api.coinmarketcap.com/v1"
    │
    ├── Create retryablehttp.Client:
    │   ├── RetryMax = 3
    │   ├── RetryWaitMin = 1s
    │   ├── RetryWaitMax = 10s
    │   └── HTTPClient.Timeout = 30s
    │
    ├── Test connection: testConnection()
    │   └── GET /key/info
    │
    ├── Create WorkerPool(4)
    ├── Start worker pool
    └── Return CoinMarketCapManager
```

### 2. Get Latest Listings Flow

```
GetLatestListings(ctx, start, limit, convert)
    │
    ├── Build query params: start, limit, convert
    ├── Create GET request: /cryptocurrency/listings/latest
    ├── Set header: X-CMC_PRO_API_KEY = apiKey
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON into []CMCCryptocurrency
    └── Return []CMCCryptocurrency
```

### 3. Get Quotes Flow

```
GetQuotes(ctx, symbols, convert)
    │
    ├── Build query params: symbol (comma-separated), convert
    ├── Create GET request: /cryptocurrency/quotes/latest
    ├── Set header: X-CMC_PRO_API_KEY = apiKey
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON into map[string]CMCCryptocurrency
    └── Return map[string]CMCCryptocurrency
```

### 4. Get Global Metrics Flow

```
GetGlobalMetrics(ctx, convert)
    │
    ├── Build query params: convert
    ├── Create GET request: /global-metrics/quotes/latest
    ├── Set header: X-CMC_PRO_API_KEY = apiKey
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON into CMCGlobalMetrics
    └── Return *CMCGlobalMetrics
```

### 5. Get Metadata Flow

```
GetMetadata(ctx, symbols)
    │
    ├── Build query params: symbol (comma-separated)
    ├── Create GET request: /cryptocurrency/info
    ├── Set header: X-CMC_PRO_API_KEY = apiKey
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON into map[string]CMCMetadata
    └── Return map[string]CMCMetadata
```

### 6. Get Price By Ticker Flow

```
GetPriceByTicker(ctx, symbol, convert)
    │
    ├── Convert symbol to uppercase
    ├── Call GetQuotes(ctx, [symbol], convert)
    ├── Extract cryptocurrency from result
    ├── Extract quote for convert currency
    └── Return price (float64), *CMCCryptocurrency, error
```

### 7. Get Status Flow

```
GetStatus()
    │
    ├── If nil: return connected=false
    ├── Create context with 5s timeout
    ├── GET /key/info
    ├── Set header: X-CMC_PRO_API_KEY = apiKey
    ├── Return connected, base_url, api_key_configured, pool_active
    └── Note: Tests actual API connectivity
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `coinmarketcap.enabled` | bool | false | Enable/disable CoinMarketCap manager |
| `coinmarketcap.api_key` | string | "" | CoinMarketCap API key |
| `coinmarketcap.sandbox` | bool | false | Use sandbox instead of production |

### Environment Variables

The CoinMarketCap manager does not directly use environment variables. Configuration is read through Viper, which can load from:
- Configuration files (JSON, YAML, TOML)
- Environment variables (if Viper is configured for it)

You can set CoinMarketCap parameters via:
```go
viper.Set("coinmarketcap.enabled", true)
viper.Set("coinmarketcap.api_key", "your_api_key")
viper.Set("coinmarketcap.sandbox", false)
```

## Usage Examples

### Getting Latest Listings

```go
ctx := context.Background()

// Get top 10 cryptocurrencies (start from 1, limit 10)
listings, err := manager.GetLatestListings(ctx, 1, 10, "USD")
if err != nil {
    panic(err)
}

for _, crypto := range listings {
    quote := crypto.Quote["USD"]
    fmt.Printf("%d. %s (%s): $%.2f (24h: %.2f%%)\n", 
        crypto.Rank, crypto.Name, crypto.Symbol, 
        quote.Price, quote.PercentChange24h)
}
```

### Getting Quotes

```go
ctx := context.Background()

// Get quotes for Bitcoin and Ethereum in USD
quotes, err := manager.GetQuotes(ctx, []string{"BTC", "ETH"}, "USD")
if err != nil {
    panic(err)
}

for symbol, crypto := range quotes {
    quote := crypto.Quote["USD"]
    fmt.Printf("%s: $%.2f (Market Cap: $%.0fB)\n", 
        symbol, quote.Price, quote.MarketCap/1e9)
}
```

### Getting Global Metrics

```go
ctx := context.Background()

metrics, err := manager.GetGlobalMetrics(ctx, "USD")
if err != nil {
    panic(err)
}

fmt.Printf("Active Cryptocurrencies: %d\n", metrics.ActiveCryptocurrencies)
fmt.Printf("Active Exchanges: %d\n", metrics.ActiveExchanges)
fmt.Printf("BTC Dominance: %.2f%%\n", metrics.BTCDominance)
fmt.Printf("ETH Dominance: %.2f%%\n", metrics.ETHDominance)
fmt.Printf("Total Market Cap: $%.0fB\n", metrics.TotalMarketCap["USD"])
fmt.Printf("Total Volume 24h: $%.0fB\n", metrics.TotalVolume24h["USD"])
```

### Getting Metadata

```go
ctx := context.Background()

// Get metadata for Bitcoin
metadata, err := manager.GetMetadata(ctx, []string{"BTC"})
if err != nil {
    panic(err)
}

if btc, exists := metadata["BTC"]; exists {
    fmt.Printf("Name: %s\n", btc.Name)
    fmt.Printf("Symbol: %s\n", btc.Symbol)
    fmt.Printf("Description: %s\n", btc.Description)
    fmt.Printf("Category: %s\n", btc.Category)
    fmt.Printf("Tags: %v\n", btc.Tags)
    if logo, exists := btc.URLs["logo"]; exists && len(logo) > 0 {
        fmt.Printf("Logo: %s\n", logo[0])
    }
}
```

### Getting Price by Ticker

```go
ctx := context.Background()

// Get Bitcoin price with details
price, crypto, err := manager.GetPriceByTicker(ctx, "BTC", "USD")
if err != nil {
    panic(err)
}

fmt.Printf("BTC Price: $%.2f\n", price)
fmt.Printf("Rank: %d\n", crypto.Rank)
fmt.Printf("Market Cap: $%.0fB\n", crypto.Quote["USD"].MarketCap/1e9)
fmt.Printf("24h Volume: $%.0fB\n", crypto.Quote["USD"].Volume24h/1e9)
fmt.Printf("Circulating Supply: %.0fM\n", crypto.CirculatingSupply/1e6)
```

### Simple Price Lookup

```go
ctx := context.Background()

// Get just the price (simpler)
price, err := manager.GetSimplePrice(ctx, "ETH", "USD")
if err != nil {
    panic(err)
}
fmt.Printf("ETH Price: $%.2f\n", price)
```

### Async Operations

```go
ctx := context.Background()

// Get listings asynchronously
listingsResult := manager.GetLatestListingsAsync(ctx, 1, 20, "USD")
listings, err := listingsResult.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Got %d listings asynchronously\n", len(listings))

// Get quotes asynchronously
quotesResult := manager.GetQuotesAsync(ctx, []string{"BTC", "ETH", "BNB"}, "USD")
quotes, err := quotesResult.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Got quotes for %d cryptos\n", len(quotes))

// Get global metrics asynchronously
metricsResult := manager.GetGlobalMetricsAsync(ctx, "USD")
metrics, err := metricsResult.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("BTC Dominance: %.2f%%\n", metrics.BTCDominance)
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
| Standard library | `encoding/json`, `net/http`, `net/url`, `strconv`, `strings`, `time` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: Returned from API ping
- **Authentication errors**: Returned from API requests (invalid API key)
- **Rate limiting**: Handled by retry logic
- **Nil checks**: Public methods check manager state

Example error handling:

```go
listings, err := manager.GetLatestListings(ctx, start, limit, convert)
if err != nil {
    if strings.Contains(err.Error(), "401") {
        return fmt.Errorf("invalid API key: %w", err)
    }
    if strings.Contains(err.Error(), "429") {
        return fmt.Errorf("rate limit exceeded: %w", err)
    }
    if strings.Contains(err.Error(), "400") {
        return fmt.Errorf("invalid parameters: %w", err)
    }
    return fmt.Errorf("API request failed: %w", err)
}
```

## Common Pitfalls

### 1. Manager Not Enabled

**Problem**: `coinmarketcap.enabled` is false

**Solution**:
- Set `coinmarketcap.enabled = true` in configuration
- Check viper configuration is loaded

### 2. API Key Missing

**Problem**: `api_key` not set

**Solution**:
- Set `coinmarketcap.api_key` to your CoinMarketCap API key
- All endpoints require API key authentication
- Get API key from https://coinmarketcap.com/api/

### 3. Invalid Symbol

**Problem**: `Cryptocurrency symbol not found` error

**Solution**:
- Check symbol format (e.g., "BTC", "ETH", "BNB")
- Use uppercase symbols
- Verify symbol exists on CoinMarketCap

### 4. Rate Limiting

**Problem**: `429` Too Many Requests

**Solution**:
- Library retries up to 3 times
- Consider using async operations to spread requests
- Respect CoinMarketCap rate limits (varies by plan)

### 5. Invalid Convert Currency

**Problem**: `Invalid convert currency` error

**Solution**:
- Use valid fiat currencies: "USD", "EUR", "GBP", etc.
- Use valid cryptocurrencies: "BTC", "ETH", etc.
- Check https://coinmarketcap.com/api/documentation/ for supported currencies

### 6. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewCoinMarketCapManager()` succeeded
- Verify `coinmarketcap.enabled` is true

### 7. Sandbox Mode

**Problem**: API calls not working in sandbox

**Solution**:
- Set `coinmarketcap.sandbox = true`
- Ensure you have sandbox API key (different from production)
- Sandbox URL: https://sandbox-api.coinmarketcap.com

## Advanced Usage

### Custom API Request

```go
func customRequest(manager *coinmarketcap.CoinMarketCapManager) error {
    ctx := context.Background()
    
    // Build custom query
    params := url.Values{}
    params.Set("start", "1")
    params.Set("limit", "5")
    params.Set("convert", "USD")
    
    // Create request
    req, err := retryablehttp.NewRequestWithContext(ctx, "GET", 
        fmt.Sprintf("%s/cryptocurrency/listings/latest?%s", manager.BaseURL, params.Encode()), nil)
    if err != nil {
        return err
    }
    
    req.Header.Set("X-CMC_PRO_API_KEY", manager.APIKey)
    req.Header.Set("Accept", "application/json")
    
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
func healthCheck(manager *coinmarketcap.CoinMarketCapManager) error {
    status := manager.GetStatus()
    if status["connected"] == false {
        return fmt.Errorf("not connected to CoinMarketCap API")
    }
    fmt.Printf("CoinMarketCap API is healthy (URL: %s)\n", status["base_url"])
    return nil
}
```

### Using Sandbox

```go
// Set sandbox mode
viper.Set("coinmarketcap.sandbox", true)
viper.Set("coinmarketcap.api_key", "sandbox_api_key")

// Initialize manager (will use sandbox URL)
manager, err := coinmarketcap.NewCoinMarketCapManager(log)
if err != nil {
    panic(err)
}
// Now all requests go to sandbox-api.coinmarketcap.com
```

### Batch Operations with Async

```go
func getMultipleQuotes(manager *coinmarketcap.CoinMarketCapManager, symbols []string) (map[string]*coinmarketcap.CMCCryptocurrency, error) {
    ctx := context.Background()
    
    // Submit async job
    result := manager.GetQuotesAsync(ctx, symbols, "USD")
    quotes, err := result.Get(ctx)
    if err != nil {
        return nil, err
    }
    
    return quotes, nil
}
```

### Price Tracking

```go
func trackPrice(manager *coinmarketcap.CoinMarketCapManager, symbol string) {
    ctx := context.Background()
    
    for {
        price, err := manager.GetSimplePrice(ctx, symbol, "USD")
        if err != nil {
            fmt.Printf("Error getting price: %v\n", err)
            time.Sleep(1 * time.Minute)
            continue
        }
        
        fmt.Printf("[%s] %s Price: $%.2f\n", time.Now().Format("15:04:05"), symbol, price)
        time.Sleep(1 * time.Minute)
    }
}
```

### Getting Multiple Convert Currencies

```go
ctx := context.Background()

// Get quotes in multiple currencies (separate calls)
for _, convert := range []string{"USD", "EUR", "GBP"} {
    quotes, err := manager.GetQuotes(ctx, []string{"BTC", "ETH"}, convert)
    if err != nil {
        panic(err)
    }
    
    for symbol, crypto := range quotes {
        quote := crypto.Quote[convert]
        fmt.Printf("%s in %s: $%.2f\n", symbol, convert, quote.Price)
    }
}
```

## Internal Algorithms

### API Key Authentication

```
All API requests include header:
X-CMC_PRO_API_KEY: <api_key>

Set in testConnection() and all doPublicRequest() calls.
```

### Connection Test

```
testConnection():
    │
    ├── GET /key/info
    ├── Set X-CMC_PRO_API_KEY header
    ├── Check status code
    └── Return error if not 200
```

### Response Parsing

```
All API responses have structure:
{
  "status": {
    "timestamp": ...,
    "error_code": 0,
    "error_message": "",
    "elapsed": ...,
    "credit_count": ...
  },
  "data": { ... }
}

Check status.error_code == 0 for success.
```

### Symbol Matching

```
GetPriceByTicker():
    │
    ├── Convert symbol to uppercase
    ├── Call GetQuotes() with []string{symbol}
    ├── Extract from map using symbol as key
    ├── Get quote for specified convert currency
    └── Return price and cryptocurrency details
```

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/cryptocurrency/listings/latest` | GET | Get latest cryptocurrency listings |
| `/cryptocurrency/quotes/latest` | GET | Get quotes for specific cryptocurrencies |
| `/cryptocurrency/info` | GET | Get metadata for cryptocurrencies |
| `/global-metrics/quotes/latest` | GET | Get global market metrics |
| `/key/info` | GET | Get API key information |

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.