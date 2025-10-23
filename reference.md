# Gurobi AutoScaler Operator Documentation

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Installation](#installation)
4. [Configuration](#configuration)
5. [API Reference](#api-reference)
6. [Examples](#examples)
7. [Scaling Behavior](#scaling-behavior)
8. [Troubleshooting](#troubleshooting)
9. [Best Practices](#best-practices)
10. [Monitoring](#monitoring)

## Overview

The Gurobi AutoScaler Operator is a Kubernetes operator that provides intelligent auto-scaling capabilities for Gurobi Compute Server deployments within a Gurobi cluster. It monitors optimization job queue metrics and compute node utilization through the Gurobi Cluster Manager API and automatically scales compute nodes up or down based on configurable rules tailored for mathematical optimization workloads.

### Key Features

- **Optimization-Aware Scaling**: Scales based on optimization job queue metrics, solver waiting times, and compute node utilization
- **Graceful Job Handling**: Properly drains nodes before termination to avoid interrupting long-running optimization jobs
- **Timeout Management**: Configurable timeouts with fallback mechanisms for complex optimization problems
- **Gurobi License Validation**: Prevents scaling when Gurobi license issues are detected to ensure solver availability
- **Node Group Support**: Manages different node groups within the same Gurobi cluster for workload segmentation
- **Secure Cluster Access**: Supports both Kubernetes secrets and direct credential configuration for Gurobi Manager API
- **Solver-Optimized Recovery**: Robust error handling with exponential backoff designed for optimization workloads

### How It Works

1. **Cluster Monitoring**: The operator continuously monitors Gurobi cluster metrics via the Cluster Manager API
2. **Optimization-Based Decisions**: Evaluates scaling rules based on job queue size, solver waiting times, and compute node states
3. **Intelligent Scaling Actions**: Performs scale up/down operations while respecting min/max replica bounds and optimization job requirements
4. **Solver-Safe Operations**: During scale down, properly stops node processing and waits for optimization job completion

### Limitations

- **Workers Scaling Not Supported**: The AutoScaler operator only supports scaling of Gurobi compute nodes. Worker scaling is not currently supported. All scaling operations target compute node deployments that run optimization solvers.

## Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   AutoScaler    │    │ Gurobi Cluster  │    │ Gurobi Compute  │
│   Controller    │◄──►│    Manager      │◄──►│     Nodes       │
│                 │    │      API        │    │   (Solvers)     │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Kubernetes    │    │    MongoDB      │    │ Optimization    │
│   Deployment    │    │   (Cluster      │    │ Job Queue &     │
│                 │    │   Persistence)  │    │   Metrics       │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

### Components

- **AutoScaler Controller**: Kubernetes controller that reconciles AutoScaler resources and manages Gurobi compute node scaling
- **Gurobi Cluster Manager**: Central coordinator providing cluster metrics API, optimization job management, and node registration
- **Gurobi Compute Nodes**: Solver nodes that process mathematical optimization jobs using Gurobi optimizer
- **MongoDB**: Persistence layer for Cluster Manager state, optimization job history, and node group configurations
- **Optimization Job Queue**: Queue management system for distributing optimization problems across available compute nodes

## Installation

### Prerequisites

- Kubernetes cluster (v1.19+) with sufficient resources for optimization workloads
- Valid Gurobi license file
- Gurobi Cluster Manager deployment (for coordinating compute nodes)
- Docker registry access (for pulling Gurobi container images)
- MongoDB instance (for Cluster Manager persistence)
- Network connectivity between AutoScaler, Cluster Manager, and compute nodes

### Quick Start

1. **Install the operator:**
   ```bash
   kubectl apply -f dist/install.yaml
   ```

2. **Prepare your Gurobi license:**
   ```bash
   # Encode your license file
   cat gurobi.lic | base64 -w 0
   ```

3. **Create your configuration:**
   ```bash
   # Copy and modify the example configuration
   cp dev.yaml my-autoscaler.yaml
   # Edit the license and credentials in my-autoscaler.yaml
   ```

4. **Deploy the Gurobi cluster with AutoScaler:**
   ```bash
   kubectl apply -f my-autoscaler.yaml
   ```

### Detailed Installation Steps

#### 1. Operator Installation

Install the AutoScaler operator in your Kubernetes cluster with the install.yml file provided:

```bash
kubectl apply -f install.yaml
```

This creates:
- Custom Resource Definitions (CRDs)
- RBAC permissions
- Operator deployment in `gurobi-manager-operator-system` namespace

#### 2. Gurobi Cluster Manager Credentials Setup

Create credentials for Gurobi Cluster Manager API access:

```bash
# Create namespace if it doesn't exist
kubectl create namespace gurobi-manager-operator-system

# Create credentials secret for Cluster Manager API authentication
kubectl create secret generic autoscaler-credentials \
  --from-literal=accessId="your-cluster-manager-access-id" \
  --from-literal=secret="your-cluster-manager-secret-key" \
  -n gurobi-manager-operator-system
```

#### 3. License Configuration

Create a secret with your Gurobi license:

```bash
kubectl create secret generic gurobi-lic \
  --from-file=gurobi.lic=path/to/your/gurobi.lic \
  -n default
```

## Configuration

### AutoScaler Resource Specification

The AutoScaler custom resource defines the scaling behavior and configuration:

```yaml
apiVersion: scalers.gurobi.com/v1alpha1
kind: AutoScaler
metadata:
  name: my-autoscaler
  namespace: gurobi-manager-operator-system
spec:
  # Authentication (choose one method)
  accessIdSecretRef:
    name: autoscaler-credentials
    key: accessId
  secretRef:
    name: autoscaler-credentials
    key: secret

  # OR direct values (not recommended for production)
  # accessId: "your-access-id"
  # secret: "your-secret-key"

  # Scaling bounds
  minReplicas: 2
  maxReplicas: 10

  # Target deployment
  targetRef:
    name: gurobi-compute
    namespace: default

  # Manager API endpoint
  manager: "http://gurobi-manager.default.svc.cluster.local:61080"

  # Scaling rules
  scaleUpRules:
    maxQueuedJobs: 5
    maxQueuedWaitingTime: 30

  scaleDownRules:
    minQueuedJobs: 2
    minQueuedWaitingTime: 10
    maxIdleTime: 5

  # Timeouts and intervals
  nodeStopMaxWaitTime: 120
  scalingOperationMaxWaitTime: 0
  requeueInterval: 30
  errorRequeueInterval: 10
```

### Configuration Parameters

#### Gurobi Cluster Manager Connection

| Parameter | Type     | Description                                                                                                      | Required |
|-----------|----------|------------------------------------------------------------------------------------------------------------------|----------|
| `manager` | `string` | URL of the Gurobi Cluster Manager API endpoint for accessing cluster metrics and controlling optimization nodes | Yes      |

#### Gurobi Cluster Manager Authentication

| Parameter           | Type           | Description                                                                    | Required |
|---------------------|----------------|--------------------------------------------------------------------------------|----------|
| `accessIdSecretRef` | `SecretKeyRef` | Reference to secret containing Cluster Manager API access ID                  | Yes*     |
| `secretRef`         | `SecretKeyRef` | Reference to secret containing Cluster Manager API secret key                 | Yes*     |
| `accessId`          | `string`       | Direct Cluster Manager API access ID value                                    | Yes*     |
| `secret`            | `string`       | Direct Cluster Manager API secret value                                       | Yes*     |

*Either secret references OR direct values must be provided

#### Gurobi Compute Node Scaling Bounds

| Parameter     | Type    | Description                                              | Default | Validation     |
|---------------|---------|----------------------------------------------------------|---------|----------------|
| `minReplicas` | `int32` | Minimum number of Gurobi compute nodes (solver engines) | -       | >= 1           |
| `maxReplicas` | `int32` | Maximum number of Gurobi compute nodes (solver engines) | -       | >= minReplicas |

#### Target Reference

| Parameter             | Type     | Description                 | Required |
|-----------------------|----------|-----------------------------|----------|
| `targetRef.name`      | `string` | Target deployment name      | Yes      |
| `targetRef.namespace` | `string` | Target deployment namespace | Yes      |

#### Node Group Management

| Parameter | Type     | Description                                                                                                                                              | Required |
|-----------|----------|----------------------------------------------------------------------------------------------------------------------------------------------------------|----------|
| `group`   | `string` | Gurobi cluster node group identifier. When specified, the autoscaler only monitors and scales nodes belonging to this specific group within the cluster | No       |

#### Optimization Job Scale Up Rules

| Parameter              | Type    | Description                                                                                                          | Default | Validation |
|------------------------|---------|----------------------------------------------------------------------------------------------------------------------|---------|------------|
| `maxQueuedJobs`        | `int32` | Max queued optimization jobs before scale up, 0 = unlimited queue allowed                                           | 0       | >= 0       |
| `maxQueuedWaitingTime` | `int64` | Max wait time (seconds) for optimization jobs before scale up, Set to 0 to disable waiting time-based scaling      | 0       | >= 0       |

#### Optimization Job Scale Down Rules

| Parameter              | Type    | Description                                                                                                                                                                                                        | Default | Validation |
|------------------------|---------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------|------------|
| `minQueuedJobs`        | `int32` | Min queued optimization jobs for scale down, Use 0 or omit this field to disable scaling down based on queued jobs.                                                                                              | 0       | >= 0       |
| `minQueuedWaitingTime` | `int64` | Min wait time (seconds) for optimization jobs for scale down, Use 0 or omit this field to disable scaling down based on queued waiting time.                                                                     | 0       | >= 0       |
| `maxIdleTime`          | `int`   | Max idle time (minutes) before scaling down compute nodes, Use -1 or omit this field to disable scaling down based on idle time. If set to -1, the autoscaler will not scale down idle optimization nodes. | 10      | >= -1      |

#### Timeouts and Intervals

| Parameter                     | Type    | Description                                                                                                                                                                                         | Default | Validation |
|-------------------------------|---------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------|------------|
| `nodeStopMaxWaitTime`         | `int32` | Max wait time (minutes) for node stop, -1 = wait indefinitely, 0 = use default (600 min)                                                                                                            | 600     | >= -1      |
| `scalingOperationMaxWaitTime` | `int32` | Max wait time (minutes) for scaling, If set to 0, NodeStopMaxWaitTime will be used as the default value. If set to -1, the autoscaler will wait indefinitely for the scaling operation to complete. | 0       | >= -1      |
| `requeueInterval`             | `int64` | Normal reconciliation interval (seconds)                                                                                                                                                            | 30      | 1-3600     |
| `errorRequeueInterval`        | `int64` | Error reconciliation interval (seconds)                                                                                                                                                             | 10      | 1-3600     |

## API Reference

### AutoScaler Custom Resource

The AutoScaler resource contains specification fields for configuration and status fields for current state reporting.

#### Spec Fields

The specification includes target reference, replica bounds, authentication credentials, Manager URL, scaling rules, and timeout configurations.

#### Status Fields

The status reports current replicas, desired replicas, scaling state, and scaling start time.

### Validation Rules

The operator enforces the following validation rules:

1. **Replica Bounds**: `minReplicas >= 1` and `maxReplicas >= minReplicas`
2. **Authentication**: Either secret references or direct values must be provided
3. **Scale Up Rules**: At least one scale up rule must be configured
4. **Scale Down Rules**: At least one scale down rule must be configured
5. **Manager URL**: Must be a valid URI format

## Examples

### Basic Configuration

```yaml
apiVersion: scalers.gurobi.com/v1alpha1
kind: AutoScaler
metadata:
  name: basic-autoscaler
  namespace: gurobi-manager-operator-system
spec:
  # Use secrets for credentials
  accessIdSecretRef:
    name: autoscaler-credentials
    key: accessId
  secretRef:
    name: autoscaler-credentials
    key: secret

  # Scale between 1 and 5 nodes
  minReplicas: 1
  maxReplicas: 5

  # Target the gurobi-compute deployment
  targetRef:
    name: gurobi-compute
    namespace: default

  # Manager service URL
  manager: "http://gurobi-manager.default.svc.cluster.local:61080"

  # Simple scaling rules
  scaleUpRules:
    maxQueuedJobs: 3        # Scale up when more than 3 jobs queued

  scaleDownRules:
    minQueuedJobs: 1        # Scale down when less than 1 job queued
    maxIdleTime: 10         # Scale down nodes idle for >10 minutes
```

### Advanced Configuration

```yaml
apiVersion: scalers.gurobi.com/v1alpha1
kind: AutoScaler
metadata:
  name: advanced-autoscaler
  namespace: gurobi-manager-operator-system
spec:
  # Authentication
  accessIdSecretRef:
    name: autoscaler-credentials
    key: accessId
  secretRef:
    name: autoscaler-credentials
    key: secret

  # Large cluster scaling bounds
  minReplicas: 5
  maxReplicas: 50

  # Target deployment
  targetRef:
    name: gurobi-compute
    namespace: production

  # External manager
  manager: "https://gurobi-manager.company.com:61080"

  # Multi-criteria scaling rules
  scaleUpRules:
    maxQueuedJobs: 10           # Scale up when >10 jobs queued
    maxQueuedWaitingTime: 60    # OR when jobs wait >60 seconds

  scaleDownRules:
    minQueuedJobs: 3            # Scale down when <3 jobs queued
    minQueuedWaitingTime: 30    # AND jobs wait <30 seconds
    maxIdleTime: 15             # OR any node idle >15 minutes

  # Extended timeouts for large jobs
  nodeStopMaxWaitTime: 180      # Wait up to 3 hours for jobs to complete
  scalingOperationMaxWaitTime: 200  # Overall scaling timeout

  # Faster monitoring for production
  requeueInterval: 15           # Check every 15 seconds
  errorRequeueInterval: 5       # Retry errors every 5 seconds
```

### High-Throughput Configuration

```yaml
apiVersion: scalers.gurobi.com/v1alpha1
kind: AutoScaler
metadata:
  name: high-throughput-autoscaler
  namespace: gurobi-manager-operator-system
spec:
  accessIdSecretRef:
    name: autoscaler-credentials
    key: accessId
  secretRef:
    name: autoscaler-credentials
    key: secret

  # Aggressive scaling for high throughput
  minReplicas: 10
  maxReplicas: 100

  targetRef:
    name: gurobi-compute
    namespace: default

  manager: "http://gurobi-manager.default.svc.cluster.local:61080"

  # Scale up quickly when queue builds
  scaleUpRules:
    maxQueuedJobs: 2            # Scale up aggressively
    maxQueuedWaitingTime: 10    # Very low wait tolerance

  # Scale down conservatively
  scaleDownRules:
    minQueuedJobs: 0            # Only scale down when no queue
    minQueuedWaitingTime: 0     # And no waiting
    maxIdleTime: 3              # But remove idle nodes quickly

  # Fast monitoring
  requeueInterval: 10
  errorRequeueInterval: 3
```

### Development/Testing Configuration

```yaml
apiVersion: scalers.gurobi.com/v1alpha1
kind: AutoScaler
metadata:
  name: dev-autoscaler
  namespace: gurobi-manager-operator-system
spec:
  # Direct credentials for testing (not recommended for production)
  accessId: "dev-access-id"
  secret: "dev-secret-key"

  # Small scale for development
  minReplicas: 1
  maxReplicas: 3

  targetRef:
    name: gurobi-compute
    namespace: default

  manager: "http://gurobi-manager.default.svc.cluster.local:61080"

  # Relaxed scaling for testing
  scaleUpRules:
    maxQueuedJobs: 1

  scaleDownRules:
    minQueuedJobs: 0
    maxIdleTime: 2              # Quick scale down for testing

  # Short timeouts for development
  nodeStopMaxWaitTime: 10
  requeueInterval: 15
```

### Group-Based Configuration

The group feature allows you to create multiple AutoScaler instances that operate on different node groups within the same Gurobi cluster. This enables fine-grained scaling control for different workload types or priorities.

```yaml
# Production workload autoscaler
apiVersion: scalers.gurobi.com/v1alpha1
kind: AutoScaler
metadata:
  name: production-autoscaler
  namespace: gurobi-manager-operator-system
spec:
  accessIdSecretRef:
    name: autoscaler-credentials
    key: accessId
  secretRef:
    name: autoscaler-credentials
    key: secret

  # Conservative scaling for production workloads
  minReplicas: 2
  maxReplicas: 6

  targetRef:
    name: gurobi-compute
    namespace: default

  manager: "http://gurobi-manager.default.svc.cluster.local:61080"

  # Only operate on nodes in group "grp1"
  group: "grp1"

  scaleUpRules:
    maxQueuedJobs: 3
    maxQueuedWaitingTime: 15

  scaleDownRules:
    minQueuedJobs: 2
    minQueuedWaitingTime: 10
    maxIdleTime: 2

  requeueInterval: 30

---
# Development workload autoscaler
apiVersion: scalers.gurobi.com/v1alpha1
kind: AutoScaler
metadata:
  name: development-autoscaler
  namespace: gurobi-manager-operator-system
spec:
  accessIdSecretRef:
    name: autoscaler-credentials
    key: accessId
  secretRef:
    name: autoscaler-credentials
    key: secret

  # Aggressive scaling for development workloads
  minReplicas: 1
  maxReplicas: 3

  targetRef:
    name: gurobi-compute-grp2
    namespace: default

  manager: "http://gurobi-manager.default.svc.cluster.local:61080"

  # Only operate on nodes in group "grp2"
  group: "grp2"

  scaleUpRules:
    maxQueuedJobs: 2
    maxQueuedWaitingTime: 15

  scaleDownRules:
    minQueuedJobs: 0
    maxIdleTime: 1

  requeueInterval: 30
```

#### Gurobi Cluster Node Group Benefits

- **Node Group Segmentation**: Different scaling policies for distinct node groups within the Gurobi cluster
- **Compute Resource Management**: Dedicated scaling control for different types of optimization workloads
- **Cluster Performance Tuning**: Independent scaling responsiveness for high-priority vs. batch processing node groups
- **Multi-Tenant Cluster Organization**: Separate autoscaling control for different teams or applications using the same cluster

#### Gurobi Cluster Node Group Configuration

1. **Node Group Registration**: Gurobi compute nodes must register with the cluster manager using specific group identifiers
2. **Target Deployment Mapping**: Each node group typically corresponds to a different Kubernetes deployment of compute nodes
3. **Shared Cluster Manager**: All node groups operate under the same Gurobi Cluster Manager instance
4. **Independent Autoscaling**: Each AutoScaler monitors and scales only nodes belonging to its assigned group within the cluster
5. **Job Distribution**: The cluster manager handles job distribution to appropriate node groups based on group configuration

## Scaling Behavior

**Note**: The AutoScaler only scales Gurobi compute nodes. Workers scaling is not supported.

### Scale Up Decision Logic

The AutoScaler scales up when **ALL** of the following conditions are met:

1. **Queue Threshold Exceeded**:
   - `totalQueuedJobs > maxQueuedJobs` (if `maxQueuedJobs > 0`)
   - AND `maxQueuedWaitingTime > threshold` (if `maxQueuedWaitingTime > 0`)

2. **No Idle Nodes Available**: No existing nodes are idle to handle the queue

3. **Replica Limit**: `currentReplicas < maxReplicas`

4. **License Validation**: All nodes have valid licenses

### Scale Down Decision Logic

The AutoScaler scales down when **ANY** of the following conditions are met:

1. **Queue-Based Scale Down**:
   - `totalQueuedJobs < minQueuedJobs`
   - AND `maxQueuedWaitingTime < minQueuedWaitingTime`

2. **Idle-Based Scale Down**:
   - Any node has been idle (0 running jobs) for longer than `maxIdleTime`

3. **Replica Limit**: Always respects `currentReplicas > minReplicas`

### Graceful Scale Down Process

When scaling down, the operator follows this process:

1. **Node Selection**: Identifies the least-used node (lowest job count, longest idle time)
2. **Stop Processing**: Calls Manager API to stop accepting new jobs on the node
3. **Wait for Completion**: Waits for running jobs to complete (up to `nodeStopMaxWaitTime`)
4. **Pod Identification**: Finds the corresponding Kubernetes pod by IP address
5. **Deployment Update**: Reduces the deployment replica count
6. **Pod Deletion**: Gracefully deletes the specific pod
7. **Verification**: Waits for pod termination confirmation

### Timeout Handling

- **Node Stop Timeout**: If a node doesn't stop within `nodeStopMaxWaitTime`, the operation fails
- **Scaling Operation Timeout**: If the entire scaling operation exceeds `scalingOperationMaxWaitTime`, the operation is reset
- **Fallback Mechanism**: If graceful shutdown fails, the operator falls back to direct deployment scaling

### Error Recovery

The operator handles various error conditions:

- **API Authentication Failures**: Retries with `errorRequeueInterval`
- **Network Timeouts**: Exponential backoff retry
- **License Issues**: Prevents scale up, logs errors
- **Pod Deletion Failures**: Falls back to deployment scaling
- **Scaling Timeouts**: Resets operation state and retries

## Troubleshooting

### Common Issues

#### 1. AutoScaler Not Scaling

**Symptoms**: Deployment replica count doesn't change despite queue conditions

**Troubleshooting Steps**:

```bash
# Check AutoScaler status
kubectl get autoscaler -o yaml

# Check controller logs
kubectl logs -n gurobi-manager-operator-system deployment/controller-manager

# Verify Manager API connectivity
kubectl exec -it deployment/gurobi-compute -- curl http://gurobi-manager.default.svc.cluster.local:61080/api/v1/ping
```

**Common Causes**:
- Invalid credentials
- Manager API unreachable
- Scaling conditions not met
- License validation failures

#### 2. Authentication Failures

**Symptoms**: Logs show "authentication failed" or "unauthorized" errors

**Troubleshooting Steps**:

```bash
# Check credentials secret
kubectl get secret autoscaler-credentials -o yaml

# Verify Manager API key
curl -u "accessId:secret" http://manager-url:61080/api/v1/ping

# Check Manager logs
kubectl logs deployment/gurobi-manager
```

**Solutions**:
- Verify API key is valid and not expired
- Ensure secret is in correct namespace
- Check secret key names match AutoScaler spec

#### 3. Nodes Not Stopping During Scale Down

**Symptoms**: Scale down operations timeout, pods remain running

**Troubleshooting Steps**:

```bash
# Check node status via Manager API
curl http://manager-url:61080/api/v1/metrics

# Check for running jobs
kubectl exec -it pod-name -- curl localhost:61000/api/v1/ping

# Review node stop timeout settings
kubectl get autoscaler -o jsonpath='{.spec.nodeStopMaxWaitTime}'
```

**Solutions**:
- Increase `nodeStopMaxWaitTime` for long-running jobs
- Check for stuck jobs in Manager UI
- Verify nodes are properly registered with Manager

#### 4. Pods Not Terminating

**Symptoms**: Pods remain in Terminating state during scale down

**Troubleshooting Steps**:

```bash
# Check pod status and events
kubectl describe pod pod-name

# Check for finalizers
kubectl get pod pod-name -o yaml | grep finalizers

# Force delete if necessary (last resort)
kubectl delete pod pod-name --force --grace-period=0
```

### Debug Commands

#### Check AutoScaler Status

```bash
# Get current status
kubectl get autoscaler example-autoscaler -o yaml

# Watch for changes
kubectl get autoscaler -w

# Check events
kubectl get events --field-selector involvedObject.name=example-autoscaler
```

#### Monitor Controller Logs

```bash
# Follow controller logs
kubectl logs -n gurobi-manager-operator-system deployment/controller-manager -f

# Filter for specific AutoScaler
kubectl logs -n gurobi-manager-operator-system deployment/controller-manager | grep example-autoscaler
```

#### Manager API Debugging

```bash
# Check Manager metrics
curl http://gurobi-manager.default.svc.cluster.local:61080/api/v1/metrics

# Check specific node
curl http://gurobi-manager.default.svc.cluster.local:61080/api/v1/nodes/{nodeId}

# Check job queue
curl http://gurobi-manager.default.svc.cluster.local:61080/api/v1/jobs
```

### Log Analysis

#### Controller Log Patterns

Look for these key log messages:

- `"AutoScaler status check - Scaling deployment"`: Scale up initiated
- `"AutoScaler status check - Scaling down deployment"`: Scale down initiated
- `"Beginning graceful pod shutdown"`: Graceful shutdown started
- `"Pod successfully terminated"`: Graceful shutdown completed
- `"Scaling operation completed successfully"`: Scaling finished
- `"Scaling reset triggered"`: Operation timeout or error recovery

#### Error Messages

Common error patterns:

- `"Failed to fetch metrics"`: Manager API connectivity issues
- `"Authentication failed"`: Credential problems
- `"License state not valid"`: License validation failures
- `"Node did not stop within timeout"`: Graceful shutdown timeouts
- `"Failed to update deployment"`: Kubernetes API issues

## Best Practices

### Production Deployment

1. **Use Secrets for Credentials**:
   ```yaml
   # Recommended
   accessIdSecretRef:
     name: autoscaler-credentials
     key: accessId

   # Not recommended
   accessId: "direct-value"
   ```

2. **Set Appropriate Timeouts**:
   ```yaml
   # For long-running optimization jobs
   nodeStopMaxWaitTime: 240  # 4 hours
   scalingOperationMaxWaitTime: 300  # 5 hours
   ```

3. **Use Health Checks**:
   ```yaml
   # In compute deployment
   livenessProbe:
     httpGet:
       path: /api/v1/ping
       port: 61000
   readinessProbe:
     httpGet:
       path: /api/v1/ping
       port: 61000
   ```

### Scaling Strategy

1. **Conservative Scale Up**: Start with higher thresholds and adjust based on usage
2. **Aggressive Scale Down**: Remove idle resources quickly to save costs
3. **Reasonable Bounds**: Set `minReplicas` to handle baseline load, `maxReplicas` based on resource limits

### Monitoring Setup

1. **Metrics Collection**: Monitor AutoScaler status and scaling events
2. **Alerting**: Set up alerts for scaling failures and license issues
3. **Dashboard**: Create dashboards showing queue metrics and replica counts

### Security

1. **RBAC**: Use least-privilege RBAC for the operator
2. **Network Policies**: Restrict network access between components
3. **Secret Management**: Rotate credentials regularly
4. **Image Security**: Use signed images and vulnerability scanning

## Monitoring

### Key Metrics to Monitor

#### AutoScaler Metrics

- Current replica count vs desired replica count
- Scaling operation duration
- Scaling failure rate
- Time between scaling events

#### Gurobi Cluster Metrics

- Total queued jobs
- Maximum queued waiting time
- Node utilization (idle vs active)
- Job completion rate
- License utilization

#### Kubernetes Metrics

- Pod startup time
- Pod termination time
- Resource utilization (CPU, memory)
- Node availability

### Monitoring Setup

#### Alerting Rules

Set up alerts for:

```yaml
# Example Prometheus alerting rules
groups:
- name: gurobi-autoscaler
  rules:
  - alert: AutoScalerDown
    expr: up{job="autoscaler-operator"} == 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "AutoScaler operator is down"

  - alert: ScalingFailed
    expr: increase(autoscaler_scaling_failures_total[5m]) > 0
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "AutoScaler scaling operation failed"

  - alert: LongQueueWaitTime
    expr: gurobi_max_queued_waiting_time > 300
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "Jobs waiting too long in queue"
```

### Health Checks

#### AutoScaler Health

```bash
# Check if AutoScaler is responding
kubectl get autoscaler example-autoscaler -o jsonpath='{.status.currentReplicas}'

# Check last scaling operation
kubectl get autoscaler example-autoscaler -o jsonpath='{.status.scalingStartTime}'
```

#### Manager API Health

```bash
# Check Manager API availability
curl -f http://gurobi-manager.default.svc.cluster.local:61080/api/v1/ping

# Check metrics endpoint
curl http://gurobi-manager.default.svc.cluster.local:61080/api/v1/metrics
```
