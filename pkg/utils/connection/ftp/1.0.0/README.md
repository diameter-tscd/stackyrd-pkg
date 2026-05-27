# Utils: FTP Client

## Overview

The `FTPClient` is a comprehensive Go FTP client for file transfer operations, built on `github.com/jlaffaye/ftp`. No configuration, no lifecycle hooks, no registration required.

**Package:** `stackyrd/pkg/utils`

**Driver:** `github.com/jlaffaye/ftp` (robust, well-maintained FTP client library)

## Features

- **Rich File Operations**: Upload, Download, Delete, Rename, FileSize, FileModTime
- **Directory Operations**: List, ListNames, MakeDir, RemoveDir, ChangeDir, CurrentDir
- **Recursive Transfers**: UploadDirectory, DownloadDirectory (full tree walk)
- **Directory Sync**: SyncToRemote with optional extraneous file deletion
- **Context Support**: UploadContext, DownloadContext with `context.Context` for cancellation
- **Progress Callbacks**: Real-time transfer progress via `ProgressCallback`
- **TLS/FTPS**: Explicit FTPS via `WithFTPTLS(*tls.Config)`
- **Auto-Reconnect**: `WithFTPRetries(n)` for automatic reconnect and retry
- **Functional Options**: `WithFTPTimeout`, `WithFTPPassive`, `WithFTPTLS`, `WithFTPRetries`
- **Thread Safety**: All operations protected by mutex

## Quick Start

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"stackyrd/pkg/utils"
)

func main() {
	client, err := utils.NewFTPClient("ftp.example.com", 21, "user", "pass",
		utils.WithFTPTimeout(10*time.Second),
		utils.WithFTPPassive(true),
	)
	if err != nil {
		log.Fatal(err)
	}
	defer client.Disconnect()

	if err := client.Upload("local.txt", "/remote/path/file.txt"); err != nil {
		log.Fatal(err)
	}

	files, _ := client.List("/remote/path")
	for _, f := range files {
		fmt.Printf("%s (%d bytes, dir=%v)\n", f.Name, f.Size, f.IsDir)
	}

	err = client.DownloadContext(context.Background(), "/remote/file.zip", "local.zip",
		func(current, total int64) {
			fmt.Printf("\rProgress: %d/%d bytes", current, total)
		},
	)
}
```

## Architecture

```
NewFTPClient(host, port, user, pass, opts...)
    │
    ├── DefaultFTPClientConfig() + apply opts
    ├── ftp.Dial(addr, dialOpts...)     // plain or TLS
    ├── conn.Login(username, password)
    └── Return *FTPClient

Upload/Download/List/Delete/...
    │
    ├── withRetry(fn)                   // optional reconnect on failure
    │   ├── c.mu.Lock()
    │   ├── c.conn.Stor/Retr/List/...
    │   └── c.mu.Unlock()
    └── return result/error

UploadDirectory(localDir, remoteDir)
    │
    ├── filepath.Walk(localDir)
    │   ├── dirs  → MakeDir(remotePath)
    │   └── files → Upload(localPath, remotePath)
    └── return first error
```

## API

### Constructor & Connection

| Method | Description |
|--------|-------------|
| `NewFTPClient(host, port, user, pass, opts...)` | Create and connect |
| `Connect()` | (Re)establish connection |
| `Disconnect()` | Close connection |

### Options

| Option | Description |
|--------|-------------|
| `WithFTPTimeout(d)` | Connection timeout (default 30s) |
| `WithFTPPassive(enabled)` | Enable/disable passive mode |
| `WithFTPTLS(cfg)` | Enable explicit FTPS |
| `WithFTPRetries(n)` | Auto-reconnect attempts on failure |

### File Transfers

| Method | Description |
|--------|-------------|
| `Upload(localPath, remotePath)` | Upload file |
| `Download(remotePath, localPath)` | Download file |
| `UploadContext(ctx, localPath, remotePath, progress)` | Upload with context + progress |
| `DownloadContext(ctx, remotePath, localPath, progress)` | Download with context + progress |
| `UploadDirectory(localDir, remoteDir)` | Recursive upload |
| `DownloadDirectory(remoteDir, localDir)` | Recursive download |
| `SyncToRemote(localDir, remoteDir, deleteExtra)` | Sync local → remote with optional cleanup |

### Directory Operations

| Method | Description |
|--------|-------------|
| `List(path)` | List entries with metadata |
| `ListNames(path)` | List entry names only |
| `MakeDir(path)` | Create directory |
| `RemoveDir(path)` | Remove directory |
| `ChangeDir(path)` | Change working directory |
| `CurrentDir()` | Get current working directory |

### File Operations

| Method | Description |
|--------|-------------|
| `Delete(path)` | Delete file |
| `Rename(from, to)` | Rename file/dir |
| `FileSize(path)` | Get file size |
| `FileModTime(path)` | Get modification time |

## Error Handling

All operations return standard Go errors wrapped with context:

```go
err := client.Upload("x.txt", "/remote/x.txt")
if err != nil {
    // err: "ftp stor /remote/x.txt: <underlying error>"
}
```

Common error patterns:
- `ftp connect to host:21: connection refused`
- `ftp login to host:21: 530 Login incorrect`
- `ftp stor /path/file.txt: 550 Permission denied`
- `ftp list /path: 550 Directory not found`

## Thread Safety

All public methods are safe for concurrent use (mutex-locked around the underlying connection). Multiple goroutines can share a single `*FTPClient` instance.

## Examples

### FTPS with TLS

```go
import "crypto/tls"

client, _ := utils.NewFTPClient("ftps.example.com", 21, "user", "pass",
    utils.WithFTPTLS(&tls.Config{
        InsecureSkipVerify: true,
    }),
)
```

### Sync with Progress

```go
client.SyncToRemote("./build", "/www", true)
```

### Recursive Download

```go
client.DownloadDirectory("/remote/data", "./local_data")
```

## Common Pitfalls

1. **Reconnect on Idle**: FTP servers may disconnect idle clients. Use `WithFTPRetries(2)` to auto-reconnect.
2. **Passive vs Active**: Prefer passive mode (`WithFTPPassive(true)`) behind NAT or firewalls.
3. **TLS/FTPS**: The library uses explicit FTPS (AUTH TLS). Ensure the server supports it.

## Dependencies

| Dependency | Role |
|-----------|------|
| `github.com/jlaffaye/ftp` | FTP client library |
| Standard library | `net`, `crypto/tls`, `sync`, `io`, `path/filepath`, `context` |

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.
