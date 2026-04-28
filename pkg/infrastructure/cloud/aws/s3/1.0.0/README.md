# Amazon S3 Manager

## Overview

The `AmazonS3Manager` is a comprehensive Go library for managing AWS S3 operations using the official AWS SDK for Go. It provides complete S3 resource management including buckets, objects, multipart uploads, presigned URLs, and transfer tracking with async support.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Bucket Management**: Create, list, delete, and configure S3 buckets
- **Object Operations**: Upload, download, copy, delete, and list objects
- **Bucket Configuration**: Policy, versioning, encryption, CORS, lifecycle, website, tagging
- **Multipart Upload**: Handle large file uploads with progress tracking
- **Presigned URLs**: Generate presigned URLs for GET, PUT, and DELETE operations
- **Transfer Tracking**: Monitor upload/download progress with status and speed
- **Async Operations**: Async execution for all operations via worker pool
- **AWS Authentication**: Supports static credentials via AWS SDK configuration
- **Retry Logic**: Built-in AWS SDK retry logic

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
    
    // Create Amazon S3 manager (configuration via viper)
    manager, err := s3.NewAmazonS3Manager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // List S3 buckets
    buckets, err := manager.ListBuckets(ctx)
    if err != nil {
        panic(err)
    }
    
    for _, bucket := range buckets {
        fmt.Printf("Bucket: %s (Created: %v)\n", bucket.Name, bucket.CreationDate)
    }
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `AmazonS3Manager` | Main manager with S3 client and presign client |
| `S3Bucket` | S3 bucket with name, creation date, region |
| `S3Object` | S3 object with key, size, metadata, versioning |
| `S3ObjectVersion` | Object version with version ID and delete markers |
| `S3UploadRequest` | Upload request with bucket, key, body, metadata |
| `S3DownloadResult` | Download result with body, content type, metadata |
| `S3CopyRequest` | Copy request between buckets/keys |
| `S3MultipartUpload` | Multipart upload session information |
| `S3CompletedPart` | Completed upload part with ETag and size |
| `S3TransferProgress` | Transfer progress with bytes, percent, speed |
| `S3PresignedURLRequest` | Presigned URL request parameters |
| `S3PresignedURLResult` | Presigned URL result with expiration |
| `S3BucketPolicy` | Bucket policy JSON |
| `S3LifecycleRule` | Lifecycle rule with transitions and expiration |
| `S3CORSRule` | CORS rule with allowed methods and origins |
| `S3NotificationConfig` | Bucket notification configuration |
| `S3EncryptionConfig` | Bucket encryption configuration |
| `S3WebsiteConfig` | Bucket website configuration |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   AmazonS3Manager                        │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  S3 Client  │  → AWS SDK with credentials           │
│  │  (official)  │  → Built-in retry logic               │
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
│  Region: AWS region (default: us-east-1)                       │
│  Bucket: Default bucket (optional)                         │
│  Endpoint: For S3-compatible services (e.g., MinIO)             │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewAmazonS3Manager()
    │
    ├── Check viper config: "amazons3.enabled"
    ├── Get region: "amazons3.region" (default: "us-east-1")
    ├── Get credentials: access_key, secret_key
    ├── Get bucket: "amazons3.bucket"
    ├── Get endpoint: "amazons3.endpoint" (for S3-compatible)
    ├── Get path_style: "amazons3.path_style" (for MinIO)
    ├── Load AWS config (static credentials or default)
    ├── Create S3 client with options
    │   ├── If path_style: set UsePathStyle = true
    │   └── If endpoint: set BaseEndpoint
    ├── Create PresignClient
    ├── Test connection: ListBuckets()
    ├── Create WorkerPool(8)
    ├── Start worker pool
    └── Return AmazonS3Manager
