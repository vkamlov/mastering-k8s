I've found four ways to solve the scheduling issue:
1. Untaint the node to allow scheduling:
   ```bash
   kubectl taint node <node-name> node.cloudprovider.kubernetes.io/uninitialized-
   ```

2. Add a toleration to the pod spec:
   ```yaml
    tolerations:
    - key: "node.cloudprovider.kubernetes.io/uninitialized"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
   ```

3. Configure control plane without CCM:

4. Configure CCM. As mentioned, it'll be the future extra homework.