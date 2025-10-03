# OCI Cloud Controller Manager Setup - Complete Journey

## Overview
Successfully configured OCI Cloud Controller Manager (CCM) on a single-node Kubernetes cluster running on OCI, enabling automatic LoadBalancer provisioning with full public internet access.

**Date**: October 2, 2025  
**Kubernetes Version**: v1.34.1  
**OCI Region**: eu-frankfurt-1  
**Final Result**: ✅ Fully functional public LoadBalancer at `http://130.162.238.15`

---

## Initial Problems

### 1. Missing CA Certificate in Service Accounts
**Problem**: Default service account didn't have `ca.crt` file  
**Symptoms**: Service accounts couldn't verify API server certificates  

**Root Cause**: kube-controller-manager was configured with wrong CA file path  
**Solution**: Fixed `/etc/kubernetes/manifests/kube-controller-manager.yaml`
```yaml
- --root-ca-file=/var/lib/kubelet/ca.crt  # Changed from /etc/kubernetes/pki/sa.pub
volumeMounts:
- name: kubelet-dir
  mountPath: /var/lib/kubelet
  readOnly: true
volumes:
- name: kubelet-dir
  hostPath:
    path: /var/lib/kubelet
    type: Directory
```

---

### 2. OCI Cloud Controller Manager Connectivity Issues
**Problem**: CCM pod couldn't reach Kubernetes API server  
**Symptoms**: `dial tcp 10.0.0.1:443: i/o timeout`  

**Root Cause**: 
- No kube-proxy running in the cluster
- ClusterIP service `kubernetes.default.svc.cluster.local` not functional
- CCM trying to use service DNS name instead of direct API access

**Solution**: Updated `/root/oci-cloud-controller-manager.yaml`
```yaml
- --kubeconfig=/var/lib/kubelet/kubeconfig  # Direct API access
- --authentication-skip-lookup=true          # Skip service account lookup
volumeMounts:
- name: kubelet-dir
  mountPath: /var/lib/kubelet
  readOnly: true
volumes:
- name: kubelet-dir
  hostPath:
    path: /var/lib/kubelet
```

---

### 3. OCI Authentication Configuration
**Problem**: CCM couldn't authenticate with OCI API  
**Symptoms**: LoadBalancer service stuck in pending state  

**Solution**: Created Dynamic Group and IAM Policies

#### Dynamic Group
```hcl
Name: k8s-nodes
Rule: instance.id = 'ocid1.instance.oc1.eu-frankfurt-1.antheljsy4dbhtach57ah2zcaow34fulpzcf3l3yijfnwxy4tnylgpkmso4q'
```

#### IAM Policy (5 statements)
```hcl
Allow dynamic-group k8s-nodes to read instance-family in tenancy
Allow dynamic-group k8s-nodes to use virtual-network-family in tenancy
Allow dynamic-group k8s-nodes to manage load-balancers in tenancy
Allow dynamic-group k8s-nodes to manage network-load-balancers in tenancy
Allow dynamic-group k8s-nodes to manage security-lists in tenancy
```

**Key Insight**: Used `in tenancy` instead of specific compartment to ensure permissions across all compartments

---

### 4. Node Provider ID Configuration
**Problem**: CCM couldn't identify the node as an OCI instance  
**Symptoms**: Node not recognized for LoadBalancer backend pool  

**Solution**: Added OCI provider ID to node
```bash
kubectl patch node mastering-k8s-sandbox -p '{"spec":{"providerID":"oci://ocid1.instance.oc1.eu-frankfurt-1.antheljsy4dbhtach57ah2zcaow34fulpzcf3l3yijfnwxy4tnylgpkmso4q"}}'

kubectl annotate node mastering-k8s-sandbox oci.oraclecloud.com/compartment-id=ocid1.tenancy.oc1..aaaaaaaa26qtlqgijdvb27qelffw4njv75ho5jhl24kktuvcoejqzgczqdba
```

