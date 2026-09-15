# Kubernetes Network Policies

This directory contains the Kubernetes `NetworkPolicy` manifests used to
restrict network communication between workloads in the `online-boutique`
namespace.

The goal is to implement a least-privilege network model where services can
communicate only with the workloads that actually require access.

## Overview

By default, Kubernetes Pods can communicate with each other without network
restrictions.

This project uses NetworkPolicies to control internal service-to-service
Ingress traffic.

```text
                    ┌───────────────┐
                    │ loadgenerator │
                    └───────┬───────┘
                            │
                            ▼
                     ┌────────────┐
                     │  frontend  │
                     └─────┬──────┘
                           │
          ┌────────────────┼──────────────────┐
          │                │                  │
          ▼                ▼                  ▼
     adservice       cartservice       productcatalogservice
                           │                  ▲
                           ▼                  │
                       redis-cart             │
                                              │
                     recommendationservice ───┘

                     checkoutservice
                    ┌──────┼──────┬─────────────┐
                    ▼      ▼      ▼             ▼
                 cart   currency  email      payment
                                              
                    └──────────────┬───────────┘
                                   ▼
                              shippingservice
```

Only the required communication paths are explicitly allowed.

## Default Deny

`default-deny.yaml` applies a default deny rule to all Pods in the
`online-boutique` namespace:

```yaml
spec:
  podSelector: {}

  policyTypes:
    - Ingress
```

An empty `podSelector` selects all Pods in the namespace.

Therefore, once this policy is active, incoming traffic is denied unless
another NetworkPolicy explicitly allows it.

## Ingress vs Egress

### Ingress

Ingress represents traffic entering a Pod.

For example:

```text
frontend ───────► cartservice
                    │
                    │ Ingress
                    ▼
                 cartservice
```

The `cartservice` policy allows traffic from:

* `frontend`
* `checkoutservice`

on TCP port `7070`.

### Egress

Egress represents traffic leaving a Pod.

For example:

```text
cartservice ───────► redis-cart
      │
      │ Egress
      ▼
 redis-cart
```

The current policies intentionally do not restrict egress traffic.

Current model:

```text
Ingress  → Restricted
Egress   → Allowed
```

Egress restrictions can be added later if required. A complete egress
restriction would also require explicitly allowing DNS and all required
internal or external dependencies.

## Policies

| Policy                          | Target Pod            | Allowed Sources                                  |  Port |
| ------------------------------- | --------------------- | ------------------------------------------------ | ----: |
| `frontend-ingress`              | frontend              | loadgenerator                                    |  8080 |
| `adservice-ingress`             | adservice              | frontend                                         |  9555 |
| `cartservice-ingress`           | cartservice            | frontend, checkoutservice                        |  7070 |
| `redis-cart-ingress`            | redis-cart             | cartservice                                      |  6379 |
| `productcatalogservice-ingress` | productcatalogservice  | frontend, checkoutservice, recommendationservice |  3550 |
| `recommendationservice-ingress` | recommendationservice  | frontend                                         |  8080 |
| `currencyservice-ingress`       | currencyservice        | frontend, checkoutservice                        |  7000 |
| `paymentservice-ingress`        | paymentservice         | checkoutservice                                  | 50051 |
| `shippingservice-ingress`       | shippingservice        | frontend, checkoutservice                        | 50051 |
| `emailservice-ingress`          | emailservice           | checkoutservice                                  |  8080 |
| `checkoutservice-ingress`       | checkoutservice        | frontend                                         |  5050 |

## Service Communication Matrix

The explicitly allowed communication paths are:

```text
loadgenerator
    │
    └──► frontend:8080

frontend
    ├──► adservice:9555
    ├──► cartservice:7070
    ├──► checkoutservice:5050
    ├──► currencyservice:7000
    ├──► productcatalogservice:3550
    ├──► recommendationservice:8080
    └──► shippingservice:50051

cartservice
    └──► redis-cart:6379

checkoutservice
    ├──► cartservice:7070
    ├──► currencyservice:7000
    ├──► emailservice:8080
    ├──► paymentservice:50051
    ├──► productcatalogservice:3550
    └──► shippingservice:50051

recommendationservice
    └──► productcatalogservice:3550
```

Any incoming connection not covered by an explicit NetworkPolicy is denied.

## Pod Selectors

The policies identify workloads using Kubernetes Pod labels.

For example:

