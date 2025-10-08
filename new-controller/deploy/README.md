# NewResource Controller Deployment

This directory contains Kubernetes manifests for deploying the NewResource controller with leader election enabled.

## Prerequisites

- Kubernetes cluster (v1.20+)
- kubectl configured to access the cluster
- Docker or Podman for building the container image

## Files

- `crd.yaml` - CustomResourceDefinition for NewResource
- `rbac.yaml` - ServiceAccount, ClusterRole, and ClusterRoleBinding
- `deployment.yaml` - Deployment manifest with leader election enabled

## Quick Start

### 1. Build the Container Image

```bash
# From the new-controller directory
docker build -t newresource-controller:latest .
```

If you're using a remote registry, tag and push:

```bash
docker tag newresource-controller:latest <your-registry>/newresource-controller:latest
docker push <your-registry>/newresource-controller:latest
```

Then update the image reference in `deployment.yaml`.

### 2. Deploy to Kubernetes

Apply the manifests in order:

```bash
# Install the CRD
kubectl apply -f deploy/crd.yaml

# Create namespace, ServiceAccount, and RBAC
kubectl apply -f deploy/rbac.yaml

# Deploy the controller
kubectl apply -f deploy/deployment.yaml
```

Or apply all at once:

```bash
kubectl apply -f deploy/
```

### 3. Verify Deployment

Check if the controller is running:

```bash
kubectl get pods -n newresource-system
```

Check the logs:

```bash
kubectl logs -n newresource-system -l app=newresource-controller -f
```

You should see leader election messages like:
```
successfully acquired lease newresource-system/newresource-controller
```

### 4. Test with a Sample Resource

Create a test NewResource:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: apps.newresource.com/v1alpha1
kind: NewResource
metadata:
  name: test-resource
  namespace: default
spec:
  foo: "bar"
EOF
```

Check the resource:

```bash
kubectl get newresources
kubectl describe newresource test-resource
```

## Leader Election

The controller is configured with leader election enabled (`--leader-elect=true`). This means:

- Multiple replicas can be run for high availability
- Only one instance will actively reconcile resources at a time
- If the leader fails, another replica will automatically take over
- Leader election uses the coordination.k8s.io/v1 Lease resource

To scale the deployment:

```bash
kubectl scale deployment newresource-controller -n newresource-system --replicas=3
```

## Security Features

The deployment includes several security best practices:

- Runs as non-root user (UID 65532)
- Read-only root filesystem
- No privilege escalation
- All capabilities dropped
- Resource limits configured

## Cleanup

To remove all resources:

```bash
kubectl delete -f deploy/
```

## Troubleshooting

### Controller not starting

Check the logs:
```bash
kubectl logs -n newresource-system -l app=newresource-controller
```

### RBAC errors

Verify the ServiceAccount and RBAC are correctly applied:
```bash
kubectl get sa,clusterrole,clusterrolebinding -n newresource-system
```

### CRD issues

Check if the CRD is installed:
```bash
kubectl get crd newresources.apps.newresource.com
```
