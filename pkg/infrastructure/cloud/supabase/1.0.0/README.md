# Supabase Manager

## Overview

The `SupabaseManager` is a comprehensive Go library for managing Supabase resources via the Supabase API. It provides authentication (sign in, sign up), database operations (select, insert, update, delete, count), and storage operations (upload, download, delete) with async support.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Authentication**: Sign in with email/password, sign up new users, get user info
- **Database Operations**: Select, insert, update, delete records via Supabase REST API (PostgREST)
- **Storage Operations**: Upload, download, delete files in Supabase Storage
- **Async Operations**: Async execution for auth, database, and storage operations via worker pool
- **Authentication**: Uses Supabase API key via `apikey` and `Authorization: Bearer` headers
- **Retry Logic**: Automatic retries with exponential backoff via `go-retryablehttp`
- **Worker Pool**: Async job execution support (4 workers)

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
    
    // Create Supabase manager (configuration via viper)
    manager, err := supabase.NewSupabaseManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Sign in a user
    session, err := manager.SignInWithEmail(ctx, "user@example.com", "password123")
    if err != nil {
        panic(err)
    }
    fmt.Printf("Signed in! Access token: %s\n", session.AccessToken)
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `SupabaseManager` | Main manager with HTTP client and worker pool |
| `SupabaseUser` | Supabase user with ID, email, metadata, timestamps |
| `SupabaseSession` | Authenticated session with access token, refresh token, user |
| `SupabaseStorageObject` | Storage object with name, bucket, owner, timestamps |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   SupabaseManager                         │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │ retryablehttp  │  → Retry logic (max 3, 1-10s wait)│
│  │ Client         │  → HTTP timeout: 60s                   │
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
│  BaseURL: Supabase project URL                           │
│  AuthURL: projectURL + "/auth/v1"                      │
│  DatabaseURL: projectURL + "/rest/v1"                   │
│  StorageURL: projectURL + "/storage/v1"                 │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewSupabaseManager()
    │
    ├── Check viper config: "supabase.enabled"
    ├── Get project URL: "supabase.project_url"
    ├── Get API key: "supabase.api_key"
    ├── Create retryablehttp.Client (max 3 retries, 1-10s wait, 60s timeout)
    ├── Set logger adapter
    ├── Build manager with BaseURL, AuthURL, DatabaseURL, StorageURL
    ├── Test connection: testConnection() → GET /rest/v1/
    ├── Create WorkerPool(4)
    ├── Start worker pool
    └── Return SupabaseManager
```

### 2. Sign In Flow

```
SignInWithEmail(ctx, email, password)
    │
    ├── Build JSON payload with email and password
    ├── Create POST request to AuthURL + "/token?grant_type=password"
    ├── Set Content-Type: application/json
    ├── Set apikey header
    ├── Execute request
    ├── Check status code (200 OK)
    ├── JSON decode to SupabaseSession
    └── Return session
```

### 3. Database Select Flow

```
Select(ctx, table, columns, filters)
    │
    ├── Build URL: DatabaseURL + "/" + table
    ├── Set headers: apikey, Authorization, Accept
    ├── Add query parameters: select=columns, plus filters
    ├── Create GET request
    ├── Execute request
    ├── Check status code (200 OK)
    ├── JSON decode to []map[string]interface{}
    └── Return results
```

### 4. Database Insert Flow

```
Insert(ctx, table, data)
    │
    ├── JSON marshal data
    ├── Create POST request to DatabaseURL + "/" + table
    ├── Set Content-Type: application/json
    ├── Set apikey, Authorization, Prefer: return=representation
    ├── Execute request
    ├── Check status code (201 Created)
    ├── JSON decode to []map[string]interface{}
    └── Return inserted records