```

### 2. Put Object Flow

```
PutObject(ctx, req)
    │
    ├── Create PutObjectInput
    │   ├── Bucket, Key
    │   ├── Body (from req.Body)
    │   ├── ContentType, Metadata, ACL
    │   ├── StorageClass, EncryptSSE
    │   └── Tags (URL encoded)
    ├── Execute Client.PutObject()
    ├── Return S3Object with Key, ETag, VersionID
    └── Track transfer progress (if tracked)
```

### 3. Get Object Flow

```
GetObject(ctx, bucket, key, versionID)
    │
    ├── Create GetObjectInput
    │   ├── Bucket, Key
    │   └── VersionId (if specified)
    ├── Execute Client.GetObject()
    ├── Read metadata from result
    └── Return S3DownloadResult with Body, ContentType, Metadata
```

### 4. Multipart Upload Flow

```
UploadMultipartFromReader(ctx, req, partSize)
    │
    ├── CreateMultipartUpload() → get UploadID
    ├── Track transfer: transfers[uploadID]
    │
    ├── For each part (read partSize chunks):
    │   ├── UploadPart(uploadID, partNumber, data)
    │   ├── Record ETag and size
    │   └── Update progress
    │
    ├── CompleteMultipartUpload(uploadID, parts)
    │
    └── Return S3Object with VersionID
```

### 5. Presigned URL Flow

```
GeneratePresignedURL(ctx, req)
    │
    ├── Create input based on method (GET/PUT/DELETE)
    ├── Set expiration: ExpiresIn
    ├── For PUT: set ContentType
    ├── Execute PresignClient.PresignGet/Put/Delete()
    └── Return S3PresignedURLResult with URL and headers
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `amazons3.enabled` | bool | false | Enable/disable Amazon S3 manager |
| `amazons3.region` | string | "us-east-1" | AWS region |
| `amazons3.bucket` | string | "" | Default bucket name |
| `amazons3.access_key` | string | "" | AWS access key ID |
| `amazons3.secret_key` | string | "" | AWS secret access key |
| `amazons3.endpoint` | string | "" | Custom endpoint (for S3-compatible) |
| `amazons3.path_style` | bool | false | Use path-style URLs (for MinIO) |

### Environment Variables

The S3 manager uses the AWS SDK which can load credentials from:
- AWS credentials file (~/.aws/credentials)
- Environment variables: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`
- IAM roles (when running on EC2)
- AWS CLI configuration

## Usage Examples

### Bucket Operations

```go
ctx := context.Background()

// List all S3 buckets
buckets, err := manager.ListBuckets(ctx)
if err != nil {
    panic(err)
}

for _, bucket := range buckets {
    fmt.Printf("Bucket: %s\n", bucket.Name)
    fmt.Printf("  Created: %v\n", bucket.CreationDate)
    fmt.Printf("  Region: %s\n", bucket.Region)
}

// Create a bucket
err = manager.CreateBucket(ctx, "my-new-bucket", "us-west-2")
if err != nil {
    panic(err)
}

// Check if bucket exists
err = manager.HeadBucket(ctx, "my-bucket")
if err != nil {
    fmt.Println("Bucket does not exist or is not accessible")
}

// Get bucket location
location, err := manager.GetBucketLocation(ctx, "my-bucket")
if err != nil {
    panic(err)
}
fmt.Printf("Bucket location: %s\n", location)

// Delete bucket
err = manager.DeleteBucket(ctx, "my-old-bucket")
if err != nil {
    panic(err)
}
```

### Bucket Configuration

```go
ctx := context.Background()

// Get and set bucket policy
policy, err := manager.GetBucketPolicy(ctx, "my-bucket")
if err != nil {
    panic(err)
}
fmt.Printf("Current policy: %s\n", policy.Policy)

newPolicy := `{"Version":"2012-10-17", "Statement":[...]}`
err = manager.PutBucketPolicy(ctx, "my-bucket", newPolicy)
if err != nil {
    panic(err)
}

// Get and set versioning
versioning, err := manager.GetBucketVersioning(ctx, "my-bucket")
if err != nil {
    panic(err)
}
fmt.Printf("Versioning: %s\n", versioning)

