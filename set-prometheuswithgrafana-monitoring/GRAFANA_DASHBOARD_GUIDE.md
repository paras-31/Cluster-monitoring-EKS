# How to Access Your Complete Grafana Dashboard

## Step 1: Connect to Grafana

```bash
# Forward Grafana port
kubectl port-forward -n monitoring svc/grafana-service 3000:3000
```

## Step 2: Open Grafana
- URL: `http://localhost:3000`
- Username: `admin`
- Password: `admin123`

## Step 3: View Your Dashboard

The new dashboard **"Kubernetes Cluster Metrics"** will automatically load with:

### Sections Included:

1. **Node CPU Usage %** (Line chart)
   - Shows CPU utilization per node
   - Color-coded: Green = Good, Red = High

2. **Node Memory Usage %** (Line chart)
   - Shows memory utilization per node
   - Real-time tracking

3. **Node Load Average** (Line chart)
   - 1-minute and 5-minute load averages
   - Helps identify node stress

4. **CPU Cores per Node** (Stat)
   - Quick reference of CPU cores available

5. **Pod Distribution per Node** (Pie chart)
   - Visual breakdown of pods across nodes
   - Shows load balance

6. **Total Running Pods** (Stat)
   - Total count of running pods in cluster

7. **Top 10 Memory Consuming Pods** (Bar chart)
   - Shows which pods use most memory
   - Helps identify resource hogs

8. **Top 10 CPU Consuming Pods** (Bar chart)
   - Shows which pods use most CPU
   - Helps identify performance issues

9. **Pod Count per Node** (Bar chart)
   - Shows how many pods per node
   - Helps identify imbalanced nodes

10. **Node Disk Usage %** (Line chart)
    - Disk utilization per node
    - Yellow warning at 70%, Red at 90%

---

## Dashboard Features

### Refresh Rate
- Auto-refreshes every 30 seconds
- Click **Refresh** to update manually

### Time Range
- Default: Last 1 hour
- Change by clicking time selector (top-right)
- Options: 5m, 15m, 1h, 6h, 24h, 7d, 30d

### Hover for Details
- Hover over any chart to see exact values
- Shows timestamp, value, and all metrics

### Legend
- Click legend items to show/hide data
- Shows Mean and Max values in table format

---

## What Each Panel Tells You

### ✅ Healthy Metrics
- CPU < 60% = Good
- Memory < 70% = Good
- Load < number_of_cores = Good
- Disk < 70% = Good

### ⚠️ Warning Signs
- CPU > 70% = Monitor
- Memory > 80% = Plan scale-up
- Load > 2 × CPU cores = Check pods
- Disk > 80% = Clean up storage

### 🔴 Critical
- CPU > 90% = Performance issues
- Memory > 95% = OOM risk
- Load > 4 × CPU cores = Node struggling
- Disk > 95% = Imminent failure

---

## Common Use Cases

### Find Resource Hogs
1. Look at "Top 10 CPU Consuming Pods" panel
2. See which namespace/pod uses most
3. Click on pod name in legend to see trend
4. Use this to optimize workloads

### Balance Workload Across Nodes
1. Check "Pod Count per Node" panel
2. If imbalanced, adjust pod affinity rules
3. Monitor "Pod Distribution per Node" pie chart
4. Should be roughly equal distribution

### Monitor Node Health
1. Track "Node CPU Usage %" over time
2. Check "Node Memory Usage %" trends
3. Monitor "Node Load Average"
4. Alerts if any node consistently >80%

### Capacity Planning
1. Review weekly trends
2. Look for steady increase in CPU/Memory
3. Watch pod count growth
4. Plan to add nodes before hitting limits

---

## Troubleshooting

### Dashboard Not Loading
1. Refresh page: Ctrl+R (Cmd+R on Mac)
2. Check if Prometheus is running:
   ```bash
   kubectl get pods -n monitoring -l app=prometheus
   ```
3. Verify metrics are available:
   ```bash
   curl http://localhost:9090/api/v1/query?query=node_cpu_seconds_total
   ```

### No Data in Panels
1. Wait 2-3 minutes for data collection
2. Ensure Node Exporter is running:
   ```bash
   kubectl get pods -n monitoring -l app=node-exporter
   ```
3. Check Prometheus targets:
   - Go to http://localhost:9090/targets
   - Verify "node-exporter" job shows green (up)

### Panels Show Different Metrics Than Expected
1. Edit the dashboard: Click dashboard title → **Edit**
2. Click on panel → **Edit query**
3. Adjust the PromQL query if needed
4. Click **Save dashboard**

---

## Customizing the Dashboard

### Add a New Panel
1. Click **Add panel** (top toolbar)
2. Select **Prometheus** data source
3. Enter a query (see PROMETHEUS_METRICS_GUIDE.md)
4. Click **Save**

### Change Refresh Rate
1. Click dashboard settings (gear icon, top-right)
2. Set "Refresh" to desired interval
3. Click **Save dashboard**

### Export Dashboard
1. Click dashboard settings (gear icon)
2. Click **Export**
3. Choose "Save to file" or "Copy JSON"
4. Share with team

---

## Useful Queries for New Panels

If you want to create more panels, here are useful queries:

```
# Container restarts
sum(rate(kube_pod_container_status_restarts_total[5m])) by (namespace)

# Pod creation rate
rate(kube_pod_created[5m])

# Memory available per node
node_memory_MemAvailable_bytes / 1024 / 1024 / 1024

# Network traffic in
rate(node_network_receive_bytes_total[5m])

# Network traffic out
rate(node_network_transmit_bytes_total[5m])
```

---

## Save Your Dashboard

Your dashboard is automatically:
- ✅ Saved in ConfigMap `grafana-dashboard-cluster-metrics`
- ✅ Persisted across pod restarts
- ✅ Ready to share with team

To export:
1. Click dashboard settings (gear icon)
2. Select **Export**
3. Copy JSON and save to file
