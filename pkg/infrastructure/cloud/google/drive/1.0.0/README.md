# Google Drive Manager

## Overview

The `GoogleDriveManager` is a comprehensive Go library for managing Google Drive resources via the Google Drive API (v3). It provides complete Google Drive resource management including files, folders, permissions, uploads, downloads, exports, search, and quota information with async support.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **File Management**: List, get, create, update, delete (trash or permanent), copy, move files
- **Folder Management**: Create folders and organize files
- **Upload/Download**: Multipart uploads, downloads, and exports (Google Workspace documents)
- **Permissions**: List, create, and delete permissions on files
- **Search**: Search files with query strings and pagination
- **Quota Information**: Get storage quota and usage information
- **About Info**: Retrieve user and Drive information
- **Batch Operations**: Batch upload and delete multiple files
- **Async Operations**: Async execution for all operations via worker pool
- **Authentication**: Uses OAuth2 access token via `Authorization: Bearer` header
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
    
    // Create Google Drive manager (configuration via viper)
    manager, err := drive.NewGoogleDriveManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // List files in Drive
    files, err := manager.ListFiles(ctx, drive.DriveSearchQuery{PageSize: 10})
    if err != nil {
        panic(err)
    }
    
    for _, file := range files.Files {
        fmt.Printf("File: %s (ID: %s, Type: %s)\n", file.Name, file.ID, file.MIMEType)
    }
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `GoogleDriveManager` | Main manager with HTTP client and worker pool |
| `DriveFile` | Drive file/folder with metadata (ID, name, MIME type, size, etc.) |
| `DriveUser` | Drive user with display name, email, photo link |
| `DrivePermission` | Permission with type, role, email address |
| `DriveFileList` | List response with files and pagination token |
| `DriveUploadRequest` | Upload request with metadata (name, MIME type, parents) |
| `DriveSearchQuery` | Search parameters (query, page size, order by, fields) |
| `DriveBatchRequest` | Single request in a batch operation |
| `DriveBatchResponse` | Single response in a batch operation |
| `DriveError` | Drive API error with code, message, status |
| `DriveQuotaInfo` | Storage quota information (limit, usage, trash) |
| `DriveAbout` | About/user info with user and quota |
| `DriveAboutUser` | User info in about response |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   GoogleDriveManager                       │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │ retryablehttp  │  → Retry logic (max 3, 1-10s wait)│
│  │ Client         │  → HTTP timeout: 120s                  │
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
│  BaseURL: https://www.googleapis.com/drive/v3         │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewGoogleDriveManager()
    │
    ├── Check viper config: "google_drive.enabled"
    ├── Get access token: "google_drive.access_token"
    ├── Create retryablehttp.Client (max 3 retries, 1-10s wait, 120s timeout)
    ├── Set logger adapter
    ├── Build manager with BaseURL "https://www.googleapis.com/drive/v3"
    ├── Test connection: testConnection() → GET /about?fields=user(displayName)
    ├── Create WorkerPool(4)
    ├── Start worker pool
    └── Return GoogleDriveManager
```

### 2. API Request Flow

```
Generic API Request (GET/POST/PATCH/DELETE)
    │
    ├── Create request with context
    ├── Set Authorization: Bearer {access_token}
    ├── Set Content-Type: application/json (for POST/PATCH)
    ├── Execute request via retryablehttp.Client
    ├── Read response body
    ├── Check status code (200 OK = success)
    └── JSON decode into result struct
```

### 3. List Files Flow

```
ListFiles(ctx, query)
    │
    ├── Build query parameters (q, pageSize, pageToken, orderBy, fields, spaces)
    ├── Create GET request to /files?params
    ├── Set Authorization header
    ├── Execute request
    ├── Check status code
    ├── JSON decode to DriveFileList
    └── Return DriveFileList with files and nextPageToken
