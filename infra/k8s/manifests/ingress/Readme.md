# Ingress

This directory contains the Ingress resource used to expose the Online Boutique frontend through the NGINX Ingress Controller.

## Configuration

The Ingress routes HTTP requests for:

```text
boutique.local
```

to the frontend Service:

```text
frontend:80
```

The frontend Service remains a `ClusterIP` because external traffic is handled by the Ingress Controller.

## Traffic Flow

```text
Client
  │
  ▼
NGINX Ingress Controller
  │
  ▼
boutique-ingress
  │
  ▼
frontend Service (ClusterIP)
  │
  ▼
frontend Pod
```

## Apply

```bash
kubectl apply -f boutique-ingress.yaml
```

Verify:

```bash
kubectl get ingress -n online-boutique
kubectl describe ingress boutique-ingress -n online-boutique
```

The Ingress Controller itself is managed separately under:

```text
../addons/ingress-controller.yaml
```