err = manager.PutBucketVersioning(ctx, "my-bucket", true)
if err != nil {
    panic(err)
}

// Get and set encryption
encryption, err := manager.GetBucketEncryption(ctx, "my-bucket")
if err != nil {
    panic(err)
}
fmt.Printf("Encryption: Algorithm=%s, KMS=%s\n", encryption.Algorithm, encryption.KMSKeyID)

err = manager.PutBucketEncryption(ctx, "my-bucket", &s3.S3EncryptionConfig{
    Algorithm: "AES256",
})
if err != nil {
    panic(err)
}

// Get and set CORS
corsRules, err := manager.GetBucketCORS(ctx, "my-bucket")
if err != nil {
    panic(err)
}

for _, rule := range corsRules {
    fmt.Printf("CORS Rule: %s\n", rule.ID)
    fmt.Printf("  Allowed Origins: %v\n", rule.AllowedOrigins)
    fmt.Printf("  Allowed Methods: %v\n", rule.AllowedMethods)
}

newCORSRules := []s3.S3CORSRule{
    {
        ID: "allow-all",
        AllowedOrigins: []string{"*"},
        AllowedMethods: []string{"GET", "PUT", "POST", "DELETE"},
    },
}
err = manager.PutBucketCORS(ctx, "my-bucket", newCORSRules)
if err != nil {
    panic(err)
}

// Get and set lifecycle
lifecycleRules, err := manager.GetBucketLifecycle(ctx, "my-bucket")
if err != nil {
    panic(err)
}

// Set lifecycle rule to delete objects after 30 days
newLifecycleRules := []s3.S3LifecycleRule{
    {
        ID:            "delete-old-versions",
        Status:        "Enabled",
        Prefix:        "logs/",
        ExpirationDays: 30,
    },
}
err = manager.PutBucketLifecycle(ctx, "my-bucket", newLifecycleRules)
if err != nil {
    panic(err)
}

// Get and set website configuration
website, err := manager.GetBucketWebsite(ctx, "my-bucket")
if err != nil {
    panic(err)
}
fmt.Printf("Website: Index=%s, Error=%s\n", website.IndexDocument, website.ErrorDocument)

err = manager.PutBucketWebsite(ctx, "my-bucket", &s3.S3WebsiteConfig{
    IndexDocument: "index.html",
    ErrorDocument: "error.html",
})
if err != nil {
    panic(err)
}

// Get and set bucket tags
tags, err := manager.GetBucketTagging(ctx, "my-bucket")
if err != nil {
    panic(err)
}
fmt.Printf("Tags: %v\n", tags)

newTags := map[string]string{
    "Environment": "production",
    "Team":        "backend",
}
err = manager.PutBucketTagging(ctx, "my-bucket", newTags)
if err != nil {
    panic(err)
}
```

### Object Operations

```go
ctx := context.Background()

// Upload an object
uploadReq := &s3.S3UploadRequest{
    Bucket:      "my-bucket",
    Key:         "path/to/file.txt",
    Body:        strings.NewReader("Hello, S3!"),
    ContentType: "text/plain",
    Metadata:    map[string]string{"author": "system"},
    ACL:         "private",
    StorageClass: "STANDARD",
}

obj, err := manager.PutObject(ctx, uploadReq)
if err != nil {
    panic(err)
}
fmt.Printf("Uploaded: %s (ETag: %s, Version: %s)\n", obj.Key, obj.ETag, obj.VersionID)

// Download an object
result, err := manager.GetObject(ctx, "my-bucket", "path/to/file.txt", "")
if err != nil {
    panic(err)
}
defer result.Body.Close()

data, _ := io.ReadAll(result.Body)
fmt.Printf("Downloaded: %s\n", string(data))
fmt.Printf("Content Type: %s\n", result.ContentType)
fmt.Printf("Size: %d bytes\n", result.ContentLength)

