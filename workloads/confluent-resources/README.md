# Confluent Platform Resources

This directory contains the Confluent Platform component deployments managed by the Confluent for Kubernetes (CFK) operator.

## Components Deployed

### KRaft Controller
- **Purpose:** Metadata management for Kafka (replaces ZooKeeper)
- **Replicas:** 1
- **Resources:** 1 CPU, 2GB RAM, 10GB storage

### Kafka Broker
- **Purpose:** Event streaming platform
- **Replicas:** 1 (3 in production)
- **Resources:** 2 CPU, 4GB RAM, 50GB storage
- **Internal endpoint:** `kafka.confluent.svc.cluster.local:9071`

### Schema Registry
- **Purpose:** Schema management for Kafka messages
- **Replicas:** 1
- **Resources:** 0.5 CPU, 1GB RAM
- **Internal endpoint:** `schemaregistry.confluent.svc.cluster.local:8081`

### Kafka Connect
- **Purpose:** Data integration with external systems
- **Replicas:** 1
- **Resources:** 1 CPU, 2GB RAM
- **Internal endpoint:** `connect.confluent.svc.cluster.local:8083`

### ksqlDB
- **Purpose:** Stream processing with SQL
- **Replicas:** 1
- **Resources:** 1 CPU, 2GB RAM
- **Internal endpoint:** `ksqldb.confluent.svc.cluster.local:8088`

### Control Center
- **Purpose:** Web-based UI for managing and monitoring Confluent Platform
- **Replicas:** 1
- **Resources:** 1 CPU, 2GB RAM
- **Internal endpoint:** `controlcenter.confluent.svc.cluster.local:9021`
- **Includes:** Embedded Prometheus and Alertmanager

## External Access

### Control Center Web UI

Control Center is exposed externally via Traefik IngressRoute with TLS support.

**Access URLs by Cluster:**
- **Portcullis:** https://controlcenter.portcullis.osow.ski
- **Artoo:** https://controlcenter.artoo.osow.ski

**Authentication:** None (default configuration)
- ⚠️ **Security Note:** Consider adding authentication for production use

**TLS Certificate:** Self-signed via cert-manager
- Browser will show certificate warning (expected)
- Add certificate to trust store or accept warning

### DNS Configuration

Ensure DNS is configured for external access:

```bash
# Check LoadBalancer IP
kubectl get svc -n ingress traefik

# Add DNS record (or /etc/hosts entry)
# *.portcullis.osow.ski -> <LoadBalancer-IP>
# *.artoo.osow.ski -> <LoadBalancer-IP>
```

### Verify Access

```bash
# Test DNS resolution
nslookup controlcenter.portcullis.osow.ski

# Test HTTPS access (allow self-signed cert)
curl -k https://controlcenter.portcullis.osow.ski
```

## Architecture

```
┌─────────────────┐
│   Traefik       │  ← HTTPS/TLS termination
│   IngressRoute  │     (port 443)
└────────┬────────┘
         │ HTTP
         ▼
┌─────────────────┐
│  Control Center │  ← Web UI on port 9021
│   ClusterIP     │
└────────┬────────┘
         │
    ┌────┴────┬────────┬────────┬─────────┐
    ▼         ▼        ▼        ▼         ▼
  Kafka   Schema   ksqlDB   Connect   Metrics
         Registry
```

## Managing Components

### View Component Status

```bash
# All Confluent Platform components
kubectl get kafka,kraftcontroller,schemaregistry,ksqldb,connect,controlcenter -n confluent

# Control Center specifically
kubectl get controlcenter -n confluent controlcenter -o yaml

# Control Center pods and logs
kubectl get pods -n confluent -l app=controlcenter
kubectl logs -n confluent controlcenter-0
```

### Verify IngressRoute

```bash
# Check IngressRoute exists
kubectl get ingressroute -n confluent controlcenter

# Check TLS certificate
kubectl get certificate -n confluent controlcenter-tls
kubectl describe certificate -n confluent controlcenter-tls
```

### Port Forwarding (Alternative Access)

If IngressRoute is not working, access Control Center directly via port-forward:

```bash
kubectl port-forward -n confluent svc/controlcenter 9021:9021
# Then browse to: http://localhost:9021
```

## Troubleshooting

### Control Center UI Not Loading

1. **Check pod status:**
   ```bash
   kubectl get pods -n confluent -l app=controlcenter
   kubectl logs -n confluent controlcenter-0
   ```