```

### 5. Storage Upload Flow

```
UploadFile(ctx, bucket, path, reader, contentType)
    │
    ├── Create POST request to StorageURL + "/object/" + bucket + "/" + path
    ├── Set Content-Type, apikey, Authorization
    ├── Execute request with reader body
    ├── Check status code (200 OK)
    └── Return nil or error
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `supabase.enabled` | bool | false | Enable/disable Supabase manager |
| `supabase.project_url` | string | "" | Supabase project URL (e.g., `https://xyz.supabase.co`) |
| `supabase.api_key` | string | "" | Supabase API key (anon or service role) |

### Environment Variables

The Supabase manager does not directly use environment variables. Configuration is read through Viper, which can load from:
- Configuration files (JSON, YAML, TOML)
- Environment variables (if Viper is configured for it)

You can set Supabase credentials via:
```go
viper.Set("supabase.project_url", "https://xyz.supabase.co")
viper.Set("supabase.api_key", "your-supabase-api-key")
```

## Usage Examples

### Authentication Operations

```go
ctx := context.Background()

// Sign in with email and password
session, err := manager.SignInWithEmail(ctx, "user@example.com", "password123")
if err != nil {
    panic(err)
}

fmt.Printf("Access Token: %s\n", session.AccessToken)
fmt.Printf("Token Type: %s\n", session.TokenType)
fmt.Printf("Expires In: %d seconds\n", session.ExpiresIn)
fmt.Printf("User ID: %s\n", session.User.ID)
fmt.Printf("User Email: %s\n", session.User.Email)

// Sign up a new user
newUser, err := manager.SignUp(ctx, "newuser@example.com", "newpassword")
if err != nil {
    panic(err)
}

fmt.Printf("Signed up user: %s (ID: %s)\n", newUser.Email, newUser.ID)

// Get user by ID (requires service role key)
user, err := manager.GetUser(ctx, "user-id-here")
if err != nil {
    panic(err)
}

fmt.Printf("User: %s (Email: %s)\n", user.ID, user.Email)
fmt.Printf("  Created: %v, Last Sign In: %v\n", user.CreatedAt, user.LastSignInAt)
fmt.Printf("  App Metadata: %v\n", user.AppMetadata)
fmt.Printf("  User Metadata: %v\n", user.UserMetadata)
```

### Database Operations

```go
ctx := context.Background()

// Select records from a table
results, err := manager.Select(ctx, "users", "id,email,name", map[string]interface{}{
    "role": "eq.admin",
})
if err != nil {
    panic(err)
}

for _, row := range results {
    fmt.Printf("User: %v (Email: %v)\n", row["name"], row["email"])
}

// Insert a new record
newRecord := map[string]interface{}{
    "name":  "John Doe",
    "email": "john@example.com",
    "role":  "user",
}

inserted, err := manager.Insert(ctx, "users", newRecord)
if err != nil {
    panic(err)
}

for _, rec := range inserted {
    fmt.Printf("Inserted: %v\n", rec["id"])
}

// Update records
updateData := map[string]interface{}{
    "name": "Jane Doe",
}

updated, err := manager.Update(ctx, "users", updateData, map[string]interface{}{
    "id": "eq.user-id-here",
})
if err != nil {
    panic(err)
}

fmt.Printf("Updated %d records\n", len(updated))

// Delete records
err = manager.Delete(ctx, "users", map[string]interface{}{
    "id": "eq.user-id-here",
})
if err != nil {
    panic(err)
}

fmt.Println("Deleted record")

// Count records
count, err := manager.Count(ctx, "users", map[string]interface{}{
    "role": "eq.admin",
})
if err != nil {
    panic(err)
}

fmt.Printf("Admin count: %d\n", count)
```

### Storage Operations

```go
ctx := context.Background()

// Upload a file
content := strings.NewReader("Hello, Supabase Storage!")
err := manager.UploadFile(ctx, "my-bucket", "uploads/hello.txt", content, "text/plain")
if err != nil {
    panic(err)
}

fmt.Println("File uploaded")

// Download a file
reader, err := manager.DownloadFile(ctx, "my-bucket", "uploads/hello.txt")
if err != nil {
    panic(err)
}
defer reader.Close()

data, _ := io.ReadAll(reader)
fmt.Printf("Downloaded: %s\n", string(data))

// Get public URL for a file
url := manager.GetPublicURL("my-bucket", "uploads/hello.txt")
fmt.Printf("Public URL: %s\n", url)

// Delete a file
err = manager.DeleteFile(ctx, "my-bucket", "uploads/hello.txt")
if err != nil {
    panic(err)
}

fmt.Println("File deleted")
```

