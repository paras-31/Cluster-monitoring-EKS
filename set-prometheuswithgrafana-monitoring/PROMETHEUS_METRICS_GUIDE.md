# Prometheus Metrics Guide - Cluster Monitoring

## Deployment Updates
Your Prometheus configuration now collects comprehensive cluster metrics including:
- **Cluster load and performance**
- **Node CPU & Memory utilization**
- **Pod resource utilization**
- **Pod creation and state on nodes**

---

## Key Metrics to Query

### 1. CLUSTER LOAD & PERFORMANCE

#### API Server Request Rate
```
rate(apiserver_request_total[5m])
```
Monitor how many requests the Kubernetes API server is handling.

#### API Server Latency
```
histogram_quantile(0.99, rate(apiserver_request_duration_seconds_bucket[5m]))
```
99th percentile latency of API server requests.

---

### 2. NODE METRICS (CPU & Memory Utilization)

#### Node CPU Usage Percentage
```
100 * (1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance))
```
CPU utilization per node (excluding idle time).

#### Node Memory Usage Percentage
```
100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))
```
Memory utilization per node.

#### Node Memory Available
```
node_memory_MemAvailable_bytes / 1024 / 1024 / 1024
```
Available memory on each node in GB.

#### Node Disk Usage
```
100 * (node_filesystem_size_bytes - node_filesystem_avail_bytes) / node_filesystem_size_bytes
```
Disk utilization percentage per node.

#### Nodes Load Average
```
node_load1  # 1-minute average
node_load5  # 5-minute average
node_load15 # 15-minute average
```

---

### 3. POD RESOURCE UTILIZATION

#### Pod CPU Usage
```
sum(rate(container_cpu_usage_seconds_total[5m])) by (pod_name, namespace)
```
CPU usage per pod.

#### Pod Memory Usage
```
sum(container_memory_usage_bytes) by (pod_name, namespace)
```
Memory usage per pod in bytes.

#### Pod Memory as Percentage of Request
```
(sum(container_memory_usage_bytes) by (pod_name, namespace)) / (sum(kube_pod_container_resource_requests_memory_bytes) by (pod_name=label_pod, namespace)) * 100
```
How much of requested memory each pod is using.

#### Pod CPU as Percentage of Limit
```
(sum(rate(container_cpu_usage_seconds_total[5m])) by (pod_name)) / (sum(kube_pod_container_resource_limits_cpu_cores) by (pod_name)) * 100
```
CPU usage compared to limits.

---

### 4. POD CREATION & STATE ON NODES

#### Total Pods per Node
```
count(kube_pod_info) by (node)
```
Number of pods running on each node.

#### Pod Creation Rate
```
rate(kube_pod_created[5m])
```
How many pods are being created per second.

#### Pod Status (Running/Pending/Failed)
```
count(kube_pod_info{pod_phase="Running"}) by (namespace)
count(kube_pod_info{pod_phase="Pending"}) by (namespace)
count(kube_pod_info{pod_phase="Failed"}) by (namespace)
```

#### Pods Pending Scheduling
```
count(kube_pod_info{pod_phase="Pending"})
```
Pods waiting to be scheduled on nodes.

#### Container Restarts
```
rate(kube_pod_container_status_restarts_total[5m])
```
How often containers are restarting.

---

### 5. CONTAINER METRICS

#### Container Network In/Out
```
rate(container_network_receive_bytes_total[5m]) # bytes in per second
rate(container_network_transmit_bytes_total[5m]) # bytes out per second
```

#### Container Filesystem Usage
```
container_fs_usage_bytes / container_fs_limit_bytes * 100
```
Filesystem usage percentage.

---

### 6. KUBE-STATE-METRICS (Cluster State)

#### Deployments Replicas Ready
```
kube_deployment_status_replicas_ready / kube_deployment_spec_replicas
```
Percentage of desired replicas that are ready.

#### StatefulSet Ready Replicas
```
kube_statefulset_status_replicas_ready
```

#### Persistent Volume Claims Bound
```
kube_persistentvolumeclaim_status_phase{phase="Bound"}
```

#### Node Status
```
kube_node_status_condition{condition="Ready",status="true"}
kube_node_status_condition{condition="Ready",status="false"}
```

---

## Dashboard Recommendations

### Create these dashboard panels in Grafana:

1. **Cluster Overview**
   - Total nodes
   - Total pods
   - Cluster CPU & Memory utilization

2. **Node Performance**
   - CPU utilization per node
   - Memory utilization per node
   - Disk utilization per node
   - Load average per node

3. **Pod Resources**
   - Top CPU consumers
   - Top memory consumers
   - Pod count per node
   - Pod creation rate

4. **Pod Health**
   - Pods by phase (Running/Pending/Failed)
   - Container restart rate
   - Pod resource requests vs actual usage

---

## Deployment Steps

### 1. Deploy Updated Prometheus
```bash
cd /Users/Paras_Kamboj/eks-work/cluster-monitoring
kubectl apply -f prometheus-deployment.yml
```

### 2. Verify Deployment
```bash
# Check if Prometheus pod is running
kubectl get pods -n monitoring

# Check logs
kubectl logs -n monitoring -l app=prometheus -f

# Port forward to access UI
kubectl port-forward -n monitoring svc/prometheus-service 9090:9090
```

### 3. Access Prometheus UI
- Open browser: `http://localhost:9090`
- Go to **Status** > **Targets** to see all scrape jobs
- Go to **Graph** to test queries

### 4. Important Notes
- **kube-state-metrics**: The `kube-state-metrics` scrape job requires kube-state-metrics to be deployed in your cluster. If not already installed, you can install it with:
  ```bash
  helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
  helm install kube-state-metrics prometheus-community/kube-state-metrics -n monitoring
  ```

- **Cadvisor metrics**: These come from kubelet's `/metrics/cadvisor` endpoint
- **Scrape interval changed**: From 5s to 15s for better performance

---

## Testing Queries

### Quick Test Queries in Prometheus UI:

1. **Check cluster CPU**
   ```
   100 * (1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])))
   ```

2. **Check available memory**
   ```
   sum(node_memory_MemAvailable_bytes) / 1024 / 1024 / 1024
   ```

3. **Check running pods**
   ```
   count(kube_pod_info{pod_phase="Running"})
   ```

4. **Check pod restarts**
   ```
   sum(rate(kube_pod_container_status_restarts_total[5m]))
   ```

---

## Troubleshooting

### If metrics are not appearing:
1. Check Prometheus targets status: `http://localhost:9090/targets`
2. Look for red targets (scrape failures)
3. Check pod logs: `kubectl logs -n monitoring -l app=prometheus`

### If kube-state-metrics targets show red:
```bash
# Install kube-state-metrics
helm install kube-state-metrics prometheus-community/kube-state-metrics -n monitoring --set prometheus.monitor.enabled=true
```

### If cadvisor metrics missing:
- Verify kubelet is accessible on port 10250
- Check RBAC permissions for Prometheus service account

---

## Resource Planning

Current Prometheus resource allocation:
- **Requests**: 500m CPU, 512Mi Memory
- **Limits**: 1000m CPU, 1Gi Memory
- **Retention**: 15 days

Monitor Prometheus pod usage and adjust if needed.
