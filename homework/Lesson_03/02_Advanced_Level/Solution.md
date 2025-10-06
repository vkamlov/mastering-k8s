To register a new node to an existing Kubernetes cluster with only kubeconfig file, we 
need to perform the following steps:
1. Setup the node with required software (kubelet, containerd, runc, CNI plugins, etc.) and setup kernels parameters.
2. Create kubelet configuration file with proper settings.
```
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration

# Disable x509 client cert auth, since we’re not using it
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: false
  x509:
    clientCAFile: ""

authorization:
  mode: AlwaysAllow

clusterDomain: "cluster.local"
clusterDNS:
  - "10.0.0.10"

# Local system config
resolvConf: "/etc/resolv.conf"
runtimeRequestTimeout: "15m"
failSwapOn: false
seccompDefault: true

# Keep using containerd
containerRuntimeEndpoint: "unix:///run/containerd/containerd.sock"

# We don’t want kubelet trying to generate certs since we’re not using a CA
serverTLSBootstrap: false
```
3. Create kubeconfig file for kubelet with proper settings.

```
apiVersion: v1
kind: Config
clusters:
- cluster:
    # Change the server address to point to the control plane
    server: https://10.0.0.70:6443
  name: test-env
contexts:
- context:
    cluster: test-env
    namespace: default
    user: test-user
  name: test-context
current-context: test-context
kind: Config
users:
- name: test-user
  user:
    token: mock-token
```

4. Start kubelet with proper parameters.
```
kubelet   --config=/var/lib/kubelet/config.yaml   --kubeconfig=/var/lib/kubelet/kubeconfig   --hostname-override=$(hostname)   --v=1
```

5. Verify the node is registered in the cluster.

```
Oct 05 18:55:34 mastering-k8s-sandbox-02 kubelet[21576]: I1005 18:55:34.361561   21576 kubelet_node_status.go:78] "Successfully registered node" node="mastering-k8s-sandbox-02"
Oct 05 18:55:35 mastering-k8s-sandbox-02 kubelet[21576]: I1005 18:55:35.153400   21576 apiserver.go:52] "Watching apiserver"
Oct 05 18:55:35 mastering-k8s-sandbox-02 kubelet[21576]: I1005 18:55:35.180850   21576 desired_state_of_world_populator.go:154] "Finished populating initial desired state of world"
```

```
NAME                       STATUS   ROLES           AGE     VERSION
mastering-k8s-sandbox      Ready    control-plane   3d13h   v1.34.1
mastering-k8s-sandbox-02   Ready    <none>          3m47s   v1.34.1
```

6. Verify pods can be scheduled on the new node.
```
kubectl rollout restart deployment nginx-deployment
```

```
NAME                                READY   STATUS    RESTARTS   AGE   IP           NODE                       NOMINATED NODE   READINESS GATES
nginx-deployment-589d974cf6-lx8v4   1/1     Running   0          7s    10.22.0.16   mastering-k8s-sandbox      <none>           <none>
nginx-deployment-589d974cf6-qdtmt   1/1     Running   0          11s   10.22.0.4    mastering-k8s-sandbox-02   <none>           <none>
nginx-deployment-589d974cf6-qx8vh   1/1     Running   0          9s    10.22.0.5    mastering-k8s-sandbox-02   <none>           <none>
```

