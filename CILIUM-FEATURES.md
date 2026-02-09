# Cilium Advanced Features - Quick Reference

This document provides a quick reference for the advanced Cilium features enabled in this setup.

## Enabled Features

### 🔭 Hubble - Network Observability

- **Hubble Relay**: Provides API access to Hubble data
- **Hubble UI**: Web-based network visualization
- **Metrics**: DNS, drop, TCP, flow, ICMP, HTTP monitoring
- **Access**: http://localhost:8080/hubble or port-forward to :12000

**Quick Commands:**

```bash
# Port-forward Hubble UI
kubectl port-forward -n kube-system svc/hubble-ui 12000:80

# Watch live network flows
cilium hubble observe

# Watch HTTP traffic
cilium hubble observe --protocol http

# Check dropped packets
cilium hubble observe --verdict DROPPED
```

### 🔒 Layer 7 (HTTP/DNS) Visibility

- HTTP request/response monitoring
- DNS query visibility
- Prometheus metrics on port 9964

**Quick Commands:**

```bash
# Watch DNS queries
cilium hubble observe --type trace:to-endpoint --protocol DNS

# View Layer 7 metrics
kubectl exec -n kube-system ds/cilium -- curl localhost:9964/metrics | grep http
```

### 🛡️ Network Policies

- Enforcement mode: `default`
- Supports Layer 3, 4, and 7 policies
- Identity-based security

**Example Policy:**

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-http-only
spec:
  endpointSelector:
    matchLabels:
      app: backend
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "80"
              protocol: TCP
          rules:
            http:
              - method: "GET"
              - method: "POST"
```

### 📊 Bandwidth Management

- BBR congestion control enabled
- Traffic shaping and rate limiting
- QoS support

**Quick Commands:**

```bash
# Check bandwidth manager status
kubectl exec -n kube-system ds/cilium -- cilium status | grep BandwidthManager

# View bandwidth settings
kubectl exec -n kube-system ds/cilium -- cilium bpf bandwidth list
```

### ⚡ eBPF Optimizations

- **kube-proxy replacement**: Full eBPF-based load balancing
- **Host routing**: BPF-based routing (`hostRouting: bpf`)
- **Masquerade**: eBPF-based NAT
- **Performance**: 10x faster than iptables-based solutions

**Quick Commands:**

```bash
# View eBPF program statistics
kubectl exec -n kube-system ds/cilium -- cilium bpf stats

# List service load balancing maps
kubectl exec -n kube-system ds/cilium -- cilium bpf lb list

# View connection tracking
kubectl exec -n kube-system ds/cilium -- cilium bpf ct list global
```

### 🌐 IPv6 Support

- Dual-stack networking enabled
- IPv4 and IPv6 masquerade
- Full feature parity with IPv4

### 📈 Monitoring & Metrics

- **Prometheus integration**: Metrics exposed on port 9962
- **Service monitors**: Ready for Prometheus Operator
- **Grafana dashboards**: Available for import

**Quick Commands:**

```bash
# View all Cilium metrics
kubectl exec -n kube-system ds/cilium -- curl localhost:9962/metrics

# Check Hubble metrics
kubectl exec -n kube-system svc/hubble-relay -- curl localhost:9966/metrics
```

### 🔐 Additional Security Features

- DNS proxy for DNS traffic visibility
- Identity allocation via CRDs
- Health checking on port 9879
- WireGuard encryption support (disabled by default, requires kernel support)

**To enable WireGuard encryption:**

```bash
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --reuse-values \
  --set encryption.enabled=true \
  --set encryption.type=wireguard
```

### ⚙️ Performance Tuning

- **Load balancing algorithm**: Random (can be changed to Maglev)
- **DSR mode**: Direct Server Return for better performance
- **Auto direct routes**: Disabled (enable if all nodes are on same L2 network)
- **Node port**: Enabled

## Resource Allocation

```yaml
Resources per Cilium Agent:
  Limits:
    CPU: 1000m (1 core)
    Memory: 1Gi
  Requests:
    CPU: 100m (0.1 core)
    Memory: 128Mi
```

Adjust these based on your cluster size and workload.

## Advanced Features (Optional)

These features are available but not enabled by default:

### BGP Control Plane

For advanced routing scenarios with physical network integration:

```bash
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --reuse-values \
  --set bgpControlPlane.enabled=true
```

### Cluster Mesh

For multi-cluster connectivity:

```bash
cilium clustermesh enable
cilium clustermesh connect --destination-context k3d-my-lab-2
```

## Health Checks

```bash
# Full Cilium status
cilium status

# Connectivity test
cilium connectivity test

# Check specific features
kubectl exec -n kube-system ds/cilium -- cilium status --verbose
```

## Useful Dashboard Queries

### Hubble UI Filters

- Filter by namespace: Click on namespace name
- Filter by pod: Click on pod name
- Filter by verdict: Use dropdown (forwarded, dropped, error)
- Filter by traffic type: HTTP, DNS, TCP, UDP

### Prometheus Queries

```promql
# Network policy drops
cilium_drop_count_total

# DNS queries per second
rate(cilium_dns_queries_total[5m])

# HTTP requests per second
rate(cilium_http_requests_total[5m])

# Bandwidth usage
rate(cilium_bpf_bandwidth_bytes_total[5m])
```

## Troubleshooting

### Hubble not showing data

```bash
# Verify Hubble is running
kubectl get pods -n kube-system -l k8s-app=hubble-relay

# Check Hubble relay logs
kubectl logs -n kube-system -l k8s-app=hubble-relay

# Restart Hubble relay
kubectl rollout restart deployment/hubble-relay -n kube-system
```

### Network policies not working

```bash
# Verify policy enforcement is enabled
kubectl exec -n kube-system ds/cilium -- cilium status | grep "Policy Enforcement"

# List all network policies
kubectl get cnp,ccnp -A

# Check policy verdict in Hubble
cilium hubble observe --type policy-verdict
```

## References

- [Cilium Documentation](https://docs.cilium.io/)
- [Hubble Documentation](https://docs.cilium.io/en/stable/observability/hubble/)
- [Network Policy Guide](https://docs.cilium.io/en/stable/network/kubernetes/policy/)
- [eBPF Performance](https://docs.cilium.io/en/stable/operations/performance/)