### Async Operations

```go
ctx := context.Background()

// Async sign in
result := manager.SignInWithEmailAsync(ctx, "user@example.com", "password123")
select {
case session := <-result.Ch:#    fmt.Printf("Signed in: %s\n", session.AccessToken)
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async sign up
result = manager.SignUpAsync(ctx, "new@example.com", "password")
select {
case user := <-result.Ch:#    fmt.Printf("Signed up: %s\n", user.Email)
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async select
result = manager.SelectAsync(ctx, "users", "id,email", map[string]interface{}{})
select {
case rows := <-result.Ch:#    fmt.Printf("Found %d rows\n", len(rows))
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async insert
result = manager.InsertAsync(ctx, "users", map[string]interface{}{"name": "Async User"})
select {
case rows := <-result.Ch:#    fmt.Printf("Inserted: %v\n", rows[0]["id"])
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async update
result = manager.UpdateAsync(ctx, "users", map[string]interface{}{"name": "Updated"}, map[string]interface{}{"id": "eq.123"})
select {
case rows := <-result.Ch:#    fmt.Printf("Updated %d rows\n", len(rows))
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async delete
result = manager.DeleteAsync(ctx, "users", map[string]interface{}{"id": "eq.123"})
select {
case <-result.Ch:#    fmt.Println("Deleted")
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}
```

### Get Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Base URL: %s\n", status["base_url"])

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
| Standard library | `context`, `encoding/json`, `fmt`, `io`, `net/http`, `time`, `bytes` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: `testConnection()` validates connectivity via `GET /rest/v1/`
- **API errors**: HTTP status codes are checked and errors returned with response body
- **Context cancellation**: All operations respect `context.Context` for timeouts
- **Nil checks**: Public methods handle nil receiver gracefully in `GetStatus()`

Example error handling:

```go
session, err := manager.SignInWithEmail(ctx, email, password)
if err != nil {
    if strings.Contains(err.Error(), "400") || strings.Contains(err.Error(), "invalid_grant") {
        return fmt.Errorf("invalid credentials: %w", err)
    }
    if strings.Contains(err.Error(), "429") {
        return fmt.Errorf("rate limited: %w", err)
    }
    return fmt.Errorf("sign in failed: %w", err)
}
```

## Common Pitfalls

### 1. API Key Not Configured

**Problem**: `401 Unauthorized` or `403 Forbidden` errors

**Solution**: 
- Configure project URL: `viper.Set("supabase.project_url", "https://xyz.supabase.co")`
- Configure API key: `viper.Set("supabase.api_key", "your-anon-key")`
- Use service role key for admin operations (sign up, get user)

### 2. Wrong Table or Column Names

**Problem**: `404 Not Found` or `400 Bad Request` errors

**Solution**:
- Verify table names exist in your Supabase database
- Check column names match the actual schema
- Use proper quoting for column names with special characters

### 3. Row Level Security (RLS)

**Problem**: `403 Forbidden` when accessing tables

**Solution**:
- Ensure RLS policies allow the operation
- Use service role key for admin access
- Or sign in as user and use their JWT token

### 4. Context Timeout

**Problem**: Operations timing out

**Solution**:
```go
ctx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
defer cancel()
// Operations will use this timeout (client has 60s default)
```

### 5. Storage Bucket Not Found

**Problem**: `404 Not Found` when accessing storage

**Solution**:
- Verify bucket exists in Supabase Storage
- Check bucket name spelling
- Ensure storage policies allow access

### 6. Database Filter Syntax

**Problem**: Filters not working as expected

**Solution**:
- Use PostgREST filter syntax: `eq`, `neq`, `gt`, `lt`, `like`, etc.
- Example: `map[string]interface{}{"role": "eq.admin"}`
- For multiple filters, add multiple keys: they are ANDed

