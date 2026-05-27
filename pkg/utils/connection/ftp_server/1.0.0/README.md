# Utils: FTP Server

## Overview

The `FTPServer` is a pure Go FTP server for serving files over the FTP protocol — all within the `utils` package. No configuration, no lifecycle hooks, no registration required.

**Package:** `stackyrd/pkg/utils`

**Implementation:** Pure Go using standard library (`net`, `os`, `crypto/tls`)

## Features

- **29 RFC-Compliant Commands**: USER, PASS, QUIT, SYST, PWD, CWD, CDUP, TYPE, PASV, EPSV, PORT, EPRT, LIST, NLST, RETR, STOR, APPE, DELE, RNFR, RNTO, MKD, RMD, SIZE, MDTM, NOOP, ALLO, HELP, STAT, OPTS, FEAT
- **Passive Mode**: PASV (IPv4) and EPSV (IPv6) with configurable port range
- **Active Mode**: PORT (IPv4) and EPRT (IPv6) data connections
- **TLS/FTPS**: Implicit FTPS via `WithFTPServerTLS(*tls.Config)` (TLS-wrapped listener)
- **Authentication**: Custom `AuthFunc` via `SetAuthFunc` (default: allow all)
- **File System Isolation**: Root directory via `WithFTPRootDir` (chroot-like)
- **Graceful Shutdown**: `Stop()` drains active sessions
- **Connection Tracking**: Active connection count via `ActiveConnections()`

## Quick Start

```go
package main

import (
	"log"

	"stackyrd/pkg/utils"
)

func main() {
	server := utils.NewFTPServer(
		utils.WithFTPRootDir("/tmp/ftp_root"),
		utils.WithFTPGreeting("220 Custom FTP Server\r\n"),
	)

	server.SetAuthFunc(func(user, pass string) bool {
		return user == "admin" && pass == "secret"
	})

	if err := server.Start(); err != nil {
		log.Fatal(err)
	}
	defer server.Stop()

	log.Printf("FTP server on :21, root=%s", "/tmp/ftp_root")
	select {}
}
```

## Architecture

```
NewFTPServer(opts...)
    │
    └── Return *FTPServer{config, authFn}

Start()
    │
    ├── net.Listen("tcp", ":21") or tls.Listen
    ├── go acceptLoop()
    │       │
    │       ├── accept() → newFTPSession(conn, server)
    │       ├── go handleSession()
    │       │       ├── read commands from control conn
    │       │       ├── handleCmd() → dispatch to USER/PASS/LIST/RETR/...
    │       │       └── PASV → listen on data port
    │       │           PORT → record client data addr
    │       │           RETR/STOR/LIST → openDataConn() → transfer
    │       └── session ends → cleanup
    └── Return nil

Stop()
    │
    ├── close(shutdown)
    ├── listener.Close()
    ├── close all sessions + data listeners
    └── wg.Wait()
```

## API

### Constructor & Lifecycle

| Method | Description |
|--------|-------------|
| `NewFTPServer(opts...)` | Create server instance |
| `Start()` | Start listening |
| `Stop()` | Graceful shutdown |
| `SetAuthFunc(fn)` | Set authentication handler |
| `ActiveConnections()` | Current connection count |

### Options

| Option | Description |
|--------|-------------|
| `WithFTPGreeting(g)` | Custom greeting message |
| `WithFTPRootDir(dir)` | Root directory for served files |
| `WithFTPServerTLS(cfg)` | Enable implicit FTPS with TLS config |
| `WithFTPPassiveRange(start, end)` | Passive data port range |

## Error Handling

All methods return standard Go errors:

```go
err := server.Start()
if err != nil {
    // err: "ftp server listen on :21: address already in use"
}
```

Common errors:
- `ftp server listen on :21: bind: address already in use`
- `ftp server already started` (calling Start twice)

## Thread Safety

Each client connection runs in its own goroutine. The server is fully thread-safe.

## Examples

### TLS Server

```go
cert, _ := tls.LoadX509KeyPair("server.crt", "server.key")
server := utils.NewFTPServer(
    utils.WithFTPRootDir("/data/ftp"),
    utils.WithFTPServerTLS(&tls.Config{Certificates: []tls.Certificate{cert}}),
)
```

### Custom Auth

```go
server.SetAuthFunc(func(user, pass string) bool {
    return user == "deploy" && pass == os.Getenv("FTP_PASSWORD")
})
```

### Passive Port Range

```go
server := utils.NewFTPServer(
    utils.WithFTPRootDir("/srv/ftp"),
    utils.WithFTPPassiveRange(50000, 50100),
)
```

## Common Pitfalls

1. **Passive Mode Firewalls**: PASV uses random ports; configure `WithFTPPassiveRange(50000, 50100)` and open those ports in your firewall.
2. **Active Mode NAT**: PORT mode won't work behind NAT without ALG or port forwarding. Prefer PASV/EPSV.
3. **TLS vs STARTTLS**: Currently supports implicit TLS only (TLS from connect). For explicit (AUTH TLS), extend the handleCmd switch.
4. **Root Directory**: Ensure the root directory exists; the server creates it on `Start()` if missing.
5. **File Permissions**: The server uses the OS file permissions of the running process. Ensure the process has read/write access to the root directory.

## Supported Commands

| Command | Description |
|---------|-------------|
| USER | Authentication username |
| PASS | Authentication password |
| QUIT | Disconnect |
| SYST | System type (UNIX) |
| FEAT | Supported features list |
| PWD | Print working directory |
| CWD | Change working directory |
| CDUP | Change to parent directory |
| TYPE | Transfer type (A=ASCII, I=Binary) |
| MODE | Transfer mode (Stream) |
| STRU | File structure (File) |
| PASV | Enter passive mode (IPv4) |
| EPSV | Enter passive mode (IPv6) |
| PORT | Active mode address (IPv4) |
| EPRT | Active mode address (IPv6) |
| LIST | List directory contents |
| NLST | List directory names |
| RETR | Download file |
| STOR | Upload file |
| APPE | Append to file |
| DELE | Delete file |
| RNFR | Rename from (source) |
| RNTO | Rename to (destination) |
| MKD | Create directory |
| RMD | Remove directory |
| SIZE | Get file size |
| MDTM | Get file modification time |
| NOOP | No operation (keepalive) |
| ALLO | Allocate space (ignored) |
| HELP | Help information |
| STAT | Server status |
| OPTS | Options |

## Dependencies

| Dependency | Role |
|-----------|------|
| Standard library | `net`, `os`, `crypto/tls`, `sync`, `io`, `path/filepath`, `time` |

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.
