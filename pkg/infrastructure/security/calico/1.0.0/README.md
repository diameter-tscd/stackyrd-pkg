# Calico CNI Networking Manager#

## Overview#

The `CalicoManager` is a Go library for managing Calico CNI (Container Network Interface) networking. It provides IP pool management, network policy configuration, BGP peer management, workload/host endpoint management, network sets, IPAM blocks, and node status monitoring. The library uses the Calico API with Bearer token authentication and includes retry logic for reliability.

**Import Path:** `stackyrd/pkg/infrastructure/security/calico`

## Features#

- **IP Pool Management**: List, create, update, delete Calico IP pools#
- **Network Policies**: Manage namespace network policies (ingress/egress rules)#
- **Global Network Policies**: Manage cluster-wide network policies#
- **BGP Peers**: Configure BGP peer connections#
- **Workload Endpoints**: List and manage workload endpoints#
- **Host Endpoints**: Manage host endpoints#
- **Network Sets**: Manage global network sets for policy matching#
- **IPAM Blocks**: View IP address management blocks#
- **Node Status**: Monitor Calico node status#
- **Retry Logic**: HTTP client with retry support (3 retries)#
- **Worker Pool**: Async job execution support (6 workers)#
- **Status Monitoring**: Get connection status and API health#

## Quick Start#

```go
package main

import (
    "context"
    "fmt"
    "stackyrd/pkg/infrastructure/security/calico"
    "stackyrd/pkg/logger"
)

func main() {
    // Initialize logger
    log := logger.NewLogger()
    
    // Create Calico manager (configuration via viper)
    manager, err := calico.NewCalicoManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // List IP pools
    pools, err := manager.GetIPPools(ctx)
    if err != nil {
        panic(err)
    }
    fmt.Printf("Found %d IP pools\n", len(pools))
    for _, pool := range pools {
        fmt.Printf("  %s: CIDR=%s, NAT=%v\n", pool.Name, pool.CIDR, pool.NATOutgoing)
    }
}
```

## Architecture#

### Core Structs#

| Struct | Description |
|--------|-------------|
| `CalicoManager` | Main manager with HTTP client, API URL, worker pool |
| `CalicoIPPool` | IP pool with CIDR, NAT, block size, node selector |
| `CalicoNetworkPolicy` | Network policy with selector, ingress/egress rules |
| `CalicoGlobalNetworkPolicy` | Global network policy (applies to all namespaces) |
| `CalicoRule` | Policy rule with action, protocol, source/destination |
| `CalicoEntity` | Source/destination entity in a rule |
| `CalicoBGPPeer` | BGP peer configuration |
| `CalicoWorkloadEndpoint` | Workload endpoint (pod, interface, IPs) |
| `CalicoHostEndpoint` | Host endpoint (node, interface, expected IPs) |
| `CalicoNetworkSet` | Global network set (list of CIDRs) |
| `CalicoIPAMBlock` | IPAM block with CIDR, node, capacity |
| `CalicoIPReservation` | IP reservation |
| `CalicoNodeStatus` | Node status (BGP, routes, peers, endpoints) |

### Concurrency Model#

```
┌─────────────────────────────────────────────────────┐
│                   CalicoManager                          │
├─────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  retryablehttp │  → API requests with retry               │
│  │  .Client      │  → Bearer token authentication           │
│  └────────────────┘                                      │
│                                                          │
│  ┌─────────────┐      ┌──────────────┐                   │
│  │  WorkerPool │◄─────│ AsyncResult │                   │
│  │  (6 workers)│      │   Channel   │                   │
│  └─────────────┘      └──────────────┘                   │
│         ▲                      │                            │
│         │                      ▼                            │
│  ┌─────────────┐      ┌──────────────┐                  │
│  │  SubmitJob  │      │ ExecuteAsync │                  │
│  └─────────────┘      └──────────────┘                  │
│                                                          │
│  API URL: http://calico-api:9099/api/v1          │
│  APIKey: Bearer token for authentication            │
└─────────────────────────────────────────────────────┘
```

## How It Works#

### 1. Initialization Flow#

```
NewCalicoManager(logger)
    │
    ├── Check viper config: "calico.enabled"
    ├── Get api_url: "calico.api_url"
    ├── Get api_key: "calico.api_key"
    ├── Get kubeconfig: "calico.kubeconfig"
    ├── Get timeout_sec: "calico.timeout_sec" (default 30s)
    │
    ├── Create retryablehttp.Client:
    │   ├── RetryMax = 3
    │   ├── RetryWaitMin = 1s
    │   ├── RetryWaitMax = 10s
    │   └── HTTPClient.Timeout = timeout
    │
    ├── Test connection: testConnection()
    │   └── GET /api/v1/nodes
    │
    ├── Create WorkerPool(6)
    ├── Start worker pool
    └── Return CalicoManager
```

### 2. Get IP Pools Flow#

```
GetIPPools(ctx)
    │
    ├── GET /api/v1/ippools
    ├── Set auth header (Bearer token)
    ├── Execute with retryablehttp.Client
    ├── Check status code (200 OK)
    ├── Decode JSON into []CalicoIPPool
    └── Return []CalicoIPPool
```

