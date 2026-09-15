# Kubernetes Secrets

This directory contains Kubernetes Secrets used to demonstrate secure configuration management for the Online Boutique project.

## Purpose

Secrets are used to keep sensitive configuration values separate from application manifests.

For demonstration purposes, `paymentservice` uses a fake API key and merchant ID stored in a Kubernetes Secret.

```text
paymentservice-secret
        │
        ▼
    SecretKeyRef
        │
        ▼
 Environment Variables
        │
        ▼
 paymentservice container
```

## Example

The Secret contains:

```yaml
type: Opaque

stringData:
  api-key: "FAKE-PAYMENT-API-KEY-12345"
  merchant-id: "FAKE-MERCHANT-001"
```

The values are injected into the container using `secretKeyRef` instead of hardcoding them directly in the Deployment.

## Security Note

All values in this directory are **fake credentials created for demonstration purposes**.

Real credentials must never be committed to Git.

Kubernetes Secrets provide a mechanism for managing sensitive configuration, but storing a value as a Secret does not automatically make it fully secure. For production environments, additional measures such as **encryption at rest** and external secret-management solutions such as **Vault or External Secrets** should be considered.

## Verification

The Secret can be inspected with:

```bash
kubectl get secret paymentservice-secret -n online-boutique
```

Secret keys can be viewed with:

```bash
kubectl describe secret paymentservice-secret -n online-boutique
```

The actual secret values should not be exposed unnecessarily.