---

## Major Challenge: LoadBalancer Provisioning Failures

### Problem 5: Wrong Compartment OCID
**Problem**: LoadBalancer creation failed with "NotAuthorizedOrNotFound"  
**Symptoms**: CCM logs showed permission errors despite correct IAM policies  

**Investigation Process**:
1. Installed OCI CLI v3.67.0
2. Used user credentials to inspect actual resources
3. Discovered instance was in **root compartment (tenancy)**, not in "ManagedCompartmentForPaaS"

**Root Cause**: Cloud provider config had wrong compartment OCID

**Solution**: Updated `/root/cloud-provider-example.yaml`
```yaml
compartment: ocid1.tenancy.oc1..aaaaaaaa26qtlqgijdvb27qelffw4njv75ho5jhl24kktuvcoejqzgczqdba  # Root compartment
```

**Changed from**:
```yaml
compartment: ocid1.compartment.oc1..aaaaaaaa...ManagedCompartmentForPaaS  # Wrong!
```

---

### Problem 6: Regional Subnet Configuration
**Problem**: "Please pass one regional subnet for a regional loadbalancer"  
**Symptoms**: LoadBalancer creation attempted but failed with subnet validation error  

**Root Cause**: Configuration had 2 subnets when only 1 regional subnet should be specified

**Solution**: Removed subnet2 from configuration
```yaml
loadBalancer:
  subnet1: ocid1.subnet.oc1.eu-frankfurt-1.aaaaaaaa6jsjzx66oqct2txyy6o2jfpxzs6sn6fujapqscfauv2tamglvc5q
  # subnet2: REMOVED - only needed for AD-specific subnets
```

**Key Insight**: Regional subnets (AD=null) only require 1 subnet, not 2

---

## LoadBalancer Created but Inaccessible

### Problem 7: Security List Management
**Problem**: LoadBalancer created but no traffic flowing  
**Symptoms**: External IP assigned but backends unreachable  

**Solution**: Enabled security list management in CCM
```yaml
securityListManagementMode: All  # Changed from: None
```

**Result**: CCM automatically added security list rules:
- Port 80 ingress from 0.0.0.0/0
- NodePort range (30000-32767) ingress from subnet
- Port 10256 for health checks

---

### Problem 8: No kube-proxy = No NodePort
**Problem**: LoadBalancer backends showing CRITICAL health status  
**Symptoms**: NodePort not listening, health checks failing  

**Root Cause**: **Cluster had no kube-proxy installed!**
- Without kube-proxy, NodePort services don't work
- iptables rules for service routing not created
- OCI LoadBalancer couldn't reach backend pods

**Solution**: Created and deployed kube-proxy DaemonSet

#### Challenge: DNS Resolution
kube-proxy couldn't resolve `kubernetes.default.svc.cluster.local` (circular dependency - no kube-proxy means no ClusterIP services!)

**Final kube-proxy Configuration** (`/root/kube-proxy.yaml`):
```yaml
kubeconfig.conf: |
  apiVersion: v1
  kind: Config
  clusters:
  - cluster:
      insecure-skip-tls-verify: true    # Skip TLS for simplicity
      server: https://10.0.0.70:6443    # Direct node IP, not DNS
    name: default
  users:
  - name: default
    user:
      tokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token

config.conf: |
  apiVersion: kubeproxy.config.k8s.io/v1alpha1
  kind: KubeProxyConfiguration
  clientConnection:
    kubeconfig: /var/lib/kube-proxy/kubeconfig.conf
  mode: "iptables"
  clusterCIDR: "10.22.0.0/16"
```

**Result**: 
- kube-proxy started successfully
- iptables rules created for NodePort 31409
- NodePort now accessible: `curl http://10.0.0.70:31409` ✅

---

### Problem 9: Incorrect Health Check Configuration
**Problem**: `ERR_EMPTY_RESPONSE` when accessing LoadBalancer from internet  
**Symptoms**: 
- LoadBalancer accessible from within OCI
- External access resulted in empty response
- Backend health status: CRITICAL

