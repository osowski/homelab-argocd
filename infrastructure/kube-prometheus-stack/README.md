# kube-prometheus-stack

Comprehensive monitoring stack for Kubernetes including Prometheus, Grafana, Alertmanager, and exporters.

## Overview

This deployment installs the [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack) Helm chart, which provides:

- **Prometheus Operator**: Manages Prometheus and Alertmanager instances
- **Prometheus**: Time-series database and monitoring system
- **Alertmanager**: Alert routing and management
- **Grafana**: Visualization and dashboards
- **Node Exporter**: Hardware and OS metrics from cluster nodes
- **Kube State Metrics**: Kubernetes object metrics
- **ServiceMonitors**: Automatic service discovery for scraping
- **PrometheusRules**: Pre-configured alerting rules

## Deployment

### Prerequisites

1. **cert-manager**: Required for TLS certificate generation
2. **Longhorn** (or another storage provider): Required for persistent storage
3. **Traefik** (or another ingress controller): Required for external access

Deploy these components first, or adjust the sync-wave annotation in the Application manifest.

### Automated Deployment

This application is automatically deployed via ArgoCD's App of Apps pattern:

```bash
# Application will be synced automatically within 3 minutes
kubectl get application kube-prometheus-stack -n argocd -w
```

### Manual Sync

If you need to sync immediately:

```bash
argocd app sync kube-prometheus-stack
```

## Access

### Grafana

**URL**: https://grafana.portcullis.osow.ski

**Default Credentials**:
- Username: `admin`
- Password: `portcullis-admin-password` (configured in overlay values)

> **Security Note**: Change the default password immediately after deployment or use Sealed Secrets/External Secrets for production.

### Prometheus

**URL**: https://prometheus.portcullis.osow.ski

No authentication by default. Access is controlled via ingress.

### Alertmanager

**URL**: https://alertmanager.portcullis.osow.ski

No authentication by default. Configure via Alertmanager configuration.

### Port-Forward Access (Alternative)

```bash
# Grafana
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80

# Prometheus
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090

# Alertmanager
kubectl port-forward -n monitoring svc/kube-prometheus-stack-alertmanager 9093:9093
```

## Configuration

### Base Configuration

**File**: `infrastructure/kube-prometheus-stack/base/values.yaml`

Common settings for all clusters:
- Resource limits (conservative for homelab)
- Retention periods (15 days for Prometheus, 5 days for Alertmanager)
- Storage sizes (10Gi Prometheus, 5Gi Grafana, 5Gi Alertmanager)
- Enabled components and ServiceMonitors
- Default alerting rules

### Portcullis Overlay

**File**: `infrastructure/kube-prometheus-stack/overlays/portcullis/values.yaml`

Cluster-specific overrides:
- Ingress configuration with TLS
- Storage class (Longhorn)
- Increased storage and retention for production
- Custom alert rules for homelab
- External URLs for components

## Verification

### Check Deployment Status

```bash
# ArgoCD Application status
kubectl get application kube-prometheus-stack -n argocd

# All pods should be running
kubectl get pods -n monitoring

# Check PVCs are bound
kubectl get pvc -n monitoring

# Verify ingress
kubectl get ingress -n monitoring
```

### Validate Prometheus Targets

```bash
# Access Prometheus UI and check Status > Targets
# All targets should be "UP"
```

### Test Grafana Dashboards

1. Access Grafana at https://grafana.portcullis.osow.ski
2. Navigate to Dashboards
3. Open "Kubernetes / Compute Resources / Cluster"
4. Verify metrics are being displayed

### Check ServiceMonitors

```bash
# List all ServiceMonitors
kubectl get servicemonitor -n monitoring

# Verify they're being discovered by Prometheus
kubectl logs -n monitoring -l app.kubernetes.io/name=prometheus -c prometheus | grep "Loading"
```

## Monitoring

### Pre-configured Dashboards

Grafana includes dozens of pre-configured dashboards:

- **Kubernetes / Compute Resources**: Cluster, namespace, pod, and workload views
- **Kubernetes / Networking**: Network I/O, bandwidth, errors
- **Node Exporter**: Detailed node metrics (CPU, memory, disk, network)
- **Prometheus**: Prometheus server performance
- **Alertmanager**: Alert statistics and status

### Custom Dashboards

