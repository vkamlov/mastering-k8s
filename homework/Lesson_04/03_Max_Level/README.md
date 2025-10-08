# Run manager in k8s with leader election

1. Modify main.go to enable health checks(optional)
2. Create Dockerfile to build image. Use multi-architecture if needed.
3. Create deploy manifests (crd, rbac, deployment with leader election)
4. Build and push image to registry
5. Apply manifests to deploy to k8s
6. Verify deployment and logs
7. Verify leader election