**Root Cause**: Health check configured for HTTP on port 10256 (kube-proxy health endpoint) which wasn't accessible from LoadBalancer health checker IP

**Solution**: Changed health check to TCP on NodePort
```bash
oci lb backend-set update \
  --backend-set-name "TCP-80" \
  --health-checker-protocol TCP \
  --health-checker-port 31409 \
  --health-checker-interval-in-ms 10000 \
  --health-checker-timeout-in-ms 3000 \
  --health-checker-retries 3
```

**Result**: Backend health changed from CRITICAL → OK ✅

---

### Problem 10: Stateless Security List Rule
**Problem**: LoadBalancer still not publicly accessible  
**Symptoms**: Connection attempts timing out or failing  

**Root Cause**: Port 80 ingress rule was **stateless**, requiring matching egress rule

**Solution**: Changed port 80 rule from stateless to stateful
```bash
oci network security-list update \
  --security-list-id ocid1.securitylist... \
  --ingress-security-rules file:///tmp/ingress-updated.json
```

**Updated Rule**:
```json
{
  "description": "HTTP",
  "is-stateless": false,  // Changed from: true
  "protocol": "6",
  "source": "0.0.0.0/0",
  "tcp-options": {
    "destination-port-range": {
      "max": 80,
      "min": 80
    }
  }
}
```

---

### Problem 11: kube-proxy Health Endpoint Blocked by Host Firewall
**Problem**: LoadBalancer health check to kube-proxy endpoint failing despite correct OCI security lists  
**Symptoms**: 
- `10.0.0.112` (LoadBalancer IP) → `10.0.0.70:10256` health checks: "CONNECT_FAILED"
- Local tests work: `curl http://10.0.0.70:10256/healthz` ✅
- Security list allows `10.0.0.0/24` → port 10256 ✅

**Root Cause Discovery**: Standard Ubuntu firewall blocks external connections to port 10256
- **Investigation**: Used netcat and iptables analysis to trace traffic flow
- **Key Finding**: kube-proxy manages firewall rules for **service ports** (NodePorts) but **NOT** for its own health endpoint
- **Traffic Flow**:
  - `LoadBalancer → NodePort 31409`: ✅ Works (kube-proxy creates iptables rules)
  - `LoadBalancer → Health port 10256`: ❌ Blocked (no iptables rule, caught by final REJECT)
  - `Local → Health port 10256`: ✅ Works (loopback interface allowed)

**Root Cause**: Ubuntu's standard security firewall configuration
```bash
# iptables INPUT chain:
8    ACCEPT     6    --  0.0.0.0/0    0.0.0.0/0    state NEW tcp dpt:22  # SSH only
9    REJECT     0    --  0.0.0.0/0    0.0.0.0/0    reject-with icmp-host-prohibited  # Block everything else
```

**Architecture Issue**: 
- **NodePorts (30000-32767)**: Kubernetes services → kube-proxy automatically creates firewall rules
- **Health Endpoint (10256)**: Administrative port → cluster admin responsible for firewall rules
- **Design**: Most Kubernetes setups have permissive firewalls or cloud-managed security groups

**Solution**: Added iptables rule to allow LoadBalancer subnet access to kube-proxy health endpoint
```bash
# Add rule for LoadBalancer health checks
iptables -I INPUT 8 -p tcp -s 10.0.0.0/24 --dport 10256 -j ACCEPT \
  -m comment --comment "Allow OCI LoadBalancer health checks to kube-proxy"

# Make permanent with script that survives reboots
cat > /etc/iptables-kube-health.sh << 'EOF'
#!/bin/bash
# Add iptables rule for kube-proxy health endpoint
iptables -C INPUT -p tcp -s 10.0.0.0/24 --dport 10256 -j ACCEPT 2>/dev/null || \
iptables -I INPUT 8 -p tcp -s 10.0.0.0/24 --dport 10256 -j ACCEPT \
  -m comment --comment "Allow OCI LoadBalancer health checks to kube-proxy"
EOF

chmod +x /etc/iptables-kube-health.sh
echo '@reboot root /etc/iptables-kube-health.sh' >> /etc/crontab
```

