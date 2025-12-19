# Grafana Dashboard Setup & Access Guide

## Quick Start - Access Grafana

### Option 1: Port Forward (Recommended for Testing)
```bash
# Forward Grafana port to localhost
kubectl port-forward -n monitoring svc/grafana-service 3000:3000

# Access Grafana
# Open browser: http://localhost:3000
```

**Default Credentials:**
- **Username**: admin
- **Password**: admin123

---

### Option 2: Using NodePort
```bash
# Get your node IP
kubectl get nodes -o wide

# Get the NodePort
kubectl get svc grafana-service -n monitoring

# Access via: http://<NODE_IP>:30300
```

---

### Option 3: Using Port Forward with Specific Node
```bash
# List all nodes
kubectl get nodes

# Port-forward from specific node pod
kubectl port-forward -n monitoring pod/grafana-xxxxx 3000:3000
```

---

## Step-by-Step Setup

### Step 1: Verify Grafana is Running
```bash
# Check if pod is running
kubectl get pods -n monitoring -l app=grafana

# Expected output: grafana-xxxxx pod should be Running

# Check logs if having issues
kubectl logs -n monitoring -l app=grafana -f
```

### Step 2: Start Port Forward
```bash
cd /Users/Paras_Kamboj/eks-work/cluster-monitoring
kubectl port-forward -n monitoring svc/grafana-service 3000:3000
```

**Output should show:**
```
Forwarding from 127.0.0.1:3000 -> 3000
Forwarding from [::1]:3000 -> 3000
Listening on port 3000
```

### Step 3: Open Grafana in Browser
- Open: `http://localhost:3000`
- You should see the Grafana login page

### Step 4: Login
- **Username**: `admin`
- **Password**: `admin123`

### Step 5: Verify Prometheus Data Source
1. Click **Configuration** (gear icon) on the left sidebar
2. Click **Data Sources**
3. You should see **"Prometheus"** listed
4. Click on it to verify connection details:
   - **URL**: `http://prometheus-service:9090`
   - Should show **"Data source is working"**

If it shows an error:
```bash
# Check Prometheus is running
kubectl get pods -n monitoring -l app=prometheus

# Check service connectivity
kubectl exec -n monitoring <grafana-pod> -- curl http://prometheus-service:9090
```

---

## Creating Custom Dashboards

### Method 1: Using Pre-Built Dashboard

1. Click **+** (plus icon) in the left sidebar
2. Select **Import**
3. Enter Dashboard ID from Grafana.com:
   - **3119** - Kubernetes Cluster Monitoring
   - **6417** - Kubernetes Cluster Health
   - **8588** - Kubernetes Deployment Statefulset Daemonset Metrics
4. Select **Prometheus** as data source
5. Click **Import**

### Method 2: Create Dashboard from Scratch

1. Click **+** → **Dashboard**
2. Click **Add new panel**
3. Select **Prometheus** as data source
4. Enter a query (see below)
5. Configure display options
6. Click **Save**

---

## Common Grafana Queries for Your Metrics

### Panel 1: Cluster CPU Usage
```
Query: 100 * (1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])))
Name: Cluster CPU Usage %
Unit: percent (0-100)
```

### Panel 2: Cluster Memory Usage
```
Query: (sum(node_memory_MemTotal_bytes) - sum(node_memory_MemAvailable_bytes)) / sum(node_memory_MemTotal_bytes) * 100
Name: Cluster Memory Usage %
Unit: percent (0-100)
```

### Panel 3: Node CPU per Node
```
Query: 100 * (1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance))
Name: CPU Usage per Node
Legend: {{instance}}
Unit: percent (0-100)
```

### Panel 4: Node Memory per Node
```
Query: 100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) by (instance)
Name: Memory Usage per Node
Legend: {{instance}}
Unit: percent (0-100)
```

### Panel 5: Running Pods Count
```
Query: count(kube_pod_info{pod_phase="Running"})
Name: Running Pods
```

### Panel 6: Total Pods per Node
```
Query: count(kube_pod_info) by (node)
Name: Pods per Node
Legend: {{node}}
```

### Panel 7: Pod Creation Rate
```
Query: rate(kube_pod_created[5m])
Name: Pod Creation Rate (per sec)
Legend: {{namespace}}
```

### Panel 8: Container Restarts
```
Query: sum(rate(kube_pod_container_status_restarts_total[5m])) by (namespace)
Name: Container Restarts per Namespace
Legend: {{namespace}}
```

### Panel 9: Top CPU Consuming Pods
```
Query: topk(10, sum(rate(container_cpu_usage_seconds_total[5m])) by (pod_name, namespace))
Name: Top 10 CPU Consuming Pods
Legend: {{namespace}} / {{pod_name}}
```

