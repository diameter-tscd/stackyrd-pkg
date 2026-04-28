# Cloudflare Manager

## Overview

The `CloudflareManager` is a comprehensive Go library for managing Cloudflare resources via the Cloudflare API (v4). It provides complete Cloudflare resource management including zones, DNS records, cache purge, SSL certificates, page rules, firewall rules, analytics, and workers with async support.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Zone Management**: List and get zone details
- **DNS Management**: List, create, update, and delete DNS records
- **Cache Management**: Purge cache (individual files, tags, hosts, or everything)
- **SSL/TLS Certificates**: List SSL certificates for zones
- **Page Rules**: List page rules for zones
- **Firewall Rules**: List firewall rules for zones
- **Analytics**: Get analytics data (requests, bandwidth, page views, threats, cached requests/bytes, SSL requests)
- **Workers**: List Cloudflare Workers
- **Async Operations**: Async execution for all operations via worker pool
- **Authentication**: Uses Cloudflare API key and email via X-Auth-Key and X-Auth-Email headers
- **Retry Logic**: Automatic retries with exponential backoff via `go-retryablehttp`

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
    
    // Create Cloudflare manager (configuration via viper)
    manager, err := cloudflare.NewCloudflareManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // List Cloudflare zones
    zones, err := manager.ListZones(ctx)
    if err != nil {
        panic(err)
    }
    
    for _, zone := range zones {
        fmt.Printf("Zone: %s (ID: %s)\n", zone.Name, zone.ID)
    }
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `CloudflareManager` | Main manager with HTTP client and worker pool |
| `CloudflareZone` | Cloudflare zone with ID and name |
| `CloudflareDNSRecord` | DNS record with type, name, content, proxy settings |
| `CloudflareCachePurgeRequest` | Cache purge request (files, tags, hosts, purge_all) |
| `CloudflareCertificate` | SSL certificate with hosts, issuer, status, expiration |
| `CloudflarePageRule` | Page rule with targets, actions, priority, status |
| `CloudflareFirewallRule` | Firewall rule with expression, action, priority |
| `CloudflareAnalytics` | Analytics data (requests, bandwidth, page views, etc.) |
| `CloudflareWorker` | Cloudflare Worker with script name, size, usage |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   CloudflareManager                         │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │ retryablehttp  │  → Retry logic (max 3, 1-10s wait)│
│  │ Client         │  → HTTP timeout: 30s                   │
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
│  BaseURL: https://api.cloudflare.com/client/v4          │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewCloudflareManager()
    │
    ├── Check viper config: "cloudflare.enabled"
    ├── Get credentials: api_key, email, account_id
    ├── Create retryablehttp.Client (max 3 retries, 1-10s wait, 30s timeout)
    ├── Set logger adapter
    ├── Build manager with BaseURL "https://api.cloudflare.com/client/v4"
    ├── Test connection: testConnection() → GET /user
    ├── Create WorkerPool(4)
    ├── Start worker pool
    └── Return CloudflareManager
```

### 2. API Request Flow

```
Generic API Request (GET/POST/PUT/DELETE)
    │
    ├── Create request with context
    ├── Set auth headers: X-Auth-Key, X-Auth-Email
    ├── Set Content-Type: application/json (for POST/PUT)
    ├── Execute request via retryablehttp.Client
    ├── Read response body
    ├── Check status code (200 OK = success)
    └── JSON decode into result struct
```

### 3. List Zones Flow

```
ListZones(ctx)
    │
    ├── Create GET request to /zones
    ├── Set auth headers
    ├── Execute request
    ├── Check status code
    ├── JSON decode to struct with Result []CloudflareZone
    └── Return []CloudflareZone
```

### 4. DNS Record Operations Flow

```
CreateDNSRecord(ctx, zoneID, record)
    │
    ├── JSON marshal record
    ├── Create POST request to /zones/{zoneID}/dns_records
    ├── Set auth headers and Content-Type
    ├── Execute request
    ├── Check status code (200 or 201)
    ├── JSON decode to struct with Result CloudflareDNSRecord
    └── Return created record
```

### 5. Cache Purge Flow

```
PurgeCache(ctx, zoneID, purgeRequest)
    │
    ├── JSON marshal purgeRequest
    ├── Create POST request to /zones/{zoneID}/purge_cache
    ├── Set auth headers and Content-Type
    ├── Execute request
    ├── Check status code
    └── Return error or nil