```yaml
podSelector:
  matchLabels:
    app: cartservice
```

This selects Pods with:

```text
app=cartservice
```

Traffic sources are also selected using Pod labels:

```yaml
from:
  - podSelector:
      matchLabels:
        app: frontend
```

This allows only Pods with:

```text
app=frontend
```

to connect to the selected target.

## Pod Ports vs Service Ports

NetworkPolicies are enforced on Pods rather than Kubernetes Services.

Some Services expose a different Service port and Pod target port.

For example:

```text
frontend

Service port: 80
Target port: 8080
```

Therefore the NetworkPolicy uses:

```yaml
ports:
  - protocol: TCP
    port: 8080
```

Similarly:

```text
emailservice

Service port: 5000
Target port: 8080
```

Therefore the NetworkPolicy for `emailservice` allows TCP port `8080`.

## Validation

Validate the manifests before applying them:

```bash
kubectl apply --dry-run=server -f k8s/network-policies/
```

Apply the policies:

```bash
kubectl apply -f k8s/network-policies/
```

Verify the policies:

```bash
kubectl get networkpolicy -n online-boutique
```

Inspect an individual policy:

```bash
kubectl describe networkpolicy <policy-name> -n online-boutique
```

## NetworkPolicy Testing

NetworkPolicy behavior was tested using a temporary BusyBox Pod.

The test Pod was given:

```text
app=cartservice
```

### Allowed Traffic

The test Pod was able to connect to Redis:

```bash
kubectl exec -n online-boutique np-cart-test -- \
  nc -zvw3 redis-cart 6379
```

Result:

```text
redis-cart (...) 6379 open
```

This confirms:

```text
cartservice ─────► redis-cart:6379
                   ALLOWED
```

### Denied Traffic

The same test Pod attempted to connect to `paymentservice`:

```bash
kubectl exec -n online-boutique np-cart-test -- \
  nc -zvw3 paymentservice 50051
```

Result:

```text
Connection timed out
```

This confirms:

```text
cartservice ─────X────► paymentservice:50051
                       BLOCKED
```

A separate `network-test` Pod without an allowed source label was also unable
to connect to protected services such as `frontend` and
`productcatalogservice`.

## Security Principle

The policies follow the principle of least privilege.

Instead of allowing:

```text
Pod A ─────► Any Pod
```

the configuration defines:

```text
Pod A ─────► Required Pod
```

For example:

```text
cartservice ─────► redis-cart:6379
                   ALLOWED

cartservice ─────X────► paymentservice:50051
                        BLOCKED
```

This reduces the potential impact of a compromised workload.

If an attacker gains control of a service, NetworkPolicy can prevent the
compromised Pod from freely communicating with unrelated services.

## Current Scope

The current implementation focuses on internal service-to-service
Ingress control.

External traffic to `frontend` through an Ingress Controller is not included
yet.

When the Kubernetes Ingress Controller is introduced, the `frontend`
NetworkPolicy should be updated to explicitly allow traffic from the
Ingress Controller Pods.

Conceptually:

```text
Internet
   │
   ▼
Ingress Controller
   │
   ▼
frontend:8080
```

The corresponding NetworkPolicy will allow the Ingress Controller as an
additional source.

## Current Security Status

```text
Default-deny Ingress       ✅
Explicit allow rules       ✅
Pod selectors verified     ✅
Service/Pod ports verified ✅
Calico enforcement         ✅
Allowed traffic tested     ✅
Denied traffic tested      ✅
Egress restrictions        ⏳ Not implemented
External Ingress access    ⏳ Will be added with Ingress Controller
```

## Directory Structure

```text
network-policies/
├── default-deny.yaml
├── frontend.yaml
├── adservice.yaml
├── cartservice.yaml
├── redis.yaml
├── productcatalogservice.yaml
├── recommendationservice.yaml
├── currencyservice.yaml
├── paymentservice.yaml
├── shippingservice.yaml
├── emailservice.yaml
└── checkoutservice.yaml
```

## Future Improvements

* Add Egress NetworkPolicies.
* Explicitly allow DNS traffic when Egress restrictions are enabled.
* Add NetworkPolicies for the Ingress Controller.
* Evaluate namespace-level network isolation.
* Use more specific labels for security-sensitive workloads.
* Add automated NetworkPolicy validation to the CI/CD pipeline.
* Document NetworkPolicy troubleshooting procedures.