Import additional dashboards from [Grafana Dashboard Library](https://grafana.com/grafana/dashboards/):

1. Copy dashboard ID (e.g., 12740 for "Kubernetes Monitoring")
2. In Grafana: Dashboards > Import
3. Enter dashboard ID
4. Select Prometheus datasource
5. Click Import

### Alerting Rules

Default alerting rules are enabled for:
- API server availability and performance
- Node health (CPU, memory, disk)
- Kubernetes resources (pods, deployments, StatefulSets)
- Network issues
- Prometheus and Alertmanager health

**Custom homelab alerts** (defined in portcullis overlay):
- HostHighCpuLoad: CPU usage > 80% for 5 minutes
- HostOutOfMemory: Available memory < 10% for 2 minutes
- HostDiskWillFillIn24Hours: Disk projected to fill in 24 hours
- PodCrashLooping: Pod restarting frequently

### Configuring Alertmanager

Edit the Alertmanager configuration to add notification channels:

```yaml
# In overlay values.yaml
alertmanager:
  config:
    global:
      resolve_timeout: 5m
    route:
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 12h
      receiver: 'default'
    receivers:
      - name: 'default'
        email_configs:
          - to: 'alerts@example.com'
            from: 'prometheus@portcullis.osow.ski'
            smarthost: 'smtp.gmail.com:587'
            auth_username: 'your-email@gmail.com'
            auth_password: 'your-app-password'
```

## Customization

### Adding ServiceMonitors

Create a ServiceMonitor to scrape metrics from your applications:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  namespace: my-namespace
  labels:
    prometheus: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
    - port: metrics
      interval: 30s
      path: /metrics
```

### Adding PrometheusRules

Create custom alerting rules:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-app-alerts
  namespace: my-namespace
  labels:
    prometheus: kube-prometheus-stack
spec:
  groups:
    - name: my-app
      interval: 30s
      rules:
        - alert: MyAppDown
          expr: up{job="my-app"} == 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "My app is down"
            description: "My app has been down for more than 5 minutes"
```

### Increasing Storage

If Prometheus storage fills up:

1. Edit overlay values: increase `storageSpec.volumeClaimTemplate.spec.resources.requests.storage`
2. Commit and push changes
3. Manually resize PVC:
   ```bash
   kubectl edit pvc -n monitoring prometheus-kube-prometheus-stack-prometheus-db-prometheus-kube-prometheus-stack-prometheus-0
   # Change storage size
   ```

## Troubleshooting

### Prometheus Not Scraping Targets

**Check ServiceMonitor labels**:
```bash
kubectl get servicemonitor -n monitoring <name> -o yaml
```

Ensure the ServiceMonitor has appropriate labels and matches the Prometheus selector.

**Check Prometheus logs**:
```bash
kubectl logs -n monitoring prometheus-kube-prometheus-stack-prometheus-0 -c prometheus
```

### Grafana Cannot Connect to Prometheus

**Verify datasource configuration**:
1. Access Grafana
2. Configuration > Data sources
3. Test Prometheus connection
4. Check URL: `http://kube-prometheus-stack-prometheus:9090`

**Check network policies**: Ensure no network policies block traffic between Grafana and Prometheus.

### Alertmanager Not Sending Alerts

**Check Alertmanager configuration**:
```bash
kubectl get secret -n monitoring alertmanager-kube-prometheus-stack-alertmanager -o yaml
```

**Check Alertmanager logs**:
```bash
kubectl logs -n monitoring alertmanager-kube-prometheus-stack-alertmanager-0
```

**Verify notification channels**: Test SMTP, Slack, or other integrations manually.

### PVC Stuck in Pending

**Check storage class**:
```bash
kubectl get storageclass
kubectl describe pvc -n monitoring <pvc-name>
```

Ensure Longhorn (or your storage provider) is installed and healthy.

### High Memory Usage

If Prometheus uses too much memory:

1. Reduce retention: `retention: 7d` instead of `30d`
2. Reduce scrape frequency: `scrapeInterval: 60s` instead of `30s`
3. Increase memory limits in overlay values

## Upgrading

### Check Available Versions

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm search repo prometheus-community/kube-prometheus-stack --versions
```

### Update Chart Version

1. Edit `clusters/portcullis/infrastructure/kube-prometheus-stack.yaml`
2. Change `targetRevision` to new version
3. Commit and push
4. ArgoCD will automatically sync the upgrade

**Important**: Review [release notes](https://github.com/prometheus-community/helm-charts/releases) for breaking changes.

## Related Documentation

- [Adding Helm Workloads Guide](../../docs/adding-helm-workloads.md)
- [Prometheus Operator Documentation](https://prometheus-operator.dev/)
- [kube-prometheus-stack Chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Grafana Documentation](https://grafana.com/docs/)

## Resources

### CPU and Memory

**Default (base values)**:
- Prometheus: 500m CPU, 1Gi memory (requests); 2000m CPU, 2Gi memory (limits)
- Grafana: 100m CPU, 128Mi memory (requests); 500m CPU, 512Mi memory (limits)
- Alertmanager: 100m CPU, 128Mi memory (requests); 200m CPU, 256Mi memory (limits)

**Portcullis (overlay)**:
- Prometheus: 1000m CPU, 2Gi memory (requests); 4000m CPU, 4Gi memory (limits)
- Grafana: 200m CPU, 256Mi memory (requests); 1000m CPU, 1Gi memory (limits)

### Storage

**Base values**:
- Prometheus: 10Gi
- Grafana: 5Gi
- Alertmanager: 5Gi

**Portcullis overlay**:
- Prometheus: 20Gi

## Security Considerations

1. **Change default passwords**: Update Grafana admin password immediately
2. **Use Sealed Secrets**: Encrypt sensitive values for production
3. **Enable authentication**: Configure OAuth or LDAP for Grafana
4. **Restrict ingress**: Use network policies or ingress authentication
5. **RBAC**: Prometheus Operator uses RBAC for cluster access
6. **TLS**: Enabled via cert-manager for all ingress endpoints

## Future Enhancements

- [ ] Configure Alertmanager notification channels (email, Slack, PagerDuty)
- [ ] Add Loki for log aggregation
- [ ] Configure Grafana OAuth authentication
- [ ] Add custom dashboards for homelab services
- [ ] Implement alert silencing and maintenance windows
- [ ] Add Prometheus federation for multi-cluster monitoring