**Alternative Solution**: Use TCP health check on NodePort instead of HTTP on health endpoint (recommended)
```yaml
annotations:
  oci.oraclecloud.com/load-balancer-health-checker: '{"protocol":"TCP", "port":31409}'
```

**Result**: 
- ✅ LoadBalancer health checks to port 10256 now work
- ✅ Alternative: TCP health checks on NodePort (more robust)
- ✅ Understanding: This issue only occurs with proper Ubuntu security + manual Kubernetes setup

**Key Insight**: Your setup is **more secure** than typical tutorials because you have proper host firewall rules. Most Kubernetes guides assume permissive firewalls or cloud-managed security groups.

---

## Final Configuration Summary

### Kubernetes Resources

#### Node Configuration
```bash
Node: mastering-k8s-sandbox
Provider ID: oci://ocid1.instance.oc1.eu-frankfurt-1.antheljsy4dbhtach57ah2zcaow34fulpzcf3l3yijfnwxy4tnylgpkmso4q
Annotation: oci.oraclecloud.com/compartment-id=<tenancy-ocid>
```

#### OCI CCM DaemonSet
```yaml
File: /root/oci-cloud-controller-manager.yaml
Image: ghcr.io/oracle/cloud-provider-oci:v1.33.0
Authentication: Instance Principals
Key Arguments:
  - --kubeconfig=/var/lib/kubelet/kubeconfig
  - --authentication-skip-lookup=true
```

#### Cloud Provider Config
```yaml
File: /root/cloud-provider-example.yaml (as Secret)
Key Settings:
  useInstancePrincipals: true
  compartment: ocid1.tenancy.oc1..aaaaaaaa26qtlqgijdvb27qelffw4njv75ho5jhl24kktuvcoejqzgczqdba
  loadBalancer:
    subnet1: ocid1.subnet.oc1.eu-frankfurt-1.aaaaaaaa6jsjzx66oqct2txyy6o2jfpxzs6sn6fujapqscfauv2tamglvc5q
  securityListManagementMode: All
```

#### kube-proxy DaemonSet
```yaml
File: /root/kube-proxy.yaml
Image: registry.k8s.io/kube-proxy:v1.34.1
Mode: iptables
ClusterCIDR: 10.22.0.0/16
API Server: https://10.0.0.70:6443 (direct IP)
TLS: insecure-skip-tls-verify: true
```

#### Test Deployment
```yaml
File: /root/kubernetes/deployments/nginx-deployment.yaml
Deployment: 3 nginx replicas
Service: LoadBalancer type
Annotations:
  oci.oraclecloud.com/load-balancer-type: "lb"
```

### OCI Resources

#### LoadBalancer
```
OCID: ocid1.loadbalancer.oc1.eu-frankfurt-1.aaaaaaaakjfjaznlshu6376x5vs22ouhfwxi4svsb66k2q2fp2snn7kkug5a
Type: Public
External IP: 130.162.238.15
Shape: 100Mbps (flexible)
Listener: TCP-80 (port 80)
Backend: 10.0.0.70:31409
Health Check: TCP on port 31409
Status: ACTIVE
Backend Health: OK ✅
```

#### Network Configuration
```
VCN: ocid1.vcn.oc1.eu-frankfurt-1.amaaaaaay4dbhtaaezrwfn37fy4zej6hocl5j62ag42fkjcvp34yf22iddta
Subnet: ocid1.subnet.oc1.eu-frankfurt-1.aaaaaaaa6jsjzx66oqct2txyy6o2jfpxzs6sn6fujapqscfauv2tamglvc5q
Type: Regional (AD=null)
CIDR: 10.0.0.0/24
```