### 3. Create IP Pool Flow#

```
CreateIPPool(ctx, pool)
    │
    ├── Marshal pool to JSON
    ├── POST /api/v1/ippools
    ├── Set auth header
    ├── Execute with retryablehttp.Client
    ├── Check status code (200/201)
    ├── Decode JSON into CalicoIPPool
    └── Return *CalicoIPPool
```

## Configuration#

### Viper Configuration Options#

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `calico.enabled` | bool | false | Enable/disable Calico manager |
| `calico.api_url` | string | "" | Calico API URL (e.g., "http://calico-api:9099") |
| `calico.api_key` | string | "" | API key (Bearer token) |
| `calico.kubeconfig` | string | "" | Kubernetes config path (optional) |
| `calico.timeout_sec` | int | 30 | HTTP client timeout in seconds |

## Usage Examples#

### List IP Pools#

```go
ctx := context.Background()

pools, err := manager.GetIPPools(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("IP Pools: %d\n", len(pools))
for _, pool := range pools {
    fmt.Printf("  %s: CIDR=%s, NAT=%v, Disabled=%v\n", 
        pool.Name, pool.CIDR, pool.NATOutgoing, pool.Disabled)
}
```

### Create IP Pool#

```go
ctx := context.Background()

pool := calico.CalicoIPPool{
    Name:        "new-pool",
    CIDR:        "10.244.2.0/24",
    NATOutgoing:  true,
    NodeSelector: "all()",
    Labels:       map[string]string{"env": "prod"},
}
result, err := manager.CreateIPPool(ctx, pool)
if err != nil {
    panic(err)
}
fmt.Printf("Created pool: %s\n", result.Name)
```

### Update IP Pool#

```go
ctx := context.Background()

pool := calico.CalicoIPPool{
    Name:        "existing-pool",
    CIDR:        "10.244.3.0/24",
    NATOutgoing:  false,
}
result, err := manager.UpdateIPPool(ctx, pool)
if err != nil {
    panic(err)
}
fmt.Printf("Updated pool: %s\n", result.Name)
```

### Delete IP Pool#

```go
ctx := context.Background()

err := manager.DeleteIPPool(ctx, "pool-name")
if err != nil {
    panic(err)
}
fmt.Println("Pool deleted")
```

### List Network Policies#

```go
ctx := context.Background()

policies, err := manager.GetNetworkPolicies(ctx, "default")
if err != nil {
    panic(err)
}

fmt.Printf("Network Policies: %d\n", len(policies))
for _, policy := range policies {
    fmt.Printf("  %s (Selector: %s)\n", policy.Name, policy.Selector)
}
```

### Create Network Policy#

```go
ctx := context.Background()

policy := calico.CalicoNetworkPolicy{
    Name:      "allow-web",
    Namespace: "default",
    Types:     []string{"ingress", "egress"},
    Selector:  "app == 'web'",
    Ingress: []calico.CalicoRule{
        {
            Action: "allow",
            Protocol: "tcp",
            Source: &calico.CalicoEntity{
                Nets: []string{"10.244.0.0/16"},
            },
        },
    },
}
result, err := manager.CreateNetworkPolicy(ctx, policy)
if err != nil {
    panic(err)
}
fmt.Printf("Created policy: %s\n", result.Name)
```

### List BGP Peers#

```go
ctx := context.Background()

peers, err := manager.GetBGPPeers(ctx)
if err != nil {
    panic(err)
}

fmt.Printf("BGP Peers: %d\n", len(peers))
for _, peer := range peers {
    fmt.Printf("  %s: IP=%s, AS=%d\n", peer.Name, peer.PeerIP, peer.ASNumber)
}
```

### Get Workload Endpoints#

```go
ctx := context.Background()

endpoints, err := manager.GetWorkloadEndpoints(ctx, "default")
if err != nil {
    panic(err)
}

fmt.Printf("Workload Endpoints: %d\n", len(endpoints))
for _, ep := range endpoints {
    fmt.Printf("  %s: Pod=%s, Node=%s, IPs=%v\n", 
        ep.Name, ep.Pod, ep.Node, ep.IPs)
}
```

### Get Node Status#

```go
ctx := context.Background()

status, err := manager.GetNodeStatus(ctx, "node-1")
if err != nil {
    panic(err)
}

fmt.Printf("Node: %s\n", status.Name)
fmt.Printf("BGP Status: %s\n", status.BGPStatus)
fmt.Printf("Routes: %d, Peers: %d, Endpoints: %d\n", 
    status.Routes, status.Peers, status.Endpoints)
```

### Async Operations#

```go
ctx := context.Background()

// Async get IP pools
poolsChan := manager.GetIPPoolsAsync(ctx)
pools, err := poolsChan.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("IP Pools: %d\n", len(pools))

// Async get network policies
policiesChan := manager.GetNetworkPoliciesAsync(ctx, "default")
policies, err := policiesChan.Get(ctx)
if err != nil {
    panic(err)
}
fmt.Printf("Policies: %d\n", len(policies))
```