```

### 4. Upload File Flow

```
UploadFile(ctx, metadata, content)
    │
    ├── Marshal metadata to JSON
    ├── Build multipart body with metadata and media parts
    ├── Create POST request to /upload/drive/v3/files?uploadType=multipart
    ├── Set Content-Type: multipart/form-data
    ├── Set Authorization header
    ├── Execute request
    ├── Check status code
    ├── JSON decode to DriveFile
    └── Return created DriveFile
```

### 5. Download File Flow

```
DownloadFile(ctx, fileID)
    │
    ├── Create GET request to /files/{fileID}?alt=media
    ├── Set Authorization header
    ├── Execute request
    ├── Check status code
    └── Return io.ReadCloser (response body) and content type
```

### 6. Search Files Flow

```
SearchFiles(ctx, query, pageSize)
    │
    ├── Call ListFiles with DriveSearchQuery{Query: query, PageSize: pageSize}
    └── Return DriveFileList
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `google_drive.enabled` | bool | false | Enable/disable Google Drive manager |
| `google_drive.access_token` | string | "" | OAuth2 access token |

### Environment Variables

The Google Drive manager does not directly use environment variables. Configuration is read through Viper, which can load from:
- Configuration files (JSON, YAML, TOML)
- Environment variables (if Viper is configured for it)

You can set the access token via:
```go
viper.Set("google_drive.access_token", "ya29.a0...")
```

## Usage Examples

### File Operations

```go
ctx := context.Background()

// List files (first page)
files, err := manager.ListFiles(ctx, drive.DriveSearchQuery{PageSize: 10})
if err != nil {
    panic(err)
}

for _, file := range files.Files {
    fmt.Printf("File: %s (ID: %s)\n", file.Name, file.ID)
    fmt.Printf("  Type: %s, Size: %d bytes\n", file.MIMEType, file.Size)
    fmt.Printf("  Created: %v, Modified: %v\n", file.CreatedTime, file.ModifiedTime)
    fmt.Printf("  Trashed: %v, Shared: %v\n", file.Trashed, file.Shared)
}

if files.NextPageToken != "" {
    fmt.Println("More files available...")
}

// Get specific file metadata
file, err := manager.GetFile(ctx, "file-id-123")
if err != nil {
    panic(err)
}
fmt.Printf("File: %s (Description: %s)\n", file.Name, file.Description)
fmt.Printf("  Owners: %v\n", file.Owners)
fmt.Printf("  Web View Link: %s\n", file.WebViewLink)

// Create a folder
folder, err := manager.CreateFolder(ctx, "My Folder", []string{"parent-folder-id"})
if err != nil {
    panic(err)
}
fmt.Printf("Created folder: %s (ID: %s)\n", folder.Name, folder.ID)

// Upload a file
content := strings.NewReader("Hello, Google Drive!")
metadata := drive.DriveUploadRequest{
    Name:    "hello.txt",
    MIMEType: "text/plain",
    ParentIDs: []string{"folder-id-456"},
}

uploaded, err := manager.UploadFile(ctx, metadata, content)
if err != nil {
    panic(err)
}
fmt.Printf("Uploaded: %s (ID: %s)\n", uploaded.Name, uploaded.ID)

// Update file metadata
metadata = drive.DriveUploadRequest{
    Name:        "updated.txt",
    Description: "Updated file",
}

updated, err := manager.UpdateFile(ctx, "file-id-123", metadata)
if err != nil {
    panic(err)
}
fmt.Printf("Updated: %s\n", updated.Name)

// Move file to another folder
moved, err := manager.MoveFile(ctx, "file-id-123", []string{"new-parent-id"}, []string{"old-parent-id"})
if err != nil {
    panic(err)
}
fmt.Printf("Moved: %s\n", moved.Name)

// Copy file
copied, err := manager.CopyFile(ctx, "file-id-123", "Copy of file", nil)
if err != nil {
    panic(err)
}
fmt.Printf("Copied: %s (New ID: %s)\n", copied.Name, copied.ID)

// Download file
reader, contentType, err := manager.DownloadFile(ctx, "file-id-123")
if err != nil {
    panic(err)
}
defer reader.Close()

data, _ := io.ReadAll(reader)
fmt.Printf("Downloaded: %s (Content-Type: %s)\n", string(data), contentType)

// Export Google Doc as PDF
pdfReader, err := manager.ExportFile(ctx, "google-doc-id", "application/pdf")
if err != nil {
    panic(err)
}
defer pdfReader.Close()

// Delete file (move to trash)
err = manager.DeleteFile(ctx, "file-id-123", false)
if err != nil {
    panic(err)
}
fmt.Println("File moved to trash")

// Permanently delete file
err = manager.DeleteFile(ctx, "file-id-123", true)
if err != nil {
    panic(err)
}
fmt.Println("File permanently deleted")
```

