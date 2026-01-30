# Network Policies Reference

Canonical reference for Kubernetes NetworkPolicy patterns. Referenced by multiple skills.

## Default Deny All

Start with zero trust - deny all traffic by default:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: ${NAMESPACE}
spec:
  podSelector: {}  # Applies to all pods
  policyTypes:
  - Ingress
  - Egress
```

## Allow DNS (Required for most workloads)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: ${NAMESPACE}
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector: {}
      podSelector:
        matchLabels:
          k8s-app: kube-dns
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

## Allow Same Namespace

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: ${NAMESPACE}
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}  # Same namespace only
```

## Allow from Ingress Controller

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-controller
  namespace: ${NAMESPACE}
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
      podSelector:
        matchLabels:
          app.kubernetes.io/name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
```

## Allow from Monitoring

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: ${NAMESPACE}
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: monitoring
      podSelector:
        matchLabels:
          app: prometheus
    ports:
    - protocol: TCP
      port: 9090  # metrics port
```

## Allow Specific Egress (Database)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-database-egress
  namespace: ${NAMESPACE}
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: database
      podSelector:
        matchLabels:
          app: postgresql
    ports:
    - protocol: TCP
      port: 5432
```

## Deny External Egress (Internal Only)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-external-egress
  namespace: ${NAMESPACE}
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  # Allow only RFC1918 private ranges
  - to:
    - ipBlock:
        cidr: 10.0.0.0/8
    - ipBlock:
        cidr: 172.16.0.0/12
    - ipBlock:
        cidr: 192.168.0.0/16
```

## Allow Specific External Egress

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-external-api
  namespace: ${NAMESPACE}
spec:
  podSelector:
    matchLabels:
      app: my-app
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 203.0.113.0/24  # External API range
    ports:
    - protocol: TCP
      port: 443
```

## Common Patterns Summary

| Pattern | Ingress/Egress | Selector |
|---------|----------------|----------|
| Default deny | Both | All pods |
| Allow DNS | Egress | All pods → kube-dns |
| Same namespace | Ingress | All pods ← same namespace |
| From ingress | Ingress | App ← ingress-nginx namespace |
| To database | Egress | App → database namespace |
| Block external | Egress | All pods → RFC1918 only |

## Platform Team Standard Set

For multi-tenant platforms, apply this standard set to each tenant namespace:

1. `default-deny-all` - Zero trust baseline
2. `allow-dns` - DNS resolution
3. `allow-same-namespace` - Intra-namespace communication
4. `allow-ingress-controller` - External traffic via ingress
5. `allow-prometheus-scrape` - Monitoring access

## Troubleshooting

```bash
# List policies in namespace
kubectl get networkpolicies -n ${NAMESPACE}

# Describe policy
kubectl describe networkpolicy ${POLICY} -n ${NAMESPACE}

# Test connectivity
kubectl run test --rm -it --image=nicolaka/netshoot -n ${NAMESPACE} -- \
  curl -m 5 http://${TARGET_SERVICE}

# Check if CNI supports NetworkPolicy
kubectl get pods -n kube-system -l k8s-app=calico-node  # Calico
kubectl get pods -n kube-system -l k8s-app=cilium       # Cilium
```

## Related Skills
- k8s-security-hardening - Full network security implementation
- k8s-platform-tenancy - Tenant isolation policies
- k8s-security-redteam - Testing network policy effectiveness