### Getting Status#

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("API URL: %s\n", status["api_url"])
fmt.Printf("IP Pools: %v\n", status["ip_pools"])

if status["connected"] == false {
    if errMsg, ok := status["error"].(string); ok {
        fmt.Printf("Error: %s\n", errMsg)
    }
}
```

## Dependencies#

| Dependency | Role |
|------------|------|
| `github.com/hashicorp/go-retryablehttp` | HTTP client with retry logic |
| `github.com/spf13/viper` | Configuration management |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `encoding/json`, `net/http`, `bytes`, `io`, `time` |

## Error Handling#

The library uses Go error handling patterns:

- **Authentication failures**: Invalid API key#
- **API errors**: Returned from Calico API#
- **Nil checks**: Public methods check manager state#

Example error handling:

```go
pools, err := manager.GetIPPools(ctx)
if err != nil {
    if strings.Contains(err.Error(), "401") {
        return fmt.Errorf("invalid API key: %w", err)
    }
    if strings.Contains(err.Error(), "403") {
        return fmt.Errorf("access forbidden: %w", err)
    }
    return fmt.Errorf("API request failed: %w", err)
}
```

## Common Pitfalls#

### 1. Manager Not Enabled#

**Problem**: `calico.enabled` is false#

**Solution**:
- Set `calico.enabled = true` in configuration#
- Check viper configuration is loaded#

### 2. API URL Missing#

**Problem**: `calico.api_url` not set#

**Solution**:
- Set `calico.api_url` to Calico API URL (e.g., "http://calico-api:9099")#
- Typically Calico API runs on port 9099#

### 3. API Key Missing#

**Problem**: `calico.api_key` not set#

**Solution**:
- Set `calico.api_key` to your Calico API key#
- Use Bearer token authentication#

### 4. Connection Failed#

**Problem**: Cannot connect to Calico API#

**Solution**:
- Verify `calico.api_url` is correct#
- Ensure network can reach the Calico API#
- Check if Calico is running in the cluster#

### 5. Worker Pool Not Available#

**Problem**: Async operations falling back to direct execution#

**Solution**:
- The pool is created during initialization#
- Check that `NewCalicoManager()` succeeded#
- Verify `calico.enabled` is true#

## Advanced Usage#

### Health Check#

```go
func healthCheck(manager *calico.CalicoManager) error {
    ctx := context.Background()
    pools, err := manager.GetIPPools(ctx)
    if err != nil {
        return fmt.Errorf("not connected to Calico: %w", err)
    }
    fmt.Printf("Calico is healthy (%d IP pools)\n", len(pools))
    return nil
}
```

### Batch Policy Check#

```go
func checkAllPolicies(manager *calico.CalicoManager) {
    ctx := context.Background()
    namespaces := []string{"default", "kube-system", "app-namespace"}
    for _, ns := range namespaces {
        policies, err := manager.GetNetworkPolicies(ctx, ns)
        if err != nil {
            fmt.Printf("Error getting policies for %s: %v\n", ns, err)
            continue
        }
        fmt.Printf("Namespace %s: %d policies\n", ns, len(policies))
    }
}
```

## Internal Algorithms#

### API Request Flow#

```
GetIPPools():
    │
    ├── GET /api/v1/ippools
    ├── Set Authorization header (Bearer token)
    ├── Execute with retryablehttp (3 retries)
    ├── Parse response JSON
    └── Return IP pool list
```

### Connection Test#

```
testConnection():
    │
    ├── GET /api/v1/nodes
    ├── Set Authorization header
    ├── Check status code (200 OK)
    └── Return error if failed
```

## API Endpoints#

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/nodes` | GET | List nodes (health check) |
| `/api/v1/ippools` | GET/POST | List/create IP pools |
| `/api/v1/ippools/{name}` | PUT/DELETE | Update/delete IP pool |
| `/api/v1/namespaces/{ns}/networkpolicies` | GET/POST | List/create network policies |
| `/api/v1/namespaces/{ns}/networkpolicies/{name}` | DELETE | Delete network policy |
| `/api/v1/globalnetworkpolicies` | GET/POST | List/create global policies |
| `/api/v1/globalnetworkpolicies/{name}` | DELETE | Delete global policy |
| `/api/v1/bgppeers` | GET/POST | List/create BGP peers |
| `/api/v1/bgppeers/{name}` | DELETE | Delete BGP peer |
| `/api/v1/workloadendpoints` | GET | List workload endpoints |
| `/api/v1/namespaces/{ns}/workloadendpoints` | GET | List workload endpoints in namespace |
| `/api/v1/hostendpoints` | GET/POST | List/create host endpoints |
| `/api/v1/hostendpoints/{name}` | DELETE | Delete host endpoint |
| `/api/v1/globalnetworksets` | GET/POST | List/create network sets |
| `/api/v1/globalnetworksets/{name}` | DELETE | Delete network set |
| `/api/v1/ipamblocks` | GET | List IPAM blocks |
| `/api/v1/nodes/{name}` | GET | Get node status |

## License#

This code is part of the Stackyrd project. See the main project LICENSE file for details.#