### Panel 10: Top Memory Consuming Pods
```
Query: topk(10, sum(container_memory_usage_bytes) by (pod_name, namespace))
Name: Top 10 Memory Consuming Pods
Legend: {{namespace}} / {{pod_name}}
Unit: bytes (short)
```

---

## Dashboard Organization Tips

### Create a Dashboard with Multiple Panels:

1. **Click +** → **Dashboard**
2. Click **Add new panel** for each metric
3. Arrange panels in a grid:
   - Use drag-and-drop to move panels
   - Resize by dragging corners
4. Group related metrics together
5. Save dashboard with a name like "Cluster Metrics"

### Suggested Dashboard Layout:

```
Row 1: Summary Stats
  - Cluster CPU %  |  Cluster Memory %  |  Total Nodes  |  Running Pods

Row 2: Node Metrics
  - CPU per Node (Graph) - full width
  - Memory per Node (Graph) - full width

Row 3: Pod Metrics
  - Top 10 CPU Pods  |  Top 10 Memory Pods
  - Pods per Node (Bar chart) - full width

Row 4: Pod Health
  - Pod Creation Rate  |  Container Restarts  |  Pods by Status (Pie chart)
```

---

## Important Features to Use

### 1. Time Range Selection
- Click **Last 1 hour** (top right) to change time range
- Options: 5m, 15m, 1h, 6h, 24h, 7d, etc.

### 2. Refresh Rate
- Click **Refresh** (top right)
- Set auto-refresh: 5s, 10s, 30s, 1m, etc.

### 3. Alerts (Optional)
- Click on panel title → **Edit**
- Go to **Alert** tab
- Set up notifications when metrics exceed thresholds

### 4. Templating (Advanced)
- Click **Dashboard settings** (gear icon)
- Select **Variables**
- Create variables for dynamic queries
- Example: Filter by namespace dropdown

---

## Troubleshooting

### Issue: Can't access Grafana on localhost:3000
```bash
# Check if port-forward is still running
# It should show "Forwarding from 127.0.0.1:3000 -> 3000"

# If not, restart it
kubectl port-forward -n monitoring svc/grafana-service 3000:3000
```

### Issue: Prometheus data source shows error
```bash
# Check if Prometheus is running
kubectl get pods -n monitoring -l app=prometheus

# Check service DNS resolution
kubectl exec -n monitoring <grafana-pod> -- nslookup prometheus-service

# Check connectivity
kubectl exec -n monitoring <grafana-pod> -- curl -v http://prometheus-service:9090
```

### Issue: No data showing in panels
1. Go to **Configuration** → **Data Sources**
2. Click **Prometheus** → **Test**
3. Should see "Data source is working"
4. If not, check Prometheus pod logs:
   ```bash
   kubectl logs -n monitoring -l app=prometheus
   ```

### Issue: Can't login with admin/admin123
```bash
# The password is set in grafana-deployment.yml
# Current credentials:
# Username: admin
# Password: admin123

# If password is wrong, edit deployment:
kubectl set env deployment/grafana -n monitoring GF_SECURITY_ADMIN_PASSWORD=newpassword
kubectl rollout restart deployment/grafana -n monitoring
```

---

## Advanced: Persistent Storage for Grafana

Your current setup uses emptyDir (temporary storage). For production:

```yaml
# Add to grafana-deployment.yml volumes section
- name: grafana-storage
  persistentVolumeClaim:
    claimName: grafana-pvc
```

Create PVC:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: grafana-pvc
  namespace: monitoring
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

---

## Quick Command Reference

```bash
# Check Grafana status
kubectl get pods -n monitoring -l app=grafana

# View Grafana logs
kubectl logs -n monitoring -l app=grafana -f

# Describe Grafana pod
kubectl describe pod -n monitoring -l app=grafana

# Get Grafana service info
kubectl get svc grafana-service -n monitoring

# Port forward
kubectl port-forward -n monitoring svc/grafana-service 3000:3000

# Access via terminal (if curl/wget available)
curl -u admin:admin123 http://localhost:3000/api/health

# Restart Grafana if issues
kubectl rollout restart deployment/grafana -n monitoring

# Scale Grafana replicas
kubectl scale deployment grafana --replicas=2 -n monitoring
```

---

## Next Steps

1. ✅ Access Grafana on `http://localhost:3000`
2. ✅ Verify Prometheus data source connection
3. ✅ Create a test dashboard with cluster metrics
4. ✅ Import pre-built Kubernetes dashboards
5. ✅ Set up alerts for critical metrics
6. ✅ Share dashboards with your team