### Search Operations

```go
ctx := context.Background()

// Search for files with query
files, err := manager.SearchFiles(ctx, "name contains 'report' and trashed = false", 20)
if err != nil {
    panic(err)
}

fmt.Printf("Found %d files:\n", len(files.Files))
for _, file := range files.Files {
    fmt.Printf("  %s (ID: %s)\n", file.Name, file.ID)
}

// Search with more options
files, err = manager.ListFiles(ctx, drive.DriveSearchQuery{
    Query:    "mimeType = 'application/pdf'",
    PageSize: 50,
    OrderBy:  "modifiedTime desc",
    Fields:   "files(id,name,modifiedTime,size)",
    Spaces:   "drive",
})
if err != nil {
    panic(err)
}
```

### Permission Operations

```go
ctx := context.Background()

// List permissions for a file
permissions, err := manager.ListPermissions(ctx, "file-id-123")
if err != nil {
    panic(err)
}

for _, perm := range permissions {
    fmt.Printf("Permission: %s (Type: %s, Role: %s)\n", perm.ID, perm.Type, perm.Role)
    fmt.Printf("  Email: %s, Domain: %s\n", perm.EmailAddress, perm.Domain)
}

// Create permission (share with user)
newPerm := drive.DrivePermission{
    Type:       "user",
    Role:       "writer",
    EmailAddress: "user@example.com",
}

created, err := manager.CreatePermission(ctx, "file-id-123", newPerm)
if err != nil {
    panic(err)
}
fmt.Printf("Created permission: %s\n", created.ID)

// Delete permission
err = manager.DeletePermission(ctx, "file-id-123", "permission-id-456")
if err != nil {
    panic(err)
}
fmt.Println("Permission deleted")
```

### About & Quota Operations

```go
ctx := context.Background()

// Get about info
about, err := manager.GetAbout(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("User: %s (Email: %s)\n", about.User.DisplayName, about.User.EmailAddress)
fmt.Printf("Max Upload Size: %d bytes\n", about.MaxUploadSize)
fmt.Printf("Storage Quota:\n")
fmt.Printf("  Limit: %d bytes\n", about.StorageQuota.Limit)
fmt.Printf("  Usage: %d bytes\n", about.StorageQuota.Usage)
fmt.Printf("  Usage in Trash: %d bytes\n", about.StorageQuota.UsageInDriveTrash)

// Get quota info only
quota, err := manager.GetQuota(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("Quota Limit: %.2f GB\n", float64(quota.Limit)/(1024*1024*1024))
fmt.Printf("Quota Used: %.2f GB (%.1f%%)\n", #float64(quota.Usage)/(1024*1024*1024), float64(quota.Usage)/float64(quota.Limit)*100)
fmt.Printf("Remaining: %.2f GB\n", float64(quota.Limit-quota.Usage)/(1024*1024*1024))
```

### Async Operations

