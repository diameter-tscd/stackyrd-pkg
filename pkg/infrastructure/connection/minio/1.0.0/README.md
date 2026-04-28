# MinIO Manager

## Overview

The `MinIOManager` is a Go library for managing MinIO object storage operations using the official MinIO Go client. It provides file upload/download, object listing, deletion, presigned URL generation, and async operations via a worker pool.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Bucket Operations**: Check bucket existence
- **File Upload**: Upload files to MinIO bucket with content type
- **File Download**: Retrieve objects from MinIO
- **Object Listing**: List objects with prefix and recursive options
- **Object Deletion**: Delete objects from bucket
- **Object Info**: Get object metadata/info
- **Presigned URLs**: Generate presigned URLs for GET requests (7 days expiry)
- **Async Operations**: Async upload, download, delete, list, get info via worker pool
- **Batch Operations**: Batch upload and delete multiple objects
- **Worker Pool**: Async job execution support (8 workers)
- **Connection Test**: Validates connectivity via bucket listing

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
    
    // Create MinIO manager (configuration via config struct)
    cfg := config.MinIOConfig{
        Enabled:       true,
        Endpoint:      "localhost:9000",
        AccessKeyID:    "minioadmin",
        SecretAccessKey: "minioadmin",
        UseSSL:        false,
        BucketName:    "my-bucket",
    }
    
    manager, err := minio.NewMinIOManager(cfg)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Upload a file
    reader := strings.NewReader("Hello MinIO!")
    info, err := manager.UploadFile(ctx, "hello.txt", reader, int64(reader.Len()), "text/plain")
    if err != nil {
        panic(err)
    }
    fmt.Printf("Uploaded: %s (Size: %d)\n", info.Key, info.Size)
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `MinIOManager` | Main manager with MinIO client, bucket name, worker pool |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   MinIOManager                           │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  MinIO      │  → MinIO Go client (minio-go/v7)    │
│  │  Client     │  → Bucket operations                   │
│  └────────────────┘                                      │
│                                                         │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ AsyncResult │                   │
│  │  (8 workers)│      │   Channel   │                   │
│  └─────────────┘      └──────────────┘                   │
│         ▲                      │                            │
│         │                      ▼                            │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  SubmitJob  │      │ ExecuteAsync │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                         │
│  Endpoint: MinIO server address (e.g., localhost:9000)   │
│  BucketName: Default bucket for operations               │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewMinIOManager(cfg)
    │
    ├── Check cfg.Enabled and cfg.Endpoint
    ├── Create MinIO client:
    │   ├── minio.New(cfg.Endpoint, &minio.Options{
    │   │   Creds:  credentials.NewStaticV4(accessKey, secretKey, ""),
    │   │   Secure: cfg.UseSSL,
    │   │ })
    │   └── Return error if fails
    │
    ├── Test connection: client.ListBuckets(context.Background())
    ├── Create WorkerPool(8)
    ├── Start worker pool
    └── Return MinIOManager
```

### 2. Upload File Flow

```
UploadFile(ctx, objectName, reader, objectSize, contentType)
    │
    └── client.PutObject(ctx, bucketName, objectName, reader, objectSize,
                          minio.PutObjectOptions{ContentType: contentType})
```

### 3. Get Object Flow

```
GetObject(ctx, objectName)
    │
    └── client.GetObject(ctx, bucketName, objectName, minio.GetObjectOptions{})
```

### 4. List Objects Flow

```
ListObjects(ctx, prefix, recursive)
    │
    ├── client.ListObjects(ctx, bucketName, minio.ListObjectsOptions{
    │       Prefix: prefix, Recursive: recursive})
    ├── Iterate over objectCh
    └── Return []minio.ObjectInfo
```

### 5. Delete Object Flow

```
DeleteObject(ctx, objectName)
    │
    └── client.RemoveObject(ctx, bucketName, objectName, minio.RemoveObjectOptions{})
```

### 6. Get Object Info Flow

```
GetObjectInfo(ctx, objectName)
    │
    └── client.StatObject(ctx, bucketName, objectName, minio.StatObjectOptions{})
```

### 7. Presigned URL Flow

```
GetFileUrl(objectName)
    │
    └── client.PresignedGetObject(context.Background(), bucketName, objectName, 7*24*time.Hour, nil)
