# Kubernetes Manager

## Overview

The `KubernetesManager` is a Go library for managing Kubernetes cluster interactions using the official Kubernetes client-go library. It provides listing and scaling of pods, deployments, services, and nodes, along with custom resource access via the dynamic client, all with async support.

**Import Path:** `stackyrd/pkg/infrastructure`

## Features

- **Pod Management**: List pods in a namespace with status, node, IP, restart count, labels
- **Deployment Management**: List deployments with replica counts, availability, labels
- **Service Management**: List services with type, cluster IP, external IPs, ports, labels
- **Node Management**: List nodes with status, role, IPs, architecture, OS image, capacity
- **Custom Resources**: Get custom resources using dynamic client and GroupVersionResource
- **Scaling**: Scale deployments to specified replica counts
- **Async Operations**: Async listing and scaling via worker pool
- **Worker Pool**: Async job execution support (6 workers)
- **Authentication**: Supports kubeconfig file or in-cluster config
- **Connection Test**: Validates connectivity via server version check

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
    
    // Create Kubernetes manager (configuration via viper)
    manager, err := kubernetes.NewKubernetesManager(log)
    if err != nil {
        panic(err)
    }
    defer manager.Close()
    
    ctx := context.Background()
    
    // List pods in default namespace
    pods, err := manager.ListPods(ctx, "")
    if err != nil {
        panic(err)
    }
    
    for _, pod := range pods {
        fmt.Printf("Pod: %s (Status: %s)\n", pod.Name, pod.Status)
    }
}
```

## Architecture

### Core Structs

| Struct | Description |
|--------|-------------|
| `KubernetesManager` | Main manager with clientset, dynamic client, config |
| `KubernetesPod` | Pod with name, namespace, status, node, IP, restarts, labels |
| `KubernetesDeployment` | Deployment with name, namespace, replicas, availability, labels |
| `KubernetesService` | Service with name, namespace, type, IPs, ports, labels |
| `KubernetesNode` | Node with name, status, role, IPs, architecture, capacity |

### Concurrency Model

```
┌─────────────────────────────────────────────────────────────┐
│                   KubernetesManager                       │
├─────────────────────────────────────────────────────────────┤
│  ┌────────────────┐                                      │
│  │  Clientset  │  → Kubernetes API client              │
│  │  (official)  │  → In-cluster or kubeconfig          │
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
│  Namespace: Default namespace (configurable)             │
│  Kubeconfig: Path to kubeconfig file (optional)       │
└─────────────────────────────────────────────────────────────┘
```

## How It Works

### 1. Initialization Flow

```
NewKubernetesManager(logger)
    │
    ├── Check viper config: "kubernetes.enabled"
    ├── Get kubeconfig: "kubernetes.kubeconfig"
    ├── Get namespace: "kubernetes.namespace"
    │
    ├── If kubeconfig path provided:
    │   └── BuildConfigFromFlags(path)
    │
    ├── Else:
    │   └── rest.InClusterConfig()
    │
    ├── Create Clientset from config
    ├── Create DynamicClient from config
    ├── Test connection: client.ServerVersion()
    ├── Create WorkerPool(6)
    ├── Start worker pool
    └── Return KubernetesManager
```

### 2. List Pods Flow

```
ListPods(ctx, namespace)
    │
    ├── If namespace empty: use default namespace
    ├── client.CoreV1().Pods(namespace).List(ctx, metav1.ListOptions{})
    ├── For each pod in result:
    │   ├── Extract start time, restart count
    │   └── Build KubernetesPod struct
    │
    └── Return []KubernetesPod
```

### 3. List Deployments Flow

```
ListDeployments(ctx, namespace)
    │
    ├── If namespace empty: use default namespace
    ├── client.AppsV1().Deployments(namespace).List(ctx, metav1.ListOptions{})
    ├── For each deployment:
    │   └── Build KubernetesDeployment struct
    │
    └── Return []KubernetesDeployment
```

### 4. List Services Flow

```
ListServices(ctx, namespace)
    │
    ├── If namespace empty: use default namespace
    ├── client.CoreV1().Services(namespace).List(ctx, metav1.ListOptions{})
    ├── For each service:
    │   ├── Extract ports
    │   └── Build KubernetesService struct
    │
    └── Return []KubernetesService
```

### 5. List Nodes Flow

```
ListNodes(ctx)
    │
    ├── client.CoreV1().Nodes().List(ctx, metav1.ListOptions{})
    ├── For each node:
    │   ├── Extract internal/external IPs
    │   ├── Determine role (master/control-plane/worker)
    │   ├── Build capacity map
    │   └── Build KubernetesNode struct
    │
    └── Return []KubernetesNode
