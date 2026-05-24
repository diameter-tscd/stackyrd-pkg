# Discord Integration Manager

## Overview

The `DiscordManager` provides a Go client for sending messages, managing webhooks, and interacting with Discord guilds via the Discord REST API. It is used throughout the Stackyrd project to post alerts, notifications, and command responses.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Send Message** : Post plain text or embed messages to a channel.
- **Webhook Support** : Create, update, and delete webhooks.
- **Guild Management** : List guilds, channels, and members.
- **Rate-Limit Handling** : Automatic back-off based on Discord's rate-limit headers.
- **Retry Logic** : Configurable retries for transient failures.
- **Async Operations** : Optional worker pool for concurrent API calls.

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
    log := logger.NewLogger()
    dm, err := discord.NewDiscordManager(log)
    if err != nil { panic(err) }
    defer dm.Close()

    ctx := context.Background()
    err = dm.SendMessage(ctx, "123456789012345678", "Hello from Stackyrd!")
    if err != nil { panic(err) }
    fmt.Println("Message sent")
}
```

## Configuration (Viper)

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `discord.enabled` | bool | false | Enable the manager |
| `discord.token` | string | "" | Bot token (prefixed with `Bot `) |
| `discord.retry_max` | int | 3 | Max retry attempts |

## Dependencies

| Dependency | Role |
|------------|------|
| `github.com/hashicorp/go-retryablehttp` | HTTP client with retries |
| `github.com/spf13/viper` | Configuration |
| `stackyrd/pkg/logger` | Structured logging |

## License

Part of the Stackyrd project. See the top-level LICENSE file.