## Advanced Usage

### User Authentication Flow

```go
func authenticateUser(manager *supabase.SupabaseManager, email, password string) (*supabase.SupabaseSession, error) {
    ctx := context.Background()
    
    // Sign in
    session, err := manager.SignInWithEmail(ctx, email, password)
    if err != nil {
        return nil, fmt.Errorf("authentication failed: %w", err)
    }
    
    // Get additional user details if needed
    user, err := manager.GetUser(ctx, session.User.ID)
    if err != nil {
        // Non-critical, just log
        fmt.Printf("Warning: could not fetch user details: %v\n", err)
    } else {
        fmt.Printf("User: %s, Last Sign In: %v\n", user.Email, user.LastSignInAt)
    }
    
    return session, nil
}
```

### Database Query with Complex Filters

```go
func queryUsers(manager *supabase.SupabaseManager) {
    ctx := context.Background()
    
    // Select with multiple filters and specific columns
    filters := map[string]interface{}{
        "role":      "eq.user",
        "created_at": "gt.2024-01-01",
    }
    
    results, err := manager.Select(ctx, "users", "id,email,name,created_at", filters)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    fmt.Printf("Found %d users:\n", len(results))
    for _, row := range results {
        fmt.Printf("  %s: %s (Created: %v)\n", row["id"], row["name"], row["created_at"])
    }
}
```

### Storage File Management

```go
func manageStorageFiles(manager *supabase.SupabaseManager, bucket string) {
    ctx := context.Background()
    
    // Upload multiple files
    files := []struct { Path, Content string }{
        {"data/file1.txt", "Content 1"},
        {"data/file2.txt", "Content 2"},
    }
    
    for _, f := range files {
        err := manager.UploadFile(ctx, bucket, f.Path, strings.NewReader(f.Content), "text/plain")
        if err != nil {
            fmt.Printf("Upload failed for %s: %v\n", f.Path, err)
            continue
        }
        fmt.Printf("Uploaded: %s\n", f.Path)
    }
    
    // Get public URLs
    for _, f := range files {
        url := manager.GetPublicURL(bucket, f.Path)
        fmt.Printf("Public URL: %s\n", url)
    }
}
```

### Batch Database Operations

```go
func batchInsertUsers(manager *supabase.SupabaseManager, users []map[string]interface{}) error {
    ctx := context.Background()
    
    for i, user := range users {
        _, err := manager.Insert(ctx, "users", user)
        if err != nil {
            return fmt.Errorf("failed to insert user %d: %w", i, err)
        }
        fmt.Printf("Inserted user %d\n", i)
    }
    
    return nil
}
```

## Internal Algorithms

### API Request Flow

```
Generic API Request:
    │
    ├── Create HTTP request with method, URL, optional body
    ├── Set headers:
    │   ├── apikey: {api_key}
    │   ├── Authorization: Bearer {api_key}
    │   └── Content-Type: application/json (for POST/PATCH)
    ├── Execute via retryablehttp.Client
    ├── Check status code == 200 (http.StatusOK)
    ├── Read body
    └── JSON decode to target struct
```

### Database Query Building

```
Select():
    │
    ├── Build URL: DatabaseURL + "/" + table
    ├── Add query parameter: select=columns
    ├── Add filter parameters (key=value)
    └── GET request with headers
```

### Storage URL Building

```
UploadFile():
    ├── URL: StorageURL + "/object/" + bucket + "/" + path
    └── POST with file content
    
DownloadFile():
    ├── URL: StorageURL + "/object/" + bucket + "/" + path
    └── GET request
    
DeleteFile():
    ├── URL: StorageURL + "/object/" + bucket + "/" + path
    └── DELETE request
```

### Retry Logic

Using `go-retryablehttp`:
- Maximum retries: 3
- Minimum wait: 1 second
- Maximum wait: 10 seconds
- HTTP client timeout: 60 seconds

Retries are automatic for transient network errors.

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.