```

### 6. Scale Deployment Flow

```
ScaleDeployment(ctx, namespace, name, replicas)
    │
    ├── If namespace empty: use default namespace
    ├── client.AppsV1().Deployments(namespace).GetScale(ctx, name, metav1.GetOptions{})
    ├── Set scale.Spec.Replicas = replicas
    ├── client.AppsV1().Deployments(namespace).UpdateScale(ctx, name, scale, metav1.UpdateOptions{})
    └── Return error
```

## Configuration

### Viper Configuration Options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `kubernetes.enabled` | bool | false | Enable/disable Kubernetes manager |
| `kubernetes.kubeconfig` | string | "" | Path to kubeconfig file |
| `kubernetes.namespace` | string | "" | Default namespace for operations |

### Environment Variables

The Kubernetes manager does not directly use environment variables. Configuration is read through Viper, which can load from:
- Configuration files (JSON, YAML, TOML)
- Environment variables (if Viper is configured for it)

You can set Kubernetes parameters via:
```go
viper.Set("kubernetes.enabled", true)
viper.Set("kubernetes.kubeconfig", "/path/to/kubeconfig")
viper.Set("kubernetes.namespace", "default")
```

For in-cluster config, leave `kubernetes.kubeconfig` empty and ensure the pod has proper service account.

## Usage Examples

### Listing Pods

```go
ctx := context.Background()

// List pods in a namespace
pods, err := manager.ListPods(ctx, "default")
if err != nil {
    panic(err)
}

for _, pod := range pods {
    fmt.Printf("Pod: %s\n", pod.Name)
    fmt.Printf("  Namespace: %s\n", pod.Namespace)
    fmt.Printf("  Status: %s\n", pod.Status)
    fmt.Printf("  Node: %s\n", pod.NodeName)
    fmt.Printf("  IP: %s\n", pod.IP)
    if pod.StartTime != nil {
        fmt.Printf("  Started: %v\n", *pod.StartTime)
    }
    fmt.Printf("  Restarts: %d\n", pod.RestartCount)
    if pod.Labels != nil {
        fmt.Printf("  Labels: %v\n", pod.Labels)
    }
}
```

### Listing Deployments

```go
ctx := context.Background()

// List deployments
deployments, err := manager.ListDeployments(ctx, "")
if err != nil {
    panic(err)
}

for _, deploy := range deployments {
    fmt.Printf("Deployment: %s\n", deploy.Name)
    fmt.Printf("  Replicas: %d, Available: %d, Updated: %d\n", 
        deploy.Replicas, deploy.AvailableReplicas, deploy.UpdatedReplicas)
    if deploy.Labels != nil {
        fmt.Printf("  Labels: %v\n", deploy.Labels)
    }
}
```

### Listing Services

```go
ctx := context.Background()

// List services
services, err := manager.ListServices(ctx, "default")
if err != nil {
    panic(err)
}

for _, svc := range services {
    fmt.Printf("Service: %s\n", svc.Name)
    fmt.Printf("  Type: %s\n", svc.Type)
    fmt.Printf("  Cluster IP: %s\n", svc.ClusterIP)
    if len(svc.ExternalIP) > 0 {
        fmt.Printf("  External IPs: %v\n", svc.ExternalIP)
    }
    if len(svc.Ports) > 0 {
        fmt.Printf("  Ports: %v\n", svc.Ports)
    }
}
```

### Listing Nodes

```go
ctx := context.Background()

// List nodes
nodes, err := manager.ListNodes(ctx)
if err != nil {
    panic(err)
}

for _, node := range nodes {
    fmt.Printf("Node: %s\n", node.Name)
    fmt.Printf("  Status: %s\n", node.Status)
    fmt.Printf("  Role: %s\n", node.Role)
    fmt.Printf("  Internal IP: %s\n", node.InternalIP)
    if node.ExternalIP != "" {
        fmt.Printf("  External IP: %s\n", node.ExternalIP)
    }
    fmt.Printf("  Architecture: %s\n", node.Architecture)
    fmt.Printf("  OS Image: %s\n", node.OSImage)
    fmt.Printf("  Kubelet: %s\n", node.KubeletVersion)
    if node.Capacity != nil {
        fmt.Printf("  Capacity: %v\n", node.Capacity)
    }
}
```

### Scaling a Deployment

```go
ctx := context.Background()