// Get object metadata only (HEAD request)
metadata, err := manager.HeadObject(ctx, "my-bucket", "path/to/file.txt", "")
if err != nil {
    panic(err)
}
fmt.Printf("Size: %d bytes\n", metadata.Size)
fmt.Printf("Last Modified: %v\n", metadata.LastModified)
fmt.Printf("Storage Class: %s\n", metadata.StorageClass)

// Copy object to another location
copyReq := &s3.S3CopyRequest{
    SourceBucket: "my-bucket",
    SourceKey:    "path/to/source.txt",
    DestBucket:   "my-bucket",
    DestKey:      "path/to/dest.txt",
}

copied, err := manager.CopyObject(ctx, copyReq)
if err != nil {
    panic(err)
}
fmt.Printf("Copied: %s\n", copied.Key)

// List objects in bucket
objects, nextToken, err := manager.ListObjects(ctx, "my-bucket", "logs/", "", 1000)
if err != nil {
    panic(err)
}

for _, obj := range objects {
    fmt.Printf("Object: %s (Size: %d)\n", obj.Key, obj.Size)
}
if nextToken != "" {
    fmt.Println("More objects available...")
}

// List all objects (handles pagination)
allObjects, err := manager.ListAllObjects(ctx, "my-bucket", "logs/")
if err != nil {
    panic(err)
}
fmt.Printf("Total objects: %d\n", len(allObjects))

// List object versions
versions, nextKey, nextVersionID, err := manager.ListObjectVersions(ctx, "my-bucket", "logs/", "", "", 1000)
if err != nil {
    panic(err)
}

for _, v := range versions {
    fmt.Printf("Version: %s (Key: %s, Latest: %v)\n", v.VersionID, v.Key, v.IsLatest)
}

// Delete an object
err = manager.DeleteObject(ctx, "my-bucket", "path/to/file.txt", "")
if err != nil {
    panic(err)
}

// Delete multiple objects
keys := []string{"file1.txt", "file2.txt", "file3.txt"}
err = manager.DeleteObjects(ctx, "my-bucket", keys)
if err != nil {
    panic(err)
}
```

### Multipart Upload

```go
ctx := context.Background()

// Upload large file using multipart upload
file, err := os.Open("/path/to/large-file.zip")
if err != nil {
    panic(err)
}
defer file.Close()

uploadReq := &s3.S3UploadRequest{
    Bucket:      "my-bucket",
    Key:         "large-file.zip",
    Body:        file,
    ContentType: "application/zip",
}

// Track progress
go func() {
    for {
        progress, exists := manager.GetTransferProgress("large-file.zip")
        if !exists {
            time.Sleep(500 * time.Millisecond)
            continue
        }
        fmt.Printf("Progress: %.1f%% (%.2f MB/s)\n", progress.Percent, progress.SpeedMBps)
        if progress.Status == "complete" || progress.Status == "error" {
            break
        }
    }
}()

obj, err := manager.UploadMultipartFromReader(ctx, uploadReq, 10*1024*1024) // 10MB parts
if err != nil {
    panic(err)
}
fmt.Printf("Upload complete: %s\n", obj.Key)

// Get all transfers
transfers := manager.GetAllTransferProgress()
fmt.Printf("Active transfers: %d\n", len(transfers))

// Clear completed transfers
manager.ClearCompletedTransfers()
```

### Presigned URLs

```go
ctx := context.Background()

// Generate presigned GET URL
getReq := &s3.S3PresignedURLRequest{
    Bucket:    "my-bucket",
    Key:       "private/file.txt",
    Method:    "get",
    ExpiresIn: 15 * time.Minute,
}

urlResult, err := manager.GeneratePresignedURL(ctx, getReq)
if err != nil {
    panic(err)
}
fmt.Printf("Presigned GET URL: %s\n", urlResult.URL)
fmt.Printf("Expires in: %v\n", urlResult.ExpiresIn)