```

### 6. Analytics Flow

```
GetAnalytics(ctx, zoneID, since, until)
    │
    ├── Build query params: since, until (RFC3339)
    ├── Create GET request to /zones/{zoneID}/analytics/dashboard?params
    ├── Set auth headers
    ├── Execute request
    ├── Check status code
    ├── JSON decode to struct with timeseries data
    ├── Sum all timeseries entries
    └── Return aggregated CloudflareAnalytics
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `cloudflare.enabled` | bool | false | Enable/disable Cloudflare manager |
| `cloudflare.api_key` | string | "" | Cloudflare API key |
| `cloudflare.email` | string | "" | Cloudflare account email |
| `cloudflare.account_id` | string | "" | Cloudflare account ID (for Workers) |

### Environment Variables

The Cloudflare manager does not directly use environment variables. Configuration is read through Viper, which can load from:
- Configuration files (JSON, YAML, TOML)
- Environment variables (if Viper is configured for it)

You can set Cloudflare credentials via:
```go
viper.Set("cloudflare.api_key", "your-api-key")
viper.Set("cloudflare.email", "your-email@example.com")
```

## Usage Examples

### Zone Operations

```go
ctx := context.Background()

// List all zones
zones, err := manager.ListZones(ctx)
if err != nil {
    panic(err)
}

for _, zone := range zones {
    fmt.Printf("Zone: %s (ID: %s)\n", zone.Name, zone.ID)
}

// Get zone details
zone, err := manager.GetZone(ctx, "zone-id-123")
if err != nil {
    panic(err)
}
fmt.Printf("Zone: %s, Name: %s\n", zone.ID, zone.Name)
```

### DNS Record Operations

```go
ctx := context.Background()

// List DNS records for a zone
records, err := manager.ListDNSRecords(ctx, "zone-id-123")
if err != nil {
    panic(err)
}

for _, record := range records {
    fmt.Printf("Record: %s (Type: %s, Name: %s, Content: %s)\n",# 
        record.ID, record.Type, record.Name, record.Content)
    fmt.Printf("  Proxied: %v, TTL: %d\n", record.Proxied, record.TTL)
}

// Create a DNS record (A record)
newRecord := cloudflare.CloudflareDNSRecord{
    Type:    "A",
    Name:    "example.com",
    Content: "192.0.2.1",
    Proxied: true,
    TTL:     1, // Auto TTL
}

created, err := manager.CreateDNSRecord(ctx, "zone-id-123", newRecord)
if err != nil {
    panic(err)
}
fmt.Printf("Created record: %s\n", created.ID)

// Update a DNS record
updatedRecord := cloudflare.CloudflareDNSRecord{
    Type:    "A",
    Name:    "example.com",
    Content: "192.0.2.2",
    Proxied: true,
    TTL:     1,
}

updated, err := manager.UpdateDNSRecord(ctx, "zone-id-123", "record-id-456", updatedRecord)
if err != nil {
    panic(err)
}
fmt.Printf("Updated record: %s\n", updated.ID)

// Delete a DNS record
err = manager.DeleteDNSRecord(ctx, "zone-id-123", "record-id-456")
if err != nil {
    panic(err)
}
fmt.Println("DNS record deleted")
```

### Cache Purge Operations

```go
ctx := context.Background()

// Purge specific files
purgeReq := cloudflare.CloudflareCachePurgeRequest{
    Files: []string{"http://example.com/css/styles.css", "http://example.com/js/app.js"},
}

err := manager.PurgeCache(ctx, "zone-id-123", purgeReq)
if err != nil {
    panic(err)
}
fmt.Println("Cache purged for specific files")

// Purge by tags
purgeReq = cloudflare.CloudflareCachePurgeRequest{
    Tags: []string{"tag1", "tag2"},
}

err = manager.PurgeCache(ctx, "zone-id-123", purgeReq)
if err != nil {
    panic(err)
}
fmt.Println("Cache purged for tags")

// Purge by hosts
purgeReq = cloudflare.CloudflareCachePurgeRequest{
    Hosts: []string{"images.example.com", "static.example.com"},
}

err = manager.PurgeCache(ctx, "zone-id-123", purgeReq)
if err != nil {
    panic(err)
}
fmt.Println("Cache purged for hosts")

// Purge all cache (be careful!)
err = manager.PurgeAllCache(ctx, "zone-id-123")
if err != nil {
    panic(err)
}
fmt.Println("All cache purged")
```