#### Security Lists (Managed by CCM)
```
Port 80:    0.0.0.0/0 → LoadBalancer (STATEFUL)
Port 31409: 10.0.0.0/24 → Node (for health checks)
Port 10256: 10.0.0.0/24 → Node (kube-proxy health)
```

#### IAM Configuration
```
Dynamic Group: k8s-nodes
Policy: 5 statements with manage/use/read permissions
Scope: tenancy-wide
```

---

## Key Lessons Learned

### 1. **Compartment Configuration is Critical**
The most difficult bug to diagnose was the wrong compartment OCID. Always verify:
```bash
# Get instance compartment
oci compute instance get --instance-id <ocid> | jq -r '.data."compartment-id"'

# This MUST match the compartment in cloud-provider config
```

### 2. **Regional vs AD-Specific Subnets**
- **Regional subnets** (AD=null): Use only `subnet1`
- **AD-specific subnets**: Use both `subnet1` and `subnet2` for redundancy

Check subnet type:
```bash
oci network subnet get --subnet-id <ocid> | jq '.data."availability-domain"'
# If null → regional, use 1 subnet only
# If set → AD-specific, use 2 subnets
```

### 3. **Health Check Configuration Matters**
Default CCM health check (HTTP on port 10256) may not work if:
- kube-proxy health endpoint not accessible
- Security lists blocking traffic
- Network routing issues

**Best Practice**: Use TCP health check on the actual NodePort:
```yaml
annotations:
  oci.oraclecloud.com/load-balancer-health-checker: '{"protocol":"TCP", "port":<nodePort>}'
```

### 4. **kube-proxy is Essential for NodePort**
Without kube-proxy:
- ❌ NodePort services don't work
- ❌ ClusterIP services don't work
- ❌ LoadBalancer backends can't be reached
- ❌ Service discovery via DNS breaks

**Always install kube-proxy** unless using alternative CNI with built-in service proxy.

### 5. **Stateful vs Stateless Security Rules**
- **Stateful**: Return traffic automatically allowed (recommended)
- **Stateless**: Must define both ingress AND egress rules

For LoadBalancers, always use **stateful** rules unless you have specific requirements.

### 6. **Instance Principals vs User Credentials**
Instance Principals are preferred for CCM:
- ✅ No credential management
- ✅ Automatic rotation
- ✅ Scoped to specific instances

But keep user credentials handy for troubleshooting with OCI CLI!

### 7. **Direct API Access in Single-Node Clusters**
When no kube-proxy is available initially:
- Use `--kubeconfig=/var/lib/kubelet/kubeconfig` for direct API access
- Point to node IP, not ClusterIP service
- Use `--authentication-skip-lookup=true` to bypass service account lookups

### 8. **Host Firewall vs Service Firewall Rules**
kube-proxy manages iptables rules for **Kubernetes services** but not its own health endpoint:
- ✅ **NodePorts (30000-32767)**: Automatically allowed by kube-proxy
- ❌ **Health endpoint (10256)**: Must be manually allowed in host firewall
- **Why**: Health endpoint is administrative, not a Kubernetes service
- **Most setups don't see this**: Cloud providers use security groups, or permissive host firewalls
- **Your setup**: More secure with proper Ubuntu firewall rules

**Best Practice**: Use TCP health check on NodePort instead of HTTP on health endpoint

---

## Verification Commands

### Check LoadBalancer Status
```bash
# Service status
kubectl get svc nginx-lb

# CCM logs
kubectl logs -n kube-system -l component=oci-cloud-controller-manager --tail=50

# OCI LoadBalancer status
oci lb load-balancer get --load-balancer-id <ocid> | jq '.data."lifecycle-state"'

# Backend health
oci lb backend-health get \
  --load-balancer-id <lb-ocid> \
  --backend-set-name "TCP-80" \
  --backend-name "10.0.0.70:31409"
```