```

## Configuration

### Config Struct (passed to NewMinIOManager)

| Field | Type | Description |
|-----|------|-------------|
| `Enabled` | bool | Enable/disable MinIO manager |
| `Endpoint` | string | MinIO server endpoint (host:port) |
| `AccessKeyID` | string | Access key ID |
| `SecretAccessKey` | string | Secret access key |
| `UseSSL` | bool | Use SSL/TLS for connection |
| `BucketName` | string | Default bucket name |

### Example Configuration

```go
cfg := config.MinIOConfig{
    Enabled:          true,
    Endpoint:         "localhost:9000",
    AccessKeyID:       "minioadmin",
    SecretAccessKey:   "minioadmin",
    UseSSL:           false,
    BucketName:       "my-bucket",
}
```

Note: The component is registered under `monitoring.minio` in config, so you might configure via:

```go
viper.Set("monitoring.minio.enabled", true)
viper.Set("monitoring.minio.endpoint", "localhost:9000")
viper.Set("monitoring.minio.access_key_id", "minioadmin")
viper.Set("monitoring.minio.secret_access_key", "minioadmin")
viper.Set("monitoring.minio.use_ssl", false)
viper.Set("monitoring.minio.bucket_name", "my-bucket")
```

## Usage Examples

### Uploading a File

```go
ctx := context.Background()

// Upload a string as a file
reader := strings.NewReader("Hello, MinIO!")
info, err := manager.UploadFile(ctx, "hello.txt", reader, int64(reader.Len()), "text/plain")
if err != nil {
    panic(err)
}
fmt.Printf("Uploaded: %s (ETag: %s)\n", info.Key, info.ETag)
```

### Downloading an Object

```go
ctx := context.Background()

// Get object
object, err := manager.GetObject(ctx, "hello.txt")
if err != nil {
    panic(err)
}
defer object.Close()

// Read content
data, err := io.ReadAll(object)
if err != nil {
    panic(err)
}
fmt.Printf("Content: %s\n", string(data))
```

### Listing Objects

```go
ctx := context.Background()

// List objects with prefix
objects, err := manager.ListObjects(ctx, "prefix/", true)
if err != nil {
    panic(err)
}

for _, obj := range objects {
    fmt.Printf("Object: %s (Size: %d, LastModified: %v)\n", 
        obj.Key, obj.Size, obj.LastModified)
}
```

### Deleting an Object

```go
ctx := context.Background()

err := manager.DeleteObject(ctx, "hello.txt")
if err != nil {
    panic(err)
}
fmt.Println("Object deleted!")
```

### Getting Object Info

```go
ctx := context.Background()

info, err := manager.GetObjectInfo(ctx, "hello.txt")
if err != nil {
    panic(err)
}
fmt.Printf("Object Info:\n")
fmt.Printf("  Key: %s\n", info.Key)
fmt.Printf("  Size: %d\n", info.Size)
fmt.Printf("  ETag: %s\n", info.ETag)
fmt.Printf("  ContentType: %s\n", info.ContentType)
```

### Generating Presigned URL

```go
url := manager.GetFileUrl("hello.txt")
if url == "" {
    fmt.Println("Failed to generate URL")
} else {
    fmt.Printf("Presigned URL: %s\n", url)
}
```

### Async Operations

```go
ctx := context.Background()