### SSL Certificate Operations

```go
ctx := context.Background()

// List SSL certificates for a zone
certs, err := manager.ListCertificates(ctx, "zone-id-123")
if err != nil {
    panic(err)
}

for _, cert := range certs {
    fmt.Printf("Certificate: %s (Type: %s, Status: %s)\n", cert.ID, cert.Type, cert.Status)
    fmt.Printf("  Hosts: %v\n", cert.Hosts)
    fmt.Printf("  Issuer: %s, Expires: %v\n", cert.Issuer, cert.ExpiresOn)
}
```

### Page Rule Operations

```go
ctx := context.Background()

// List page rules for a zone
rules, err := manager.ListPageRules(ctx, "zone-id-123")
if err != nil {
    panic(err)
}

for _, rule := range rules {
    fmt.Printf("Page Rule: %s (Priority: %d, Status: %s)\n", #rule.ID, rule.Priority, rule.Status)
}
```

### Firewall Rule Operations

```go
ctx := context.Background()

// List firewall rules for a zone
rules, err := manager.ListFirewallRules(ctx, "zone-id-123")
if err != nil {
    panic(err)
}

for _, rule := range rules {
    fmt.Printf("Firewall Rule: %s (Description: %s)\n", rule.ID, rule.Description)
    fmt.Printf("  Expression: %s, Action: %s, Enabled: %v\n", #rule.Expression, rule.Action, rule.Enabled)
}
```

### Analytics Operations

```go
ctx := context.Background()

// Get analytics for the last 24 hours
since := time.Now().Add(-24 * time.Hour)
until := time.Now()

analytics, err := manager.GetAnalytics(ctx, "zone-id-123", since, until)
if err != nil {
    panic(err)
}

fmt.Printf("Analytics from %v to %v:\n", analytics.Since, analytics.Until)
fmt.Printf("  Requests: %d\n", analytics.Requests)
fmt.Printf("  Bandwidth: %d bytes\n", analytics.Bandwidth)
fmt.Printf("  Page Views: %d\n", analytics.PageViews)
fmt.Printf("  Unique Visitors: %d\n", analytics.Uniques)
fmt.Printf("  Threats: %d\n", analytics.Threats)
fmt.Printf("  Cached Requests: %d\n", analytics.CachedRequests)
fmt.Printf("  Cached Bytes: %d\n", analytics.CachedBytes)
fmt.Printf("  SSL Requests: %d\n", analytics.SSLRequests)
```

### Worker Operations

```go
ctx := context.Background()

// List workers for account
workers, err := manager.ListWorkers(ctx)
if err != nil {
    panic(err)
}

for _, worker := range workers {
    fmt.Printf("Worker: %s (Script: %s, Size: %d bytes)\n", #worker.ID, worker.ScriptName, worker.Size)
    fmt.Printf("  Created: %v, Modified: %v\n", worker.CreatedOn, worker.ModifiedOn)
}
```

### Async Operations

```go
ctx := context.Background()

// Async list zones
result := manager.ListZonesAsync(ctx)
select {
case zones := <-result.Ch:#    fmt.Printf("Found %d zones\n", len(zones))
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async list DNS records
result = manager.ListDNSRecordsAsync(ctx, "zone-id-123")
select {
case records := <-result.Ch:#    fmt.Printf("Found %d DNS records\n", len(records))
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async create DNS record
newRecord := cloudflare.CloudflareDNSRecord{
    Type: "CNAME",
    Name: "www.example.com",
    Content: "example.com",
    Proxied: true,
}

result = manager.CreateDNSRecordAsync(ctx, "zone-id-123", newRecord)
select {
case record := <-result.Ch:#    fmt.Printf("Created: %s\n", record.ID)
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async update DNS record
updatedRecord := cloudflare.CloudflareDNSRecord{Type: "A", Name: "example.com", Content: "192.0.2.3"}
result = manager.UpdateDNSRecordAsync(ctx, "zone-id-123", "record-id-456", updatedRecord)
select {
case record := <-result.Ch:#    fmt.Printf("Updated: %s\n", record.ID)
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async delete DNS record
result = manager.DeleteDNSRecordAsync(ctx, "zone-id-123", "record-id-456")
select {
case <-result.Ch:#    fmt.Println("Deleted")
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async purge cache
purgeReq := cloudflare.CloudflareCachePurgeRequest{PurgeAll: true}
result = manager.PurgeCacheAsync(ctx, "zone-id-123", purgeReq)
select {
case <-result.Ch:#    fmt.Println("Cache purged")
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async get analytics
since := time.Now().Add(-1 * time.Hour)
until := time.Now()
result = manager.GetAnalyticsAsync(ctx, "zone-id-123", since, until)
select {
case analytics := <-result.Ch:#    fmt.Printf("Requests: %d\n", analytics.Requests)
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}
```

