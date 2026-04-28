# Firebase Manager

## Overview

The `FirebaseManager` is a comprehensive Go library for managing Firebase services using the Firebase Admin SDK for Go. It provides Firebase Cloud Messaging (push notifications), Authentication, Firestore database operations, and Cloud Storage operations with async support.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Cloud Messaging**: Send push notifications to devices
- **Authentication**: Verify ID tokens and retrieve user information
- **Firestore**: Get, create, update, delete documents and query collections
- **Cloud Storage**: Upload, download, delete files and generate signed URLs
- **Async Operations**: Async execution for messaging and auth operations via worker pool
- **Authentication**: Uses service account credentials file via Firebase Admin SDK
- **Worker Pool**: Async job execution support

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
    
    // Create Firebase manager (configuration via viper)
    manager, err := firebase.NewFirebaseManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // Send a push notification
    message := firebase.FirebaseMessage{
        Token: "device-fcm-token",
        Title: "Hello",
        Body:  "Welcome to Firebase!",
        Data:  map[string]string{"key": "value"},
    }
    
    response, err := manager.SendPushNotification(ctx, message)
    if err != nil {
        panic(err)
    }
    fmt.Printf("Message sent: %s\n", response)
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `FirebaseManager` | Main manager with Firebase app and service clients |
| `FirebaseMessage` | Push notification with token, title, body, data |
| `FirebaseUser` | Firebase Auth user with UID, email, display name, etc. |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   FirebaseManager                          │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  Firebase   │  → Firebase App (service account)    │
│  │  App        │  → Auth, Firestore, Storage, Messaging│
│  └────────────────┘                                      │
│                                                         │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ AsyncResult │                   │
│  │  (6 workers)│      │   Channel   │                   │
│  └─────────────┘      └──────────────┘                   │
│         ▲                      │                            │
│         │                      ▼                            │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  SubmitJob  │      │ ExecuteAsync │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                         │
│  Service Account: configured via viper                        │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewFirebaseManager()
    │
    ├── Check viper config: "firebase.enabled"
    ├── Get service account path: "firebase.service_account"
    ├── Create option.WithCredentialsFile(serviceAccountPath)
    ├── Initialize Firebase App with context and option
    ├── Initialize Auth client
    ├── Initialize Firestore client
    ├── Initialize Storage client
    ├── Initialize Messaging client
    ├── Create WorkerPool(6)
    ├── Start worker pool
    └── Return FirebaseManager
```

### 2. Send Push Notification Flow

```
SendPushNotification(ctx, message)
    │
    ├── Build messaging.Message with token, notification, data
    ├── Call messagingClient.Send(ctx, msg)
    └── Return response string
```

### 3. Verify ID Token Flow

```
VerifyIDToken(ctx, idToken)
    │
    └── Call authClient.VerifyIDToken(ctx, idToken)
```

### 4. Get User Flow

```
GetUser(ctx, uid)
    │
    ├── Call authClient.GetUser(ctx, uid)
    ├── Convert to FirebaseUser struct
    └── Return FirebaseUser
```

### 5. Firestore Get Document Flow

```
GetDocument(ctx, collection, docID)
    │
    └── Call firestore.Collection(collection).Doc(docID).Get(ctx)
```

### 6. Storage Upload Flow

```
UploadFile(ctx, path, reader, contentType)
    │
    ├── Get default bucket
    ├── Create writer for object at path
    ├── Set ContentType
    ├── Copy reader to writer
    └── Close writer
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `firebase.enabled` | bool | false | Enable/disable Firebase manager |
| `firebase.service_account` | string | "" | Path to Firebase service account JSON file |
| `firebase.storage_bucket` | string | "" | Firebase Storage bucket name (optional) |

### Environment Variables

The Firebase manager uses the Firebase Admin SDK which loads credentials from:
- Service account JSON file (specified in config)
- `GOOGLE_APPLICATION_CREDENTIALS` environment variable
- Firebase configuration in the app (for client SDK)

