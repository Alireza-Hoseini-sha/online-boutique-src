# Horizontal Pod Autoscaling

This directory contains HPA configurations for the Online Boutique services.

## Services

* `frontend`
* `checkoutservice`

## Configuration

Both HPAs:

* Scale between **2 and 5 replicas**
* Scale based on **CPU** and **Memory**
* CPU target: **60%**
* Memory target: **70%**
* Scale up quickly
* Scale down gradually with a **5-minute stabilization window**

## Metrics

```yaml
CPU:
  target: 60%

Memory:
  target: 70%
```

CPU and memory utilization are calculated relative to the resource
requests defined in the corresponding Deployments.

## Apply

```bash
kubectl apply -f frontend-hpa.yaml
kubectl apply -f checkoutservice-hpa.yaml
```

## Verify

```bash
kubectl get hpa -n online-boutique
kubectl describe hpa frontend -n online-boutique
kubectl describe hpa checkoutservice -n online-boutique
```

Metrics are provided through Kubernetes Metrics Server.