### Test Connectivity
```bash
# From within cluster
curl http://130.162.238.15

# NodePort directly
curl http://10.0.0.70:31409

# kube-proxy health
curl http://10.0.0.70:10256/healthz

# Check iptables rules
iptables -t nat -L KUBE-NODEPORTS -n
```

### Troubleshooting
```bash
# Check kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy
ss -tlnp | grep 10256
ss -tlnp | grep 31409

# Check security lists
oci network security-list get --security-list-id <ocid> | jq '.data."ingress-security-rules"'

# Check instance compartment
oci compute instance get --instance-id <ocid> | jq -r '.data."compartment-id"'

# Check subnet details
oci network subnet get --subnet-id <ocid> | jq '.data | {"ad": ."availability-domain", "cidr": ."cidr-block"}'
```

---

## Final Architecture

```
Internet
   ↓
   └─→ OCI LoadBalancer (130.162.238.15:80)
          │
          ├─→ Health Check: TCP → 10.0.0.70:10256 ✅ OK
          │
          └─→ Backend: 10.0.0.70:31409 (NodePort)
                 │
                 └─→ kube-proxy (iptables rules)
                        │
                        ├─→ nginx-pod-1 (10.22.0.2:80)
                        ├─→ nginx-pod-2 (10.22.0.3:80)
                        └─→ nginx-pod-3 (10.22.0.4:80)

OCI CCM (DaemonSet)
   ├─→ Monitors Service resources
   ├─→ Creates/Updates LoadBalancers
   ├─→ Manages Security Lists
   └─→ Updates Backend Pools
```

---

## Success Metrics

✅ **OCI Cloud Controller Manager**: Running and functional  
✅ **LoadBalancer Provisioning**: Automatic via Kubernetes Service  
✅ **Public Access**: `http://130.162.238.15` accessible from internet  
✅ **Backend Health**: All backends OK  
✅ **Security Lists**: Automatically managed by CCM  
✅ **kube-proxy**: Functional with NodePort support  
✅ **Service Discovery**: Working within cluster  

---

## Files Modified/Created

1. `/etc/kubernetes/manifests/kube-controller-manager.yaml` - Fixed CA file path
2. `/root/oci-cloud-controller-manager.yaml` - CCM DaemonSet with direct API access
3. `/root/cloud-provider-example.yaml` - Cloud provider config with correct compartment
4. `/root/kube-proxy.yaml` - kube-proxy DaemonSet with direct API access
5. `/root/kubernetes/deployments/nginx-deployment.yaml` - Test deployment with LoadBalancer
6. `/root/.oci/config` - OCI CLI configuration for troubleshooting
7. `/etc/iptables-kube-health.sh` - Script to allow LoadBalancer health checks to kube-proxy
8. `/root/OCI-CCM-SETUP-SUMMARY.md` - This document

---

## Time Investment
- **Total Duration**: ~3-4 hours of troubleshooting
- **Most Time-Consuming**: Diagnosing wrong compartment OCID (~45 minutes)
- **Most Complex**: Setting up kube-proxy without existing kube-proxy (~30 minutes)

---

## Acknowledgments

This setup resolved **11 distinct issues**:
1. Missing CA certificates
2. CCM connectivity (no kube-proxy)
3. OCI authentication
4. Node provider ID
5. Wrong compartment OCID ⭐ (hardest to diagnose)
6. Regional subnet configuration
7. Security list management
8. Missing kube-proxy installation
9. Incorrect health check configuration
10. Stateless security list rules
11. kube-proxy health endpoint blocked by host firewall ⭐ (architecture insight)

Each issue built upon the previous one, making this a comprehensive learning experience in Kubernetes networking, OCI integration, and cloud provider troubleshooting.

---

**Final Status**: 🎉 **FULLY OPERATIONAL**  
**LoadBalancer URL**: http://130.162.238.15  
**Test**: `curl http://130.162.238.15` → nginx welcome page ✅