### Get Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Email: %s\n", status["email"])

if status["connected"] == false {
    if errMsg, ok := status["error"].(string); ok {
        fmt.Printf("Error: %s\n", errMsg)
    }
}
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/hashicorp/go-retryablehttp` | HTTP client with retry logic for API resilience |
| `github.com/spf13/viper` | Configuration management for API credentials |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `context`, `encoding/json`, `io`, `net/http`, `net/url`, `time`, `fmt` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: `testConnection()` validates connectivity via `GET /user`
- **API errors**: HTTP status codes are checked and errors returned with response body
- **Context cancellation**: All operations respect `context.Context` for timeouts
- **Nil checks**: Public methods handle nil receiver gracefully in `GetStatus()`

Example error handling:

```go
zones, err := manager.ListZones(ctx)
if err != nil {
    if strings.Contains(err.Error(), "401") || strings.Contains(err.Error(), "Unauthorized") {
        return fmt.Errorf("authentication failed: %w", err)
    }
    if strings.Contains(err.Error(), "403") || strings.Contains(err.Error(), "Forbidden") {
        return fmt.Errorf("permission denied: %w", err)
    }
    return fmt.Errorf("failed to list zones: %w", err)
}
```

## Common Pitfalls

### 1. API Credentials Not Configured

**Problem**: `401 Unauthorized` or `403 Forbidden` errors

**Solution**: 
- Configure API key: `viper.Set("cloudflare.api_key", "your-api-key")`
- Configure email: `viper.Set("cloudflare.email", "your-email@example.com")`
- Verify API key has necessary permissions

### 2. Wrong Zone ID

**Problem**: `1011 Invalid zone ID` error

**Solution**:
- Verify zone ID format (hex string, e.g., "023e105f4ecef8ad9ca31a8372d0c353")
- Use `ListZones()` to get correct zone ID
- Check zone exists in your account

### 3. Rate Limiting

**Problem**: `429 Too Many Requests` error

**Solution**:
- Implement backoff (retry logic already built-in)
- Reduce request frequency
- Check Cloudflare rate limits (1200 requests per 5 minutes)

### 4. Context Timeout

**Problem**: Operations timing out

**Solution**:
```go
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
// Operations will use this timeout (client has 30s default)
```

### 5. DNS Record Validation

**Problem**: `1004 DNS Validation Error`

**Solution**:
- Ensure DNS record content is valid for the type (e.g., valid IP for A record)
- Check record name is within the zone
- Verify proxied setting is allowed for the record type

### 6. Cache Purge Limitations

**Problem**: `1413 Cache Purge Too Many Files`

**Solution**:
- Limit files to purge (max 30 per request)
- Use tags or hosts for bulk purge
- For full purge, use `PurgeAllCache()`

## Advanced Usage

### Zone Monitoring

```go
func monitorZones(manager *cloudflare.CloudflareManager) {
    ticker := time.NewTicker(5 * time.Minute)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:#            ctx := context.Background()
            zones, err := manager.ListZones(ctx)
            if err != nil {
                fmt.Printf("Error: %v\n", err)
                continue
            }
            
            fmt.Printf("\n=== Cloudflare Zones (%d) ===\n", len(zones))
            for _, zone := range zones {
                fmt.Printf("  %s: %s\n", zone.ID, zone.Name)
            }
        }
    }
}
```

### Analytics Dashboard

```go
func generateAnalyticsReport(manager *cloudflare.CloudflareManager, zoneID string) {
    ctx := context.Background()
    
    // Get analytics for last 7 days
    since := time.Now().Add(-7 * 24 * time.Hour)
    until := time.Now()
    
    analytics, err := manager.GetAnalytics(ctx, zoneID, since, until)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    fmt.Printf("=== Analytics Report for Zone %s ===\n", zoneID)
    fmt.Printf("Period: %v to %v\n", analytics.Since, analytics.Until)
    fmt.Printf("Total Requests: %d\n", analytics.Requests)
    fmt.Printf("Total Bandwidth: %.2f MB\n", float64(analytics.Bandwidth)/(1024*1024))
    fmt.Printf("Page Views: %d\n", analytics.PageViews)
    fmt.Printf("Unique Visitors: %d\n", analytics.Uniques)
    fmt.Printf("Threats Blocked: %d\n", analytics.Threats)
    fmt.Printf("Cached Requests: %d (%.1f%%)\n", #analytics.CachedRequests, float64(analytics.CachedRequests)/float64(analytics.Requests)*100)
    fmt.Printf("Cached Bytes: %.2f MB\n", float64(analytics.CachedBytes)/(1024*1024))
    fmt.Printf("SSL Requests: %d (%.1f%%)\n", analytics.SSLRequests, float64(analytics.SSLRequests)/float64(analytics.Requests)*100)
}
```

### Bulk DNS Management

```go
func bulkCreateDNSRecords(manager *cloudflare.CloudflareManager, zoneID string, records []cloudflare.CloudflareDNSRecord) error {
    ctx := context.Background()
    
    for _, record := range records {
        _, err := manager.CreateDNSRecord(ctx, zoneID, record)
        if err != nil {
            return fmt.Errorf("failed to create record %s: %w", record.Name, err)
        }
        fmt.Printf("Created: %s\n", record.Name)
    }
    
    return nil
}
```

### Cache Management Strategy

```go
func smartCachePurge(manager *cloudflare.CloudflareManager, zoneID string) {
    ctx := context.Background()
    
    // Purge specific critical files
    purgeReq := cloudflare.CloudflareCachePurgeRequest{
        Files: []string{
            "https://example.com/css/main.css",
            "https://example.com/js/bundle.js",
        },
    }
    
    err := manager.PurgeCache(ctx, zoneID, purgeReq)
    if err != nil {
        fmt.Printf("Error purging files: %v\n", err)
    } else {
        fmt.Println("Critical files purged")
    }
    
    // Purge by tags for updated content
    purgeReq = cloudflare.CloudflareCachePurgeRequest{
        Tags: []string{"images-updated", "api-v2"},
    }
    
    err = manager.PurgeCache(ctx, zoneID, purgeReq)
    if err != nil {
        fmt.Printf("Error purging tags: %v\n", err)
    } else {
        fmt.Println("Tagged content purged")
    }
}
```

### Worker Deployment Check

```go
func checkWorkerDeployments(manager *cloudflare.CloudflareManager) {
    ctx := context.Background()
    
    workers, err := manager.ListWorkers(ctx)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    fmt.Println("=== Worker Deployments ===")
    for _, worker := range workers {
        fmt.Printf("Script: %s (Size: %d bytes)\n", worker.ScriptName, worker.Size)
        fmt.Printf("  Created: %v, Modified: %v\n", worker.CreatedOn, worker.ModifiedOn)
        
        if usage, ok := worker.Usage.(map[string]interface{}); ok {
            fmt.Printf("  Usage: %v\n", usage)
        }
    }
}
```

## Internal Algorithms

### API Request Flow

```
Generic API Request:
    │
    ├── Create HTTP request with method, URL, optional body
    ├── Set headers:
    │   ├── X-Auth-Key: {api_key}
    │   ├── X-Auth-Email: {email}
    │   └── Content-Type: application/json (for POST/PUT)
    ├── Execute via retryablehttp.Client
    ├── Check status code == 200 (http.StatusOK)
    ├── Read body
    └── JSON decode to target struct
```

### Analytics Aggregation

```
GetAnalytics():
    │
    ├── Build query params with since and until (RFC3339)
    ├── GET /zones/{zoneID}/analytics/dashboard?since=X&until=Y
    ├── Decode response with timeseries array
    ├── For each timeseries entry, sum:
    │   ├── Requests
    │   ├── Bandwidth
    │   ├── PageViews
    │   ├── Uniques
    │   ├── Threats
    │   ├── CachedRequests (from cache_metrics)
    │   ├── CachedBytes (from cache_metrics)
    │   └── SSLRequests (from ssl_metrics)
    └── Return aggregated CloudflareAnalytics
```

### Retry Logic

Using `go-retryablehttp`:
- Maximum retries: 3
- Minimum wait: 1 second
- Maximum wait: 10 seconds
- HTTP client timeout: 30 seconds

Retries are automatic for transient network errors.

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.