```go
ctx := context.Background()

// Async list files
result := manager.ListFilesAsync(ctx, drive.DriveSearchQuery{PageSize: 10})
select {
case fileList := <-result.Ch:#    fmt.Printf("Found %d files\n", len(fileList.Files))
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async get file
result = manager.GetFileAsync(ctx, "file-id-123")
select {
case file := <-result.Ch:#    fmt.Printf("File: %s\n", file.Name)
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async create folder
result = manager.CreateFolderAsync(ctx, "Async Folder", nil)
select {
case folder := <-result.Ch:#    fmt.Printf("Created: %s\n", folder.Name)
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async upload file
content := strings.NewReader("Async upload test")
metadata := drive.DriveUploadRequest{Name: "async-test.txt", MIMEType: "text/plain"}
result = manager.UploadFileAsync(ctx, metadata, content)
select {
case file := <-result.Ch:#    fmt.Printf("Uploaded: %s\n", file.Name)
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async update file
metadata = drive.DriveUploadRequest{Name: "updated-async.txt"}
result = manager.UpdateFileAsync(ctx, "file-id-123", metadata)
select {
case file := <-result.Ch:#    fmt.Printf("Updated: %s\n", file.Name)
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async delete file
result = manager.DeleteFileAsync(ctx, "file-id-123", false)
select {
case <-result.Ch:#    fmt.Println("Deleted")
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async copy file
result = manager.CopyFileAsync(ctx, "file-id-123", "Copy", nil)
select {
case file := <-result.Ch:#    fmt.Printf("Copied: %s\n", file.Name)
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async move file
result = manager.MoveFileAsync(ctx, "file-id-123", []string{"new-parent"}, []string{"old-parent"})
select {
case file := <-result.Ch:#    fmt.Printf("Moved: %s\n", file.Name)
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async list permissions
result = manager.ListPermissionsAsync(ctx, "file-id-123")
select {
case perms := <-result.Ch:#    fmt.Printf("Found %d permissions\n", len(perms))
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async create permission
newPerm := drive.DrivePermission{Type: "user", Role: "reader", EmailAddress: "test@example.com"}
result = manager.CreatePermissionAsync(ctx, "file-id-123", newPerm)
select {
case perm := <-result.Ch:#    fmt.Printf("Created permission: %s\n", perm.ID)
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async get about
result = manager.GetAboutAsync(ctx)
select {
case about := <-result.Ch:#    fmt.Printf("User: %s\n", about.User.DisplayName)
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async search files
result = manager.SearchFilesAsync(ctx, "name contains 'report'", 10)
select {
case files := <-result.Ch:#    fmt.Printf("Found %d files\n", len(files.Files))
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}
```

### Batch Operations

```go
ctx := context.Background()

// Batch upload multiple files
files := []struct {
    Metadata drive.DriveUploadRequest
    Content  io.Reader
}{
    {Metadata: drive.DriveUploadRequest{Name: "file1.txt", MIMEType: "text/plain"}, Content: strings.NewReader("Content 1")},
    {Metadata: drive.DriveUploadRequest{Name: "file2.txt", MIMEType: "text/plain"}, Content: strings.NewReader("Content 2")},
}

results, errors := manager.BatchUpload(ctx, files)
for i, file := range results {
    if errors[i] != nil {
        fmt.Printf("Upload %d failed: %v\n", i, errors[i])
    } else {
        fmt.Printf("Uploaded: %s\n", file.Name)
    }
}

// Batch delete multiple files
fileIDs := []string{"file-id-1", "file-id-2", "file-id-3"}
errors = manager.BatchDelete(ctx, fileIDs, false)
for i, err := range errors {
    if err != nil {
        fmt.Printf("Delete %s failed: %v\n", fileIDs[i], err)
    }
}
```

### Get Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("User: %s\n", status["user"])
fmt.Printf("Email: %s\n", status["email"])
fmt.Printf("Max Upload Size: %v\n", status["max_upload_size"])