// Scale deployment to 3 replicas
err := manager.ScaleDeployment(ctx, "default", "my-deployment", 3)
if err != nil {
    panic(err)
}
fmt.Println("Deployment scaled!")
```

### Getting Custom Resources

```go
import (
    "k8s.io/apimachinery/pkg/runtime/schema"
)

ctx := context.Background()

// Get a custom resource (e.g., Prometheus rule)
gvr := schema.GroupVersionResource{
    Group:    "monitoring.coreos.com",
    Version:  "v1",
    Resource: "prometheusrules",
}

resource, err := manager.GetResource(ctx, gvr, "default", "my-rule")
if err != nil {
    panic(err)
}
fmt.Printf("Resource: %v\n", resource)
```

### Async Operations

```go
ctx := context.Background()

// Async list pods
result := manager.ListPodsAsync(ctx, "default")
select {
case pods := <-result.Ch:
    fmt.Printf("Found %d pods\n", len(pods))
case err := <-result.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async list deployments
result = manager.ListDeploymentsAsync(ctx, "")
select {
case deployments := <-result.Ch:
    fmt.Printf("Found %d deployments\n", len(deployments))
case err := <-result.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async list services
result = manager.ListServicesAsync(ctx, "default")
select {
case services := <-result.Ch:
    fmt.Printf("Found %d services\n", len(services))
case err := <-result.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async list nodes
result = manager.ListNodesAsync(ctx)
select {
case nodes := <-result.Ch:
    fmt.Printf("Found %d nodes\n", len(nodes))
case err := <-result.ErrCh:
    fmt.Printf("Error: %v\n", err)
}

// Async scale deployment
result = manager.ScaleDeploymentAsync(ctx, "default", "my-deployment", 5)
select {
case <-result.Ch:
    fmt.Println("Scaled asynchronously!")
case err := <-result.ErrCh:
    fmt.Printf("Error: %v\n", err)
}
```

### Get Status

```go
// Get manager status
status := manager.GetStatus()
fmt.Printf("Connected: %v\n", status["connected"])
fmt.Printf("Server Version: %s\n", status["server_version"])
fmt.Printf("Platform: %s\n", status["platform"])
fmt.Printf("Namespace: %s\n", status["namespace"])

if status["connected"] == false {
    if errMsg, ok := status["error"].(string); ok {
        fmt.Printf("Error: %s\n", errMsg)
    }
}
```

## Dependencies

| Dependency | Role |
|------------|------|
| `k8s.io/client-go` | Kubernetes client library |
| `k8s.io/apimachinery` | Kubernetes API machinery |
| `k8s.io/client-go/tools/clientcmd` | Kubeconfig loading |
| `github.com/spf13/viper` | Configuration management |
| `stackyrd/config` | Internal configuration package |
| `stackyrd/pkg/logger` | Structured logging |
| Standard library | `context`, `encoding/json`, `fmt`, `time` |

## Error Handling

The library uses Go error handling patterns:

- **Connection failures**: `testConnection()` validates via `ServerVersion()`
- **API errors**: Kubernetes API errors returned from list/get operations
- **Context cancellation**: All operations respect `context.Context` for timeouts
- **Nil checks**: Public methods handle nil receiver gracefully in `GetStatus()`

Example error handling:

```go
pods, err := manager.ListPods(ctx, namespace)
if err != nil {
    if strings.Contains(err.Error(), "forbidden") {
        return fmt.Errorf("permission denied: %w", err)
    }
    if strings.Contains(err.Error(), "not found") {
        return fmt.Errorf("namespace not found: %w", err)
    }
    return fmt.Errorf("failed to list pods: %w", err)
}
```

## Common Pitfalls

### 1. Kubeconfig Not Found

**Problem**: `failed to load kubeconfig` error

**Solution**: 
- Configure path: `viper.Set("kubernetes.kubeconfig", "/path/to/kubeconfig")`
- Or use in-cluster config by leaving kubeconfig empty
- Ensure the file exists and is readable

### 2. In-Cluster Config Fails

**Problem**: `failed to get in-cluster config` error

**Solution**:
- When running inside a pod, ensure service account is properly configured
- Check that `/var/run/secrets/kubernetes.io/serviceaccount/token` exists
- For local development, use kubeconfig instead

### 3. Namespace Not Found

**Problem**: `namespaces "xxx" not found` error

**Solution**:
- Verify namespace exists: `kubectl get namespace`
- Use empty string for default namespace
- Check spelling and case

### 4. Permission Denied

**Problem**: `forbidden` or `unauthorized` errors

**Solution**:
- Check RBAC permissions for the service account
- Ensure role/clusterrole allows the operation (get, list, update)
- For scaling: need `update` permission on deployments/scale subresource

### 5. Context Timeout

**Problem**: Operations timing out

**Solution**:
```go
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()
```

### 6. Custom Resource Errors

**Problem**: `resource type not found` or `group not found`

**Solution**:
- Verify GVR (GroupVersionResource) is correct
- Check that the CRD is installed
- Use `kubectl api-resources` to list available resources

## Advanced Usage

### Monitoring Pods

```go
func monitorPods(manager *kubernetes.KubernetesManager, namespace string) {
    ticker := time.NewTicker(10 * time.Second)
    defer ticker.Stop()
    
    for {
        select {
        case <-ticker.C:
            ctx := context.Background()
            pods, err := manager.ListPods(ctx, namespace)
            if err != nil {
                fmt.Printf("Error listing pods: %v\n", err)
                continue
            }
            
            fmt.Printf("\n=== Pods in %s (%d) ===\n", namespace, len(pods))
            for _, pod := range pods {
                fmt.Printf("  %s: %s (Restarts: %d)\n", 
                    pod.Name, pod.Status, pod.RestartCount)
            }
        }
    }
}
```

### Watching Deployment Replicas

```go
func watchDeploymentScale(manager *kubernetes.KubernetesManager, namespace, name string) {
    ctx := context.Background()
    
    for {
        deploy, err := manager.ListDeployments(ctx, namespace)
        if err != nil {
            fmt.Printf("Error: %v\n", err)
            time.Sleep(10 * time.Second)
            continue
        }
        
        for _, d := range deploy {
            if d.Name == name {
                fmt.Printf("Deployment %s: %d/%d replicas available\n", 
                    d.Name, d.AvailableReplicas, d.Replicas)
                break
            }
        }
        
        time.Sleep(15 * time.Second)
    }
}
```

### Node Health Check

```go
func checkNodeHealth(manager *kubernetes.KubernetesManager) {
    ctx := context.Background()
    
    nodes, err := manager.ListNodes(ctx)
    if err != nil {
        fmt.Printf("Error: %v\n", err)
        return
    }
    
    for _, node := range nodes {
        if node.Status != "Ready" {
            fmt.Printf("WARNING: Node %s is not ready (Status: %s)\n", 
                node.Name, node.Status)
        }
        
        // Check capacity
        if cap, ok := node.Capacity["pods"]; ok {
            fmt.Printf("Node %s: %s pods capacity\n", node.Name, cap)
        }
    }
}
```

### Custom Resource Controller

```go
func watchCustomResources(manager *kubernetes.KubernetesManager, gvr schema.GroupVersionResource, namespace string) {
    ctx := context.Background()
    
    for {
        resource, err := manager.GetResource(ctx, gvr, namespace, "my-resource")
        if err != nil {
            fmt.Printf("Error getting resource: %v\n", err)
            time.Sleep(30 * time.Second)
            continue
        }
        
        fmt.Printf("Resource: %v\n", resource)
        // Process resource...
        
        time.Sleep(30 * time.Second)
    }
}
```

## Internal Algorithms

### Client Initialization

```
NewKubernetesManager():
    │
    ├── If kubeconfig path:
    │   └── BuildConfigFromFlags(path)
    │
    ├── Else:
    │   └── rest.InClusterConfig()
    │
    ├── kubernetes.NewForConfig(config) → Clientset
    ├── dynamic.NewForConfig(config) → DynamicClient
    ├── client.ServerVersion() → test connection
    ├── NewWorkerPool(6)
    ├── pool.Start()
    └── Return manager
```

### Pod Listing

```
ListPods():
    │
    ├── Pods(namespace).List(ctx, options)
    ├── For each pod:
    │   ├── Get start time from pod.Status.StartTime
    │   ├── Sum restart counts from container statuses
    │   └── Build KubernetesPod
    │
    └── Return pods
```

### Node Role Detection

```
ListNodes():
    │
    ├── For each node:
    │   ├── Check labels for role:
    │   │   ├── "node-role.kubernetes.io/master" → "master"
    │   │   ├── "node-role.kubernetes.io/control-plane" → "control-plane"
    │   │   └── Else → "worker"
    │   │
    │   ├── Extract internal/external IPs from addresses
    │   ├── Build capacity map from node.Status.Capacity
    │   └── Build KubernetesNode
    │
    └── Return nodes
```

### Async Operation Pattern

```
ListPodsAsync(ctx, namespace):
    │
    └── ExecuteAsync(ctx, func):
        └── return ListPods(ctx, namespace)
            └── Returns ([]KubernetesPod, error)
```

## License

This code is part of the Stackyrd project. See the main project LICENSE file for details.