## Usage Examples

### Push Notification (Cloud Messaging)

```go
ctx := context.Background()

// Send a push notification
message := firebase.FirebaseMessage{
    Token: "device-fcm-token-here",
    Title: "Hello",
    Body:  "This is a test notification",
    Data:  map[string]string{
        "click_action": "FLUTTER_NOTIFICATION_CLICK_ACTION",
        "screen":       "home",
    },
}

response, err := manager.SendPushNotification(ctx, message)
if err != nil {
    panic(err)
}
fmt.Printf("Message sent successfully: %s\n", response)
```

### Authentication Operations

```go
ctx := context.Background()

// Verify an ID token (from client login)
token, err := manager.VerifyIDToken(ctx, "id-token-from-client")
if err != nil {
    panic(err)
}
fmt.Printf("Token verified for UID: %s\n", token.UID)
fmt.Printf("Claims: %v\n", token.Claims)

// Get user by UID
user, err := manager.GetUser(ctx, "user-uid-here")
if err != nil {
    panic(err)
}

fmt.Printf("User: %s\n", user.UID)
fmt.Printf("  Email: %s\n", user.Email)
fmt.Printf("  Display Name: %s\n", user.DisplayName)
fmt.Printf("  Phone: %s\n", user.PhoneNumber)
fmt.Printf("  Disabled: %v\n", user.Disabled)
fmt.Printf("  Email Verified: %v\n", user.EmailVerified)
fmt.Printf("  Created: %v\n", user.CreatedAt)
fmt.Printf("  Last Login: %v\n", user.LastLoginAt)
fmt.Printf("  Custom Claims: %v\n", user.CustomClaims)
```

### Firestore Operations

```go
ctx := context.Background()

// Get a document
doc, err := manager.GetDocument(ctx, "users", "user-123")
if err != nil {
    panic(err)
}

data := doc.Data()
fmt.Printf("Document data: %v\n", data)

// Create a new document
docRef, err := manager.CreateDocument(ctx, "messages", map[string]interface{}{
    "text":     "Hello Firestore!",
    "timestamp": time.Now(),
    "user_id":  "user-123",
})
if err != nil {
    panic(err)
}
fmt.Printf("Created document: %s\n", docRef.ID)

// Update a document
err = manager.UpdateDocument(ctx, "messages", "doc-id-456", []firestore.Update{
    {Path: "text", Value: "Updated text"},
    {Path: "edited", Value: true},
})
if err != nil {
    panic(err)
}
fmt.Println("Document updated")

// Delete a document
err = manager.DeleteDocument(ctx, "messages", "doc-id-456")
if err != nil {
    panic(err)
}
fmt.Println("Document deleted")

// Query collection
query := manager.Firestore.Collection("messages").Where("user_id", "==", "user-123").OrderBy("timestamp", firestore.Desc)
docs, err := manager.QueryCollection(ctx, "messages", query)
if err != nil {
    panic(err)
}

for _, doc := range docs {
    fmt.Printf("Document %s: %v\n", doc.Ref.ID, doc.Data())
}
```

### Storage Operations

```go
ctx := context.Background()

// Upload a file
content := strings.NewReader("Hello, Firebase Storage!")
err := manager.UploadFile(ctx, "uploads/hello.txt", content, "text/plain")
if err != nil {
    panic(err)
}
fmt.Println("File uploaded")

// Download a file
reader, err := manager.DownloadFile(ctx, "uploads/hello.txt")
if err != nil {
    panic(err)
}
defer reader.Close()

data, _ := io.ReadAll(reader)
fmt.Printf("Downloaded: %s\n", string(data))

// Get signed URL (for public access, see note)
url, err := manager.GetSignedURL(ctx, "uploads/hello.txt", 15*time.Minute)
if err != nil {
    panic(err)
}
fmt.Printf("Signed URL: %s\n", url)

// Delete a file
err = manager.DeleteFile(ctx, "uploads/hello.txt")
if err != nil {
    panic(err)
}
fmt.Println("File deleted")
```