// Generate presigned PUT URL
putReq := &s3.S3PresignedURLRequest{
    Bucket:      "my-bucket",
    Key:         "uploads/new-file.txt",
    Method:      "put",
    ExpiresIn:   30 * time.Minute,
    ContentType: "text/plain",
}

urlResult, err = manager.GeneratePresignedURL(ctx, putReq)
if err != nil {
    panic(err)
}
fmt.Printf("Presigned PUT URL: %s\n", urlResult.URL)

// Convenience methods
getURL, err := manager.GeneratePresignedGetURL(ctx, "my-bucket", "file.txt", 15*time.Minute)
if err != nil {
    panic(err)
}

putURL, err := manager.GeneratePresignedPutURL(ctx, "my-bucket", "upload.txt", "text/plain", 30*time.Minute)
if err != nil {
    panic(err)
}
```

### Async Operations

```go
ctx := context.Background()

// Async list buckets
result := manager.ListBucketsAsync(ctx)
select {
case buckets := <-result.Ch:
    fmt.Printf("Found %d buckets\n", len(buckets))
case err := <-result.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async upload
uploadReq := &s3.S3UploadRequest{
    Bucket: "my-bucket",
    Key:    "async-file.txt",
    Body:   strings.NewReader("Async upload test"),
}

result = manager.PutObjectAsync(ctx, uploadReq)
select {
case obj := <-result.Ch:
    fmt.Printf("Uploaded: %s\n", obj.Key)
case err := <-result.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async download
result = manager.GetObjectAsync(ctx, "my-bucket", "file.txt", "")
select {
case download := <-result.Ch:
    defer download.Body.Close()
    data, _ := io.ReadAll(download.Body)
    fmt.Printf("Downloaded: %s\n", string(data))
case err := <-result.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async delete
result = manager.DeleteObjectAsync(ctx, "my-bucket", "old-file.txt", "")
select {
case <-result.Ch:
    fmt.Println("Deleted successfully")
case err := <-result.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async batch delete
keys := []string{"file1.txt", "file2.txt"}
batchResult := manager.DeleteBatchAsync(ctx, "my-bucket", keys)
select {
case <-batchResult.Ch:
    fmt.Println("Batch delete complete")
case err := <-batchResult.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Other async methods:
// - CreateBucketAsync
// - DeleteBucketAsync
// - CopyObjectAsync
// - UploadFromPathAsync
// - DownloadToPathAsync
// - ListObjectsAsync
// - ListAllObjectsAsync
// - GeneratePresignedURLAsync
```

### Get Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Region: %s\n", status["region"])
fmt.Printf("Default Bucket: %s\n", status["default_bucket"])
fmt.Printf("Endpoint: %s\n", status["endpoint"])
fmt.Printf("Bucket Count: %v\n", status["bucket_count"])
fmt.Printf("Active Transfers: %v\n", status["active_transfers"])

if status["connected"] == false {
    if errMsg, ok := status["error"].(string); ok {
        fmt.Printf("Error: %s\n", errMsg)
    }
}
```

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/aws/aws-sdk-go-v2` | Official AWS SDK for Go |
| `github.com/aws/aws-sdk-go-v2/config` | AWS configuration loading |
| `github.com/aws/aws-sdk-go-v2/credentials` | Static credentials provider |
| `github.com/aws/aws-sdk-go-v2/service/s3` | S3 service client |
| `github.com/aws/aws-sdk-go-v2/aws/signer/v4` | AWS v4 signing |
| `github.com/aws/smithy-go` | Smithy middleware for AWS SDK |
| `github.com/spf13/viper` | Configuration management |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `context`, `encoding/json`, `bytes`, `io`, `time`, `fmt`, `net/url`, `os`, `path`, `strings`, `sync` |

## Error Handling

The library uses Go error handling patterns:

- **SDK errors**: AWS SDK returns typed errors (smithy.APIError)
- **Connection failures**: `testConnection()` validates via `ListBuckets()`
- **Context cancellation**: All operations respect `context.Context` for timeouts
- **Nil checks**: Public methods handle nil receiver gracefully in `GetStatus()`

Example error handling:

```go
buckets, err := manager.ListBuckets(ctx)
if err != nil {
    var apiErr *types.NoSuchBucket
    if errors.As(err, &apiErr) {
        return fmt.Errorf("bucket not found: %w", err)
    }
    var apiErr2 *types.BucketAlreadyExists
    if errors.As(err, &apiErr2) {
        return fmt.Errorf("bucket already exists: %w", err)
    }
    return fmt.Errorf("failed to list buckets: %w", err)
}
```

## Common Pitfalls

### 1. AWS Credentials Not Configured

**Problem**: `NoCredentialProviders` or `InvalidAccessKeyId` errors

**Solution**: 
- Configure credentials: `viper.Set("amazons3.access_key", "AKIA...")`
- Or use AWS CLI: `aws configure`
- Or set environment variables: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`
- Or use IAM role when running on EC2

### 2. Wrong Region

**Problem**: Resources not found or empty results

**Solution**:
- Check region: `viper.Set("amazons3.region", "us-west-2")`
- Verify resources exist in the specified region
- Common regions: us-east-1, us-west-2, eu-west-1

### 3. Bucket Not Found

**Problem**: `NoSuchBucket` error

**Solution**:
- Verify bucket name format (3-63 chars, lowercase, no underscores)
- Check bucket exists in the region
- Use `HeadBucket()` to check existence

### 4. Context Timeout

**Problem**: Operations timing out

**Solution**:
```go
ctx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
defer cancel()
// Operations will use this timeout
```

### 5. Large File Uploads

**Problem**: Memory issues with large files

**Solution**:
- Use `UploadMultipartFromReader()` for files > 100MB
- Specify appropriate part size (default: 10MB)
- Use `UploadFromPath()` for file paths

### 6. S3-Compatible Services (MinIO)

**Problem**: DNS-style URLs not working

**Solution**:
- Set path-style: `viper.Set("amazons3.path_style", true)`
- Set custom endpoint: `viper.Set("amazons3.endpoint", "http://localhost:9000")`

### 7. Access Denied

**Problem**: `AccessDenied` error

**Solution**:
- Check bucket/object permissions
- Verify IAM policy allows S3 operations
- Check bucket policy

## Advanced Usage

### Upload File from Path

```go
func uploadFile(manager *s3.AmazonS3Manager, bucket, localPath, key string) error {
    ctx := context.Background()
    
    req := &s3.S3UploadRequest{
        Bucket:      bucket,
        Key:         key,
        Body:        nil, // Will be set by UploadFromPath
        ContentType: "application/octet-stream",
    }
    
    // This method handles large files automatically
    _, err := manager.UploadFromPath(ctx, req, localPath)
    return err
}
```

### Download to Path

```go
func downloadFile(manager *s3.AmazonS3Manager, bucket, key, localPath string) error {
    ctx := context.Background()
    return manager.DownloadToPath(ctx, bucket, key, localPath)
}
```

### Bucket Analytics

```go
func analyzeBucket(manager *s3.AmazonS3Manager, bucket string) {
    ctx := context.Background()
    
    // Get bucket size
    totalSize, objectCount, err := manager.GetBucketSize(ctx, bucket)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    fmt.Printf("Bucket: %s\n", bucket)
    fmt.Printf("  Total Size: %.2f GB\n", float64(totalSize)/(1024*1024*1024))
    fmt.Printf("  Object Count: %d\n", objectCount)
    
    // Get storage class breakdown
    classes, err := manager.GetBucketStorageClasses(ctx, bucket)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    fmt.Println("  Storage Classes:")
    for class, size := range classes {
        fmt.Printf("    %s: %.2f MB\n", class, float64(size)/(1024*1024))
    }
}
```

### S3 Event Processing

```go
type S3EventHandler struct {
    s3Manager *s3.AmazonS3Manager
}

func (h *S3EventHandler) HandleS3Event(eventData map[string]interface{}) error {
    ctx := context.Background()
    
    // Parse S3 event (from Lambda/SQS/SNS)
    records := eventData["Records"].([]interface{})
    
    for _, record := range records {
        r := record.(map[string]interface{})
        s3Info := r["s3"].(map[string]interface{})
        bucket := s3Info["bucket"].(map[string]interface{})["name"].(string)
        obj := s3Info["object"].(map[string]interface{})
        key := obj["key"].(string)
        
        fmt.Printf("S3 Event: %s/%s\n", bucket, key)
        
        // Process the object
        // ...
    }
    
    return nil
}
```

### Transfer Monitoring

```go
func monitorTransfers(manager *s3.AmazonS3Manager) {
    ticker := time.NewTicker(2 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            transfers := manager.GetAllTransferProgress()
            
            if len(transfers) == 0 {
                fmt.Println("No active transfers")
                continue
            }
            
            fmt.Println("\n=== Active Transfers ===")
            for _, p := range transfers {
                fmt.Printf("  %s: %.1f%% (%s, %.2f MB/s)\n", p.Key, p.Percent, p.Status, p.SpeedMBps)
                if p.Error != "" {
                    fmt.Printf("    Error: %s\n", p.Error)
                }
            }
            
            manager.ClearCompletedTransfers()
        }
    }
}
```

## Internal Algorithms

### AWS SDK Client Creation

```
NewAmazonS3Manager()
    │
    ├── Load AWS config:
    │   ├── If static credentials: use StaticCredentialsProvider
    │   └── Else: use LoadDefaultConfig (credentials file, env vars, IAM role)
    │
    ├── Create S3 client options:
    │   ├── If path_style: UsePathStyle = true
    │   └── If endpoint: BaseEndpoint = endpoint
    │
    ├── Create S3 client: s3.NewFromConfig(cfg, options...)
    ├── Create PresignClient: s3.NewPresignClient(client, ...)
    └── Return AmazonS3Manager
```

### Multipart Upload Algorithm

```
UploadMultipartFromReader(ctx, req, partSize)
    │
    ├── CreateMultipartUpload() → get UploadID
    ├── Initialize transfer tracking
    │
    ├── For each part:
    │   ├── Read partSize bytes from req.Body
    │   ├── UploadPart(UploadID, partNumber, data)
    │   ├── Store ETag and size
    │   ├── Update progress: BytesDone, Percent, SpeedMBps
    │   └── partNumber++
    │
    ├── CompleteMultipartUpload(UploadID, parts)
    │
    └── Return S3Object with VersionID
```

### Presigned URL Generation

```
GeneratePresignedURL(ctx, req)
    │
    ├── Switch on req.Method:
    │   ├── "get":
    │   │   └── PresignClient.PresignGet(objectInput, WithPresignExpires)
    │   ├── "put":
    │   │   └── PresignClient.PresignPut(objectInput, WithPresignExpires)
    │   └── "delete":
    │       └── PresignClient.PresignDelete(objectInput, WithPresignExpires)
    │
    ├── Return S3PresignedURLResult with URL and signed headers
    └── Error for unsupported methods
```

### Transfer Progress Tracking

```
Transfer Progress Structure:
  - ID: Transfer identifier (usually key name)
  - Type: "upload" or "download"
  - BytesTotal: Total bytes to transfer
  - BytesDone: Bytes transferred so far
  - Percent: (BytesDone / BytesTotal) * 100
  - SpeedMBps: BytesDone / 1024 / 1024 / duration_seconds
  - Status: "pending", "transferring", "complete", "error"
  - Error: Error message if status = "error"
```

### Bucket Size Calculation

```
GetBucketSize(ctx, bucket)
    │
    ├── ListAllObjects(bucket, "")
    │
    ├── For each object:
    │   └── totalSize += obj.Size
    │
    ├── Return totalSize and len(objects)
    └── Error if list fails
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.