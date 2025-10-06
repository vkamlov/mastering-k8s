To register a new node to an existing Kubernetes cluster with CSR authentication, we need to perform the following steps:

1. Setup the node with required software (kubelet, containerd, runc, CNI plugins, etc.) and setup kernels parameters.
2. Modify the api-server to enable CSR authentication. Add the next flags to the api-server manifest file and generate new certificates
to use instead of auth token:
```
--client-ca-file=/etc/kubernetes/pki/ca.crt
--enable-bootstrap-token-auth=true
```
3. Update kubeconfig file for all control plane components to use new certificates.
4. Create a bootstrap token to use it for initial authentication of the new node:
```
head -c 16 /dev/urandom | od -An -t x | tr -d ' '
```

The token file should look like the following example, where the first three values can be anything and the quoted group name should be as depicted:
```
02b50b05283e98dd0fd71db496ef01e8,kubelet-bootstrap,10001,"system:bootstrappers"
```

5. Authorize kubelet to create CSR. Create a ClusterRoleBinding with the following content:
```yaml
# enable bootstrapping nodes to create CSR
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: create-csrs-for-bootstrapping
subjects:
- kind: Group
  name: system:bootstrappers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:node-bootstrapper
  apiGroup: rbac.authorization.k8s.io
```
6. Modify the controller-manager 
In order for the controller-manager to sign certificates, it needs the following:

access to the "Kubernetes CA key and certificate" that we created and distributed enabling CSR signing
the flags:
```
- --cluster-signing-cert-file=/etc/kubernetes/pki/ca-proper.crt
- --cluster-signing-key-file=/etc/kubernetes/pki/ca.key
```
For manual approval of CSRs, add the "-csrapproving" param to this flag to the controller-manager manifest file:
```
- --controllers=*,bootstrapsigner,tokencleaner
```

7. Autoapprove CSRs for nodes(optional)
To autoapprove CSRs for nodes, we need to create a ClusterRoleBinding with the following content:
```yaml
# Approve all CSRs for the group "system:bootstrappers"
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: auto-approve-csrs-for-group
subjects:
- kind: Group
  name: system:bootstrappers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
  apiGroup: rbac.authorization.k8s.io

---

# Approve renewal CSRs for the group "system:nodes"
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: auto-approve-renewals-for-nodes
subjects:
- kind: Group
  name: system:nodes
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
  apiGroup: rbac.authorization.k8s.io
```

8. Collect CA certificate from the control plane and copy it to the new node to the /etc/kubernetes/pki/ca.crt path.

9. Create kubelet configuration file with proper settings.
```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration

authentication:
  anonymous:
    enabled: true
  webhook:
    enabled: true
  x509:
    clientCAFile: "/etc/kubernetes/pki/ca.crt"

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

# Optional — load any static pods here
staticPodPath: "/etc/kubernetes/manifests"
```

10. Create bootstrap configuration file for kubelet with proper settings.
```yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/pki/ca.crt
    server: https://10.0.0.70:6443
  name: bootstrap
contexts:
- context:
    cluster: bootstrap
    user: kubelet-bootstrap
  name: bootstrap
current-context: bootstrap
preferences: {}
users:
- name: kubelet-bootstrap
  user:
    token: mock-token
```

11. Start kubelet with proper parameters.
```shell
kubelet --config=/var/lib/kubelet/config.yaml --bootstrap-kubeconfig=/var/lib/kubelet/bootstrap-kubeconfig --kubeconfig=/var/lib/kubelet/kubeconfig --hostname-override=mastering-k8s-sandbox-02 --v=2
```

12. Check the CSR request and approve it manually if autoapprove is not configured.

```shell
kubectl get csr
NAME                                                   AGE   SIGNERNAME                                    REQUESTOR           REQUESTEDDURATION   CONDITION
node-csr-gvBsfk4mEgsShLyPVVusW_gzVoqQ38EtWgJ7vHZaGHM   13s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   <none>              Pending
```

```shell
kubectl certificate approve node-csr-gvBsfk4mEgsShLyPVVusW_gzVoqQ38EtWgJ7vHZaGHM
```

```shell
kubectl get csr
NAME                                                   AGE     SIGNERNAME                                    REQUESTOR           REQUESTEDDURATION   CONDITION
node-csr-gvBsfk4mEgsShLyPVVusW_gzVoqQ38EtWgJ7vHZaGHM   2m31s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   <none>              Approved,Issued
```

13. Check that the new node is registered in the cluster.
```shell
kubectl get nodes
NAME                       STATUS   ROLES           AGE     VERSION
mastering-k8s-sandbox      Ready    control-plane   3d20h   v1.34.1
mastering-k8s-sandbox-02   Ready    <none>          58s     v1.34.1
```

### Used documentation and references:
https://kubernetes.io/docs/reference/access-authn-authz/kubelet-tls-bootstrapping/
