# Kubernetes RBAC

This directory contains the Kubernetes RBAC configuration used to demonstrate **Role-Based Access Control (RBAC)** with the Online Boutique project.

## Components

* **ServiceAccount:** `paymentservice-sa`
* **Role:** `paymentservice-reader`
* **RoleBinding:** `paymentservice-reader-binding`

The Role grants the `paymentservice` ServiceAccount read-only access to Pods within the `online-boutique` namespace:

```text
get
list
watch
```

It does **not** allow actions such as deleting Pods or accessing Secrets.

## Permission Flow

```text
paymentservice-sa
       │
       ▼
RoleBinding
       │
       ▼
paymentservice-reader
       │
       ▼
get / list / watch Pods
```

## Verification

Permissions were verified using:

```bash
kubectl auth can-i get pods \
  --as=system:serviceaccount:online-boutique:paymentservice-sa \
  -n online-boutique
```

The ServiceAccount was also verified to have no permission to delete Pods, access Secrets, or access Pods in other namespaces.

## Note

The `paymentservice` application does **not actually require Kubernetes API access**.

This RBAC configuration is intentionally included as a practical demonstration of:

* ServiceAccounts
* Roles and RoleBindings
* Namespace-scoped permissions
* Least-privilege access
* RBAC permission testing with `kubectl auth can-i`

Workloads that do not need access to the Kubernetes API should not be granted unnecessary permissions.
