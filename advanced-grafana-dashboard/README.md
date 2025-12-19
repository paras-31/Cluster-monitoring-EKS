# Advanced Grafana Dashboard Setup for EKS

This folder contains a complete, production-ready Kubernetes monitoring stack with Prometheus, Grafana, and kube-state-metrics.

## What's Included

### 1. **prometheus-deployment.yml**
- Prometheus v3.8.1 configured to scrape:
  - Node Exporter metrics (CPU, memory, disk, network)
  - kube-state-metrics (cluster state, pods, deployments, namespaces)
  - Kubernetes API server metrics
- 15-day data retention
- LoadBalancer service on port 80
- RBAC configured for Kubernetes API access

### 2. **grafana-deployment.yml**
- Grafana v10.2.0 with auto-provisioning
- Connects to Prometheus datasource
- Dashboard auto-loads on startup
- LoadBalancer service on port 80
- Default credentials: admin/admin123
- RBAC configured

### 3. **kube-state-metrics-deployment.yml**
- kube-state-metrics v2.9.2
- Runs as single replica deployment
- Exposes metrics on port 8080
- Collects state for:
  - Nodes, Pods, Namespaces
  - Deployments, StatefulSets, DaemonSets
  - Jobs, CronJobs
  - Services, Endpoints, PVCs
  - Resource quotas, Network policies

### 4. **grafana-dashboard-comprehensive.yml**
- Single comprehensive dashboard with 13 panels
- Covers cluster, namespace, and pod metrics
- Includes:
  - Ready Nodes count
  - Total Pods count
  - Namespaces count
  - Deployments count
  - Node CPU Usage %
  - Node Memory Usage %
  - Pod Count by Node
  - Pod Status Distribution
  - Container Restarts
  - Pods by Namespace
  - CPU Requests vs Limits
  - Memory Requests vs Limits
  - Running vs Desired Replicas

## Quick Start

### Deploy Everything
```bash
cd advanced-grafana-dashboard
kubectl apply -f prometheus-deployment.yml
kubectl apply -f kube-state-metrics-deployment.yml
kubectl apply -f grafana-deployment.yml
kubectl apply -f grafana-dashboard-comprehensive.yml
```

### Access Grafana
```bash
# Get LoadBalancer URL
kubectl get svc -n monitoring grafana-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

Then open in browser: `http://<LoadBalancer-DNS>`
- Username: `admin`
- Password: `admin123`

### Access Prometheus
```bash
# Get LoadBalancer URL
kubectl get svc -n monitoring prometheus-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

Then open: `http://<LoadBalancer-DNS>`

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    EKS Cluster                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │          monitoring namespace                     │  │
│  │                                                  │  │
│  │  ┌─────────────────┐  ┌──────────────────────┐  │  │
│  │  │   Prometheus    │  │  kube-state-metrics  │  │  │
│  │  │   (TSDB)        │  │  (Cluster state)     │  │  │
│  │  │ Port: 9090      │  │  Port: 8080          │  │  │
│  │  └────────┬────────┘  └──────────┬───────────┘  │  │
│  │           │                      │              │  │
│  │           └──────────┬───────────┘              │  │
│  │                      │ Scrapes                  │  │
│  │           ┌──────────▼──────────┐              │  │
│  │           │ Node Exporter (x2)  │              │  │
│  │           │ DaemonSet           │              │  │
│  │           └─────────────────────┘              │  │
│  │                                                  │  │
│  │  ┌──────────────────────────────────────────┐  │  │
│  │  │          Grafana                          │  │  │
│  │  │      (Visualization)                      │  │  │
│  │  │      Port: 3000 (via port 80 ELB)        │  │  │
│  │  └──────────────────────────────────────────┘  │  │
│  └──────────────────────────────────────────────────┘  │
│                                                         │
└─────────────────────────────────────────────────────────┘
         │                                    │
         │ AWS ELB (LoadBalancer)              │
         │                                    │
    ┌────▼────────────────────────────────────▼────┐
    │        External Internet                      │
    │  (Accessible via DNS)                        │
    └───────────────────────────────────────────────┘