2. **Verify service exists:**
   ```bash
   kubectl get svc -n confluent controlcenter
   kubectl get endpoints -n confluent controlcenter
   ```

3. **Check IngressRoute:**
   ```bash
   kubectl describe ingressroute -n confluent controlcenter
   ```

4. **Verify TLS certificate:**
   ```bash
   kubectl get certificate -n confluent controlcenter-tls
   kubectl describe certificate -n confluent controlcenter-tls
   ```

### Connection Errors Inside Control Center

If Control Center UI loads but shows connection errors to Kafka/Connect/ksqlDB:

1. **Verify all components are running:**
   ```bash
   kubectl get pods -n confluent
   ```

2. **Check Control Center configuration:**
   ```bash
   kubectl get controlcenter -n confluent controlcenter -o yaml
   ```

3. **Test connectivity from Control Center pod:**
   ```bash
   kubectl exec -n confluent controlcenter-0 -- curl http://kafka.confluent.svc.cluster.local:9071
   kubectl exec -n confluent controlcenter-0 -- curl http://schemaregistry.confluent.svc.cluster.local:8081
   ```

### DNS Not Resolving

1. **Verify LoadBalancer has external IP:**
   ```bash
   kubectl get svc -n ingress traefik
   ```

2. **Add to /etc/hosts temporarily:**
   ```bash
   echo "<LoadBalancer-IP>  controlcenter.portcullis.osow.ski" | sudo tee -a /etc/hosts
   ```

3. **Update DNS server with wildcard or specific record**

## Storage

All Confluent Platform components use persistent volumes:

- **Storage Class:** `standard` (configurable per cluster overlay)
- **Total Storage:** ~70GB across all components
- **Backup:** Recommended for production deployments

## Scaling

Adjust replicas in cluster-specific overlays:

**Example:** `workloads/confluent-resources/overlays/portcullis/replica-patch.yaml`

```yaml
apiVersion: platform.confluent.io/v1beta1
kind: Kafka
metadata:
  name: kafka
  namespace: confluent
spec:
  replicas: 3  # Scale to 3 brokers
```

## Related Documentation

- [Confluent Platform Documentation](../docs/confluent-platform.md)
- [Confluent for Kubernetes Documentation](https://docs.confluent.io/operator/current/overview.html)
- [Control Center User Guide](https://docs.confluent.io/control-center/current/overview.html)
- [Adding Applications Guide](../../docs/adding-applications.md)

## Security Considerations

### Current State
- ✅ TLS for external access (via IngressRoute)
- ⚠️ No authentication on Control Center
- ⚠️ No encryption between Confluent components (plaintext)
- ⚠️ No RBAC within Control Center

### Recommended Enhancements
1. **Add authentication to Control Center**
   - LDAP/SAML integration
   - Or use Traefik BasicAuth middleware as quick solution

2. **Enable TLS between components**
   - Configure mutual TLS for Kafka, Schema Registry, etc.
   - See: [CFK Security Documentation](https://docs.confluent.io/operator/current/co-security.html)

3. **Implement Network Policies**
   - Restrict pod-to-pod communication
   - Allow only necessary ingress/egress

4. **Migrate to Let's Encrypt**
   - Replace self-signed certificates with trusted certificates
   - Configure ACME ClusterIssuer in cert-manager

## Monitoring

Control Center includes embedded Prometheus and Alertmanager for monitoring Confluent Platform components.

**Metrics endpoints:**
- Prometheus: `http://controlcenter.confluent.svc.cluster.local:9090`
- Alertmanager: `http://controlcenter.confluent.svc.cluster.local:9093`

To integrate with external Prometheus (kube-prometheus-stack):

```yaml
# Add to Prometheus additionalScrapeConfigs
- job_name: 'confluent-platform'
  kubernetes_sd_configs:
    - role: pod
      namespaces:
        names:
          - confluent
  relabel_configs:
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      action: keep
      regex: true
```

## Future Enhancements

- [ ] External access for Schema Registry (port 8081)
- [ ] External access for ksqlDB (port 8088)
- [ ] External access for Kafka Connect (port 8083)
- [ ] OAuth/SSO authentication for Control Center
- [ ] Let's Encrypt TLS certificates
- [ ] Rate limiting on external endpoints
- [ ] Monitoring dashboards in Grafana
   - [ ] Validate https://github.com/confluentinc/confluent-kubernetes-examples/tree/master/monitoring/grafana-dashboard