### Async Operations

```go
ctx := context.Background()

// Async send push notification
message := firebase.FirebaseMessage{
    Token: "device-token",
    Title: "Async Test",
    Body:  "Sent asynchronously",
}

result := manager.SendPushNotificationAsync(ctx, message)
select {
case response := <-result.Ch:#    fmt.Printf("Sent: %s\n", response)
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async verify ID token
result = manager.VerifyIDTokenAsync(ctx, "id-token")
select {
case token := <-result.Ch:#    fmt.Printf("Verified: %s\n", token.UID)
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}

// Async get user
result = manager.GetUserAsync(ctx, "user-uid")
select {
case user := <-result.Ch:#    fmt.Printf("User: %s\n", user.Email)
case err := <-result.ErrCh:#    fmt.Printf("Error: %v\n", err)
}
```

### Get Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Initialized: %v\n", status["initialized"])

if status["connected"] == false {
    if errMsg, ok := status["error"].(string); ok {
        fmt.Printf("Error: %s\n", errMsg)
    }
}
```

## Dependencies

| Dependency | Role |
|------------|------|
| `firebase.google.com/go` | Firebase Admin SDK for Go |
| `cloud.google.com/go/firestore` | Firestore client |
| `firebase.google.com/go/auth` | Auth client |
| `firebase.google.com/go/messaging` | Cloud Messaging client |
| `firebase.google.com/go/storage` | Cloud Storage client |
| `google.golang.org/api/option` | Google API options |
| `github.com/spf13/viper` | Configuration management |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `context`, `fmt`, `io`, `time` |

## Error Handling

The library uses Go error handling patterns:

- **Initialization failures**: Each service client initialization is checked
- **Firestore errors**: Returned from Firestore operations
- **Auth errors**: Token verification and user lookup errors
- **Storage errors**: Upload/download/delete errors
- **Context cancellation**: All operations respect `context.Context` for timeouts
- **Nil checks**: Public methods handle nil receiver gracefully in `GetStatus()`

Example error handling:

```go
response, err := manager.SendPushNotification(ctx, message)
if err != nil {
    if strings.Contains(err.Error(), "registration-token-not-registered") {
        return fmt.Errorf("device token invalidated: %w", err)
    }
    if strings.Contains(err.Error(), "invalid-argument") {
        return fmt.Errorf("invalid message format: %w", err)
    }
    return fmt.Errorf("failed to send notification: %w", err)
}
```

## Common Pitfalls

### 1. Service Account Not Configured

**Problem**: `Failed to initialize Firebase app` error

**Solution**: 
- Configure service account: `viper.Set("firebase.service_account", "/path/to/service-account.json")`
- Or set environment: `export GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json`
- Download service account from Firebase Console → Project Settings → Service Accounts

### 2. Invalid FCM Token

**Problem**: `registration-token-not-registered` error

**Solution**:
- Ensure device token is valid and not expired
- Get fresh token from client app
- Handle token refresh in client

### 3. Firestore Permissions

**Problem**: `permission-denied` error

**Solution**:
- Check service account has Firestore access
- Verify Firestore security rules
- Check IAM permissions in Google Cloud Console

### 4. Storage Bucket Not Configured

**Problem**: `bucket-not-found` error

**Solution**:
- Configure bucket: `viper.Set("firebase.storage_bucket", "my-project.appspot.com")`
- Or use default bucket from Firebase project

### 5. Context Timeout

**Problem**: Operations timing out

**Solution**:
```go
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
// Operations will use this timeout
```

### 6. Signed URLs

**Problem**: Signed URLs not working

**Solution**:
- Note: The current implementation returns public URLs for public buckets
- For proper signed URLs, you need Google Cloud IAM permissions
- Consider using Firebase Storage security rules instead

## Advanced Usage

### Batch Push Notifications

```go
func sendBatchNotifications(manager *firebase.FirebaseManager, tokens []string, title, body string) {
    ctx := context.Background()
    
    for i, token := range tokens {
        message := firebase.FirebaseMessage{
            Token: token,
            Title: title,
            Body:  body,
            Data:  map[string]string{"batch_id": fmt.Sprintf("%d", i)},
        }
        
        _, err := manager.SendPushNotification(ctx, message)
        if err != nil {
            fmt.Printf("Failed to send to %s: %v\n", token[:10]+"...", err)
            continue
        }
        fmt.Printf("Sent to token %d\n", i)
    }
}
```

### User Management

```go
func manageUser(manager *firebase.FirebaseManager, uid string) {
    ctx := context.Background()
    
    // Get user info
    user, err := manager.GetUser(ctx, uid)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    fmt.Printf("User: %s (%s)\n", user.DisplayName, user.Email)
    fmt.Printf("  Created: %v, Last Login: %v\n", user.CreatedAt, user.LastLoginAt)
    
    if user.CustomClaims != nil {
        fmt.Printf("  Custom Claims: %v\n", user.CustomClaims)
    }
}
```

### Firestore Data Sync

```go
func syncFirestoreData(manager *firebase.FirebaseManager, collection string) {
    ctx := context.Background()
    
    // Query all documents
    query := manager.Firestore.Collection(collection).OrderBy("timestamp", firestore.Asc)
    docs, err := manager.QueryCollection(ctx, collection, query)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    fmt.Printf("Found %d documents in %s\n", len(docs), collection)
    for _, doc := range docs {
        fmt.Printf("  %s: %v\n", doc.Ref.ID, doc.Data())
    }
}
```

### Storage File Management

```go
func manageStorageFiles(manager *firebase.FirebaseManager) {
    ctx := context.Background()
    
    // Upload multiple files
    files := []struct { Path, Content string }{
        {"data/file1.txt", "Content 1"},
        {"data/file2.txt", "Content 2"},
    }
    
    for _, f := range files {
        err := manager.UploadFile(ctx, f.Path, strings.NewReader(f.Content), "text/plain")
        if err != nil {
            fmt.Printf("Upload failed for %s: %v\n", f.Path, err)
            continue
        }
        fmt.Printf("Uploaded: %s\n", f.Path)
    }
}
```

## Internal Algorithms

### Firebase Initialization

```
NewFirebaseManager():
    │
    ├── Load service account credentials
    ├── firebase.NewApp(context, nil, option)
    ├── app.Auth(context) → Auth client
    ├── app.Firestore(context) → Firestore client
    ├── app.Storage(context) → Storage client
    ├── app.Messaging(context) → Messaging client
    └── Return FirebaseManager with all clients
```

### Push Notification Sending

```
SendPushNotification():
    │
    ├── Build messaging.Message with:
    │   ├── Token
    │   ├── Notification (Title, Body)
    │   └── Data (key-value pairs)
    ├── Call messagingClient.Send(ctx, msg)
    └── Return response string
```

### Firestore Document Operations

```
GetDocument():
    └── firestore.Collection(collection).Doc(docID).Get(ctx)
    
CreateDocument():
    └── firestore.Collection(collection).Add(ctx, data)
    
UpdateDocument():
    └── firestore.Collection(collection).Doc(docID).Update(ctx, updates)
    
DeleteDocument():
    └── firestore.Collection(collection).Doc(docID).Delete(ctx)
```

### Storage Operations

```
UploadFile():
    │
    ├── storage.DefaultBucket()
    ├── bucket.Object(path).NewWriter(ctx)
    ├── Set ContentType
    ├── io.Copy(writer, reader)
    └── writer.Close()
    
DownloadFile():
    └── bucket.Object(path).NewReader(ctx)
    
DeleteFile():
    └── bucket.Object(path).Delete(ctx)
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.