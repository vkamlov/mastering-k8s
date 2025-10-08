# Tests
To run tests for this lesson, we need:

1. Download the kubebuilder assets.

```bash
curl -L https://storage.googleapis.com/kubebuilder-tools/kubebuilder-tools-1.30.0-linux-amd64.tar.gz -o /tmp/kubebuilder-tools.tar.gz
sudo tar -C ./new-controller/bin/k8s --strip-components=1 -zxf /tmp/kubebuilder-tools.tar.gz
```

2. Go to the `new-controller/test/' directory and run the tests.

```bash
cd new-controller/test/
KUBEBUILDER_ASSETS=../bin/k8s/ go test
```
After the tests are done, we can see the results in the terminal.
```
2025-10-08T09:42:19+03:00       DEBUG   controller-runtime.test-env     starting control plane
2025-10-08T09:42:21+03:00       DEBUG   controller-runtime.test-env     installing CRDs
2025-10-08T09:42:21+03:00       DEBUG   controller-runtime.test-env     reading CRDs from path  {"path": "../config/crd/bases"}
2025-10-08T09:42:21+03:00       DEBUG   controller-runtime.test-env     read CRDs from file     {"file": "apps.newresource.com_newresources.yaml"}
2025-10-08T09:42:21+03:00       DEBUG   controller-runtime.test-env     installing CRD  {"crd": "newresources.apps.newresource.com"}
2025-10-08T09:42:21+03:00       DEBUG   controller-runtime.test-env     adding API in waitlist  {"GV": "apps.newresource.com/v1alpha1"}
2025-10-08T09:42:21+03:00       DEBUG   controller-runtime.test-env     installing webhooks
2025-10-08T09:42:21+03:00       INFO    Starting EventSource    {"controller": "newresource", "controllerGroup": "apps.newresource.com", "controllerKind": "NewResource", "source": "kind source: *v1alpha1.NewResource"}
2025-10-08T09:42:21+03:00       INFO    Starting Controller     {"controller": "newresource", "controllerGroup": "apps.newresource.com", "controllerKind": "NewResource"}
2025-10-08T09:42:21+03:00       INFO    Starting workers        {"controller": "newresource", "controllerGroup": "apps.newresource.com", "controllerKind": "NewResource", "worker count": 1}
2025-10-08T09:42:22+03:00       INFO    Stopping and waiting for non leader election runnables
2025-10-08T09:42:22+03:00       INFO    Stopping and waiting for leader election runnables
2025-10-08T09:42:22+03:00       INFO    Stopping and waiting for warmup runnables
2025-10-08T09:42:22+03:00       INFO    Shutdown signal received, waiting for all workers to finish     {"controller": "newresource", "controllerGroup": "apps.newresource.com", "controllerKind": "NewResource"}
2025-10-08T09:42:22+03:00       INFO    All workers finished    {"controller": "newresource", "controllerGroup": "apps.newresource.com", "controllerKind": "NewResource"}
2025-10-08T09:42:22+03:00       INFO    Stopping and waiting for caches
2025-10-08T09:42:22+03:00       INFO    Stopping and waiting for webhooks
2025-10-08T09:42:22+03:00       INFO    Stopping and waiting for HTTP servers
2025-10-08T09:42:22+03:00       INFO    Wait completed, proceeding to shutdown the manager
PASS
ok      github.com/vkamlov/mastering-k8s/new-controller/test    6.104s
```
# Metrics
To expose metrics for this lesson, we need:

1. Working control plane and access to it.

2. Run the controller with the following command:
```bash
./bin/manager --metrics-bind-address :8888 ### (or run without --metrics-bind-address flag to use default :8080)
```
3. Hit the metrics endpoint:
```bash
curl http://localhost:8888/metrics
```