if quotaLimit, ok := status["quota_limit"].(int64); ok {
    fmt.Printf("Quota Limit: %.2f GB\n", float64(quotaLimit)/(1024*1024*1024))
}
if quotaUsed, ok := status["quota_used"].(int64); ok {
    fmt.Printf("Quota Used: %.2f GB\n", float64(quotaUsed)/(1024*1024*1024))
}

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
| `github.com/spf13/viper` | Configuration management for access token |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `context`, `encoding/json`, `io`, `mime/multipart`, `net/http`, `net/url`, `strings`, `time`, `fmt` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: `testConnection()` validates connectivity via `GET /about`
- **API errors**: HTTP status codes are checked and errors returned with response body
- **Context cancellation**: All operations respect `context.Context` for timeouts
- **Nil checks**: Public methods handle nil receiver gracefully in `GetStatus()`

Example error handling:

```go
files, err := manager.ListFiles(ctx, drive.DriveSearchQuery{})
if err != nil {
    if strings.Contains(err.Error(), "401") || strings.Contains(err.Error(), "Unauthorized") {
        return fmt.Errorf("authentication failed: %w", err)
    }
    if strings.Contains(err.Error(), "403") || strings.Contains(err.Error(), "Forbidden") {
        return fmt.Errorf("permission denied: %w", err)
    }
    if strings.Contains(err.Error(), "404") || strings.Contains(err.Error(), "Not Found") {
        return fmt.Errorf("file not found: %w", err)
    }
    return fmt.Errorf("failed to list files: %w", err)
}
```

## Common Pitfalls

### 1. Access Token Not Configured

**Problem**: `401 Unauthorized` or `403 Forbidden` errors

**Solution**: 
- Configure access token: `viper.Set("google_drive.access_token", "ya29.a0...")`
- Obtain token via OAuth2 flow (see Google Drive API docs)
- Token must have appropriate scopes (e.g., `https://www.googleapis.com/auth/drive`)

### 2. Invalid File ID

**Problem**: `404 Not Found` error

**Solution**:
- Verify file ID format (e.g., "0B7lS1h3kFtXc3RkZTZSaU")
- Use `ListFiles()` or `SearchFiles()` to get correct file ID
- Check file exists and is accessible

### 3. Rate Limiting

**Problem**: `403 Rate Limit Exceeded` error

**Solution**:
- Implement backoff (retry logic already built-in)
- Reduce request frequency
- Check Google Drive API quotas

### 4. Context Timeout

**Problem**: Operations timing out

**Solution**:
```go
ctx, cancel := context.WithTimeout(context.Background(), 120*time.Second)
defer cancel()
// Operations will use this timeout (client has 120s default)
```

### 5. Large File Uploads

**Problem**: Upload failing for large files

**Solution**:
- Use multipart upload (already implemented)
- Check `maxUploadSize` from `GetAbout()`
- For files > 5MB, consider resumable upload (not implemented yet)

### 6. Permission Issues

**Problem**: `403` when trying to access/modify files

**Solution**:
- Verify access token has correct scopes
- Check file permissions (use `ListPermissions()`)
- Ensure token is not expired

### 7. Trashed Files

**Problem**: File not found but exists

**Solution**:
- Search with `trashed = false` in query
- Or include `trashed = true` to find trashed files
- Use `DeleteFile(ctx, fileID, false)` to move to trash, or `true` for permanent deletion

## Advanced Usage

### File Monitoring

```go
func monitorFiles(manager *drive.GoogleDriveManager) {
    ticker := time.NewTicker(5 * time.Minute)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:#            ctx := context.Background()
            files, err := manager.SearchFiles(ctx, "modifiedTime > '"+time.Now().Add(-5*time.Minute).Format(time.RFC3339)+"'", 10)
            if err != nil {
                fmt.Printf("Error: %v\n", err)
                continue
            }
            
            fmt.Printf("\n=== Recently Modified Files (%d) ===\n", len(files.Files))
            for _, file := range files.Files {
                fmt.Printf("  %s: %s (Modified: %v)\n", file.ID, file.Name, file.ModifiedTime)
            }
        }
    }
}
```