```

## Metrics Available

### Node Metrics (from Node Exporter)
- `node_cpu_seconds_total` - CPU usage
- `node_memory_MemTotal_bytes` - Total memory
- `node_memory_MemAvailable_bytes` - Available memory
- `node_load1`, `node_load5`, `node_load15` - Load average
- `node_filesystem_*` - Disk usage

### Cluster Metrics (from kube-state-metrics)
- `kube_node_labels` - Node information
- `kube_pod_info` - Pod information
- `kube_pod_status_phase` - Pod status (Running, Pending, Failed, etc.)
- `kube_deployment_*` - Deployment state
- `kube_namespace_labels` - Namespace info
- `kube_pod_container_resource_requests` - Resource requests
- `kube_pod_container_resource_limits` - Resource limits
- `kube_pod_container_status_restarts_total` - Container restarts

## Troubleshooting

### Dashboard Shows "No data"
1. **Wait 2-3 minutes** - Prometheus needs time to scrape metrics
2. **Refresh browser** - F5 or Ctrl+R
3. **Check Prometheus**:
   ```bash
   kubectl port-forward -n monitoring svc/prometheus-service 9090:80
   # Go to http://localhost:9090
   # Search for: kube_pod_info, node_cpu_seconds_total
   ```

### Datasource Connection Error (504)
1. Check datasource URL is correct:
   ```bash
   kubectl get configmap -n monitoring grafana-datasources -o yaml | grep url
   ```
   Should be: `http://prometheus-service.monitoring.svc.cluster.local:80`

2. Restart Grafana:
   ```bash
   kubectl rollout restart deployment/grafana -n monitoring
   ```

### kube-state-metrics Not Collecting
1. Check pod is running:
   ```bash
   kubectl get pods -n monitoring -l app=kube-state-metrics
   ```

2. Check logs for RBAC errors:
   ```bash
   kubectl logs -n monitoring -l app=kube-state-metrics | grep -i error
   ```

3. Verify it can list resources:
   ```bash
   kubectl exec -it -n monitoring $(kubectl get pod -n monitoring -l app=kube-state-metrics -o jsonpath='{.items[0].metadata.name}') -- wget -q -O- http://localhost:8080/metrics | head -20
   ```

## Performance Tuning

### For Large Clusters (>100 nodes)
In `prometheus-deployment.yml`, adjust:
```yaml
scrape_interval: 30s  # Increase from 15s
evaluation_interval: 30s  # Increase from 15s
```

### Reduce Memory Usage
In `prometheus-deployment.yml`:
```yaml
retention: 7d  # Reduce from 15d
```

In `grafana-deployment.yml`:
```yaml
limits:
  memory: "128Mi"  # Reduce from 256Mi
```

## Security

### Production Hardening

1. **Change Grafana Password**:
   - Log in to Grafana
   - Profile → Change password
   - Use strong password

2. **Enable HTTPS**:
   ```bash
   # Create certificate
   kubectl create secret tls grafana-tls \
     --cert=path/to/cert.crt \
     --key=path/to/key.key \
     -n monitoring
   ```

3. **Restrict Access**:
   - Add AWS Security Group ingress rules
   - Or use Kubernetes NetworkPolicy

4. **Enable Authentication**:
   - In `grafana-deployment.yml`, add OAuth/LDAP configuration

## Cleanup

```bash
# Delete all monitoring resources
kubectl delete namespace monitoring

# Or delete individual components
kubectl delete -f prometheus-deployment.yml
kubectl delete -f grafana-deployment.yml
kubectl delete -f kube-state-metrics-deployment.yml
kubectl delete -f grafana-dashboard-comprehensive.yml
```

## Next Steps

1. **Customize Dashboard** - Edit dashboard in Grafana UI
2. **Add Alerts** - Configure alert rules in Prometheus
3. **Add More Dashboards** - Import from Grafana dashboard library
4. **Set Up Persistence** - Replace emptyDir with PVC for Prometheus/Grafana data
5. **Enable RBAC** - Restrict Grafana user permissions

## Files Reference

| File | Purpose | Key Component |
|------|---------|---|
| `prometheus-deployment.yml` | Metrics collection | Prometheus + Node Exporter |
| `grafana-deployment.yml` | Visualization | Grafana + Datasource config |
| `kube-state-metrics-deployment.yml` | Cluster state | kube-state-metrics |
| `grafana-dashboard-comprehensive.yml` | Pre-built dashboard | 13-panel comprehensive dashboard |

---

**Version**: 1.0  
**Last Updated**: December 19, 2025  
**Status**: Production Ready ✅