// Async upload
reader := strings.NewReader("Async content")
result := manager.UploadFileAsync(ctx, "async.txt", reader, int64(reader.Len()), "text/plain")
select {
case info := <-result.Ch:
    fmt.Printf("Async upload complete: %s\n", info.Key)
case err := <-result.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async get object
result2 := manager.GetObjectAsync(ctx, "async.txt")
select {
case obj := <-result2.Ch:
    fmt.Println("Got object asynchronously!")
    obj.Close()
case err := <-result2.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async delete
result3 := manager.DeleteObjectAsync(ctx, "async.txt")
select {
case <-result3.Ch:
    fmt.Println("Deleted asynchronously!")
case err := <-result3.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async list objects
result4 := manager.ListObjectsAsync(ctx, "", true)
select {
case objects := <-result4.Ch:
    fmt.Printf("Found %d objects\n", len(objects))
case err := <-result4.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async get object info
result5 := manager.GetObjectInfoAsync(ctx, "hello.txt")
select {
case info := <-result5.Ch:
    fmt.Printf("Object info: %s\n", info.Key)
case err := <-result5.ErrCh:
    fmt.Printf("Error: %v\n", err)
}
```

### Batch Operations

```go
ctx := context.Background()

// Batch upload
uploads := []struct {
    ObjectName  string
    Reader      io.Reader
    ObjectSize  int64
    ContentType string
}{
    {"file1.txt", strings.NewReader("Content 1"), 9, "text/plain"},
    {"file2.txt", strings.NewReader("Content 2"), 9, "text/plain"},
}
result := manager.UploadBatchAsync(ctx, uploads)
select {
case infos := <-result.Ch:
    fmt.Printf("Batch uploaded %d files\n", len(infos))
case err := <-result.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Batch delete
objectNames := []string{"file1.txt", "file2.txt"}
result2 := manager.DeleteBatchAsync(ctx, objectNames)
select {
case <-result2.Ch:
    fmt.Println("Batch deleted!")
case err := <-result2.ErrCh:
    fmt.Printf("Error: %v\n", err)
}
```

### Get Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Bucket: %s\n", status["bucket_name"])
fmt.Printf("Object Count: %v\n", status["object_count"])
fmt.Printf("Total Size (KB): %v\n", status["total_size_kb"])
fmt.Printf("Status: %s\n", status["status"])
fmt.Printf("Endpoint: %s\n", status["endpoint"])

if status["connected"] == false {
    if errMsg, ok := status["error"].(string); ok {
        fmt.Printf("Error: %s\n", errMsg)
    }
}
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/minio/minio-go/v7` | MinIO Go client library |
| `github.com/minio/minio-go/v7/pkg/credentials` | Credentials provider |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `context`, `io`, `time` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: `ListBuckets()` validates connectivity
- **Bucket not found**: `GetStatus()` checks bucket existence
- **API errors**: Returned from MinIO client operations
- **Context cancellation**: All operations respect `context.Context`
- **Nil checks**: Public methods handle nil receiver gracefully in `GetStatus()`

Example error handling:

```go
info, err := manager.UploadFile(ctx, "test.txt", reader, size, "text/plain")
if err != nil {
    if strings.Contains(err.Error(), "bucket not found") {
        return fmt.Errorf("bucket does not exist: %w", err)
    }
    if strings.Contains(err.Error(), "authentication") {
        return fmt.Errorf("invalid credentials: %w", err)
    }
    return fmt.Errorf("failed to upload: %w", err)
}
```

## Common Pitfalls

### 1. MinIO Server Not Reachable

**Problem**: `connection refused` or `no route to host` errors

**Solution**: 
- Verify endpoint: `cfg.Endpoint = "localhost:9000"`
- Check network connectivity (ping/telnet to MinIO port)
- Verify firewall allows access to MinIO port (default: 9000)

### 2. Invalid Credentials

**Problem**: `authentication error` or `access denied`

**Solution**:
- Verify AccessKeyID and SecretAccessKey
- Check that credentials have proper permissions
- Ensure user has write access to bucket

### 3. Bucket Not Found

**Problem**: `bucket not found` errors

**Solution**:
- Create bucket manually via MinIO client or console
- Verify bucket name spelling
- Check that bucket exists in MinIO server

### 4. SSL/TLS Issues

**Problem**: SSL certificate errors when UseSSL is true

**Solution**:
- For development, set `UseSSL: false`
- For production, ensure proper certificates are configured
- Verify MinIO server is configured with SSL

### 5. Context Timeout

**Problem**: Operations timing out

**Solution**:
```go
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
```

### 6. Worker Pool Not Available

**Problem**: Async operations falling back to direct execution

**Solution**:
- The pool is created during initialization
- Check that `NewMinIOManager()` succeeded
- Verify `cfg.Enabled` is true

## Advanced Usage

### Monitoring Bucket Usage

```go
func monitorBucket(manager *minio.MinIOManager) {
    ticker := time.NewTicker(1 * time.Minute)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            status := manager.GetStatus()
            if status["connected"] == true {
                fmt.Printf("Bucket: %s, Objects: %v, Size: %v KB\n",
                    status["bucket_name"], status["object_count"], status["total_size_kb"])
            } else {
                fmt.Println("WARNING: MinIO disconnected!")
            }
        }
    }
}
```

### Uploading Large Files

```go
func uploadLargeFile(manager *minio.MinIOManager, filePath, objectName string) error {
    ctx := context.Background()
    
    // Open file
    file, err := os.Open(filePath)
    if err != nil {
        return fmt.Errorf("failed to open file: %w", err)
    }
    defer file.Close()
    
    // Get file size
    stat, err := file.Stat()
    if err != nil {
        return fmt.Errorf("failed to get file stats: %w", err)
    }
    
    // Upload
    info, err := manager.UploadFile(ctx, objectName, file, stat.Size(), "application/octet-stream")
    if err != nil {
        return fmt.Errorf("failed to upload: %w", err)
    }
    
    fmt.Printf("Uploaded large file: %s (Size: %d)\n", info.Key, info.Size)
    return nil
}
```

### Batch Upload from Directory

```go
func uploadDirectory(manager *minio.MinIOManager, dirPath, prefix string) error {
    ctx := context.Background()
    
    files, err := os.ReadDir(dirPath)
    if err != nil {
        return err
    }
    
    var uploads []struct {
        ObjectName  string
        Reader      io.Reader
        ObjectSize  int64
        ContentType string
    }
    
    for _, file := range files {
        if file.IsDir() {
            continue
        }
        
        filePath := filepath.Join(dirPath, file.Name())
        f, err := os.Open(filePath)
        if err != nil {
            continue
        }
        
        stat, _ := f.Stat()
        uploads = append(uploads, struct {
            ObjectName  string
            Reader      io.Reader
            ObjectSize  int64
            ContentType string
        }{
            ObjectName:  prefix + file.Name(),
            Reader:      f,
            ObjectSize:  stat.Size(),
            ContentType: "application/octet-stream",
        })
    }
    
    result := manager.UploadBatchAsync(ctx, uploads)
    select {
    case infos := <-result.Ch:
        fmt.Printf("Uploaded %d files from directory\n", len(infos))
        return nil
    case err := <-result.ErrCh:
        return fmt.Errorf("batch upload failed: %w", err)
    }
}
```

### Presigned URL with Custom Expiry

```go
func getPresignedURL(manager *minio.MinIOManager, objectName string, expiry time.Duration) (string, error) {
    url, err := manager.Client.PresignedGetObject(context.Background(), manager.BucketName, objectName, expiry, nil)
    if err != nil {
        return "", fmt.Errorf("failed to generate presigned URL: %w", err)
    }
    return url.String(), nil
}
```

## Internal Algorithms

### Connection Test

```
NewMinIOManager():
    │
    ├── minio.New(endpoint, options)
    ├── client.ListBuckets(ctx) → test connection
    ├── If error: return with Connected=false
    ├── Create WorkerPool(8)
    ├── pool.Start()
    └── Return manager with Connected=true
```

### Object Upload

```
UploadFile():
    │
    └── client.PutObject(ctx, bucketName, objectName, reader, objectSize, options)
        └── Returns (UploadInfo, error)
```

### Object Listing with Limit

```
GetStatus():
    │
    ├── client.BucketExists(ctx, bucketName)
    ├── client.ListObjects(ctx, bucketName, options)
    ├── Iterate up to 1000 objects
    │   ├── count++
    │   └── size += obj.Size
    └── Return stats map
```

### Async Operation Pattern

```
UploadFileAsync(ctx, objectName, reader, objectSize, contentType):
    │
    └── ExecuteAsync(ctx, func):
        └── return UploadFile(ctx, objectName, reader, objectSize, contentType)
            └── Returns (minio.UploadInfo, error)
```

### Batch Upload

```
UploadBatchAsync(ctx, uploads):
    │
    ├── For each upload:
    │   └── Create AsyncOperation that calls UploadFile
    │
    └── ExecuteBatchAsync(ctx, operations)
        └── Returns BatchAsyncResult[minio.UploadInfo]
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.