### Storage Analytics

```go
func analyzeStorage(manager *drive.GoogleDriveManager) {
    ctx := context.Background()
    
    quota, err := manager.GetQuota(ctx)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    fmt.Println("=== Storage Analytics ===")
    fmt.Printf("Total Limit: %.2f GB\n", float64(quota.Limit)/(1024*1024*1024))
    fmt.Printf("Used: %.2f GB (%.1f%%)\n", float64(quota.Usage)/(1024*1024*1024), float64(quota.Usage)/float64(quota.Limit)*100)
    fmt.Printf("In Trash: %.2f GB\n", float64(quota.UsageInDriveTrash)/(1024*1024*1024))
    fmt.Printf("Available: %.2f GB\n", float64(quota.Limit-quota.Usage)/(1024*1024*1024))
}
```

### Bulk File Operations

```go
func bulkUpload(manager *drive.GoogleDriveManager, files []struct { Name, Content string }) error {
    ctx := context.Background()
    
    for _, f := range files {
        metadata := drive.DriveUploadRequest{
            Name:    f.Name,
            MIMEType: "text/plain",
        }
        
        _, err := manager.UploadFile(ctx, metadata, strings.NewReader(f.Content))
        if err != nil {
            return fmt.Errorf("failed to upload %s: %w", f.Name, err)
        }
        fmt.Printf("Uploaded: %s\n", f.Name)
    }
    
    return nil
}
```

### Permission Audit

```go
func auditPermissions(manager *drive.GoogleDriveManager, fileID string) {
    ctx := context.Background()
    
    permissions, err := manager.ListPermissions(ctx, fileID)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    fmt.Printf("=== Permissions for file %s ===\n", fileID)
    for _, perm := range permissions {
        fmt.Printf("ID: %s\n", perm.ID)
        fmt.Printf("  Type: %s, Role: %s\n", perm.Type, perm.Role)
        if perm.EmailAddress != "" {
            fmt.Printf("  Email: %s\n", perm.EmailAddress)
        }
        if perm.Domain != "" {
            fmt.Printf("  Domain: %s\n", perm.Domain)
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
    │   ├── Authorization: Bearer {access_token}
    │   └── Content-Type: application/json (for POST/PATCH)
    ├── Execute via retryablehttp.Client
    ├── Check status code == 200 (http.StatusOK)
    ├── Read body
    └── JSON decode to target struct
```

### Multipart Upload Algorithm

```
UploadFile():
    │
    ├── Marshal metadata to JSON
    ├── Create multipart writer
    ├── Create "metadata" part → write metadata JSON
    ├── Create "media" part → copy content reader
    ├── Close multipart writer
    ├── Create POST request to /upload/drive/v3/files?uploadType=multipart
    ├── Set Content-Type: multipart/form-data; boundary=...
    ├── Set Authorization header
    ├── Execute request
    ├── Check status code
    └── JSON decode to DriveFile
```

### Search Query Building

```
ListFiles():
    │
    ├── Build url.Values:
    │   ├── q: query string (e.g., "name contains 'report'")
    │   ├── pageSize: max number of results
    │   ├── pageToken: for pagination
    │   ├── orderBy: field to sort by (e.g., "modifiedTime desc")
    │   ├── fields: specific fields to return
    │   └── spaces: "drive", "appDataFolder", or "photos"
    ├── Encode parameters to URL query string
    └── GET /files?{query_string}
```

### Retry Logic

Using `go-retryablehttp`:
- Maximum retries: 3
- Minimum wait: 1 second
- Maximum wait: 10 seconds
- HTTP client timeout: 120 seconds

Retries are automatic for transient